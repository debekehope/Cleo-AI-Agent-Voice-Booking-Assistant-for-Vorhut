# VORHUT Voice AI Consultation Booking

## Configuration history and documentation scope

This README records the final setup as described during the project conversation, including the Savannah voice, GPT-4.1 model settings captured in the call log, the 5-second Start Speaking Plan wait, the booking API tool, the three n8n workflows, the final prompt and the troubleshooting history. The Stop Speaking Plan and other advanced values were retained as configured; no new values are inferred here.


A voice-based appointment booking assistant for VORHUT, built with
**Vapi**, **n8n**, **Supabase/Postgres**, and **Google Calendar**. The
assistant is named **Cleo**.

This README documents the final workflow design and the configuration
decisions recorded during development. It is intended as a practical
project guide and handover document.

> **Public GitHub safety:** Never commit API keys, Supabase secret keys,
> Authorization headers, private webhook URLs, credential exports,
> customer details, private calendar IDs, Google Sheet IDs, Telegram
> chat IDs, or Slack channel IDs. Use sanitized workflow exports and
> placeholders.

------------------------------------------------------------------------

## Contents

-   [System overview](#system-overview)
-   [Workflow 1: Main booking
    workflow](#workflow-1-main-booking-workflow)
-   [Workflow 2: Post-booking
    notifications](#workflow-2-post-booking-notifications)
-   [Workflow 3: Global workflow error
    handler](#workflow-3-global-workflow-error-handler)
-   [Vapi assistant configuration](#vapi-assistant-configuration)
-   [Booking API tool](#booking-api-tool)
-   [Database and idempotency](#database-and-idempotency)
-   [Duplicate booking and original registration
    timestamp](#duplicate-booking-and-original-registration-timestamp)
-   [Cleo master prompt](#cleo-master-prompt)
-   [Testing checklist](#testing-checklist)
-   [Important limitations and security
    notes](#important-limitations-and-security-notes)

------------------------------------------------------------------------

## System overview

``` text
Caller
  ↓
Vapi voice assistant: Cleo
  ↓ BookVorhutConsultation (API Request tool)
n8n production webhook
  ↓
Main booking workflow
  ├─ Validate request
  ├─ Register / identify booking in Supabase
  ├─ Check Google Calendar availability
  ├─ Create Calendar event
  ├─ Mark booking as confirmed
  ├─ Return booking result to Vapi
  └─ Trigger post-booking notifications asynchronously
```

The main booking workflow returns the booking result to Vapi.
Notifications run separately so an email, spreadsheet, or owner
notification problem does not hold up the caller-facing booking
response.

All appointment times use **Africa/Lagos**.

------------------------------------------------------------------------

## Workflow 1: Main booking workflow

**Workflow name:** `VORHUT Voice AI Consultation Booking`

### Entry points

-   **Webhook:** receives the booking request from Vapi.
-   **Manual test trigger:** used with the `VORHUT Test Booking Data`
    node for controlled testing.

The production webhook is configured to use a response node, and
`Respond to Webhook` returns the final JSON result to Vapi.

### Validation and registration

The main path includes:

1.  `Validate Consultation Details`
2.  `Booking Details Valid?`
3.  `Register Booking Request`
4.  `Check Booking Registration Outcome`

The validation step checks the required booking fields, validates the
email format, checks the appointment date/time and timezone, rejects
past appointments, calculates the end time, and creates an idempotency
key.

The phone number is optional. The consultation duration is 30 minutes
for Cleo's booking flow.

The booking registration uses the Postgres node to call:

``` sql
SELECT *
FROM public.register_vorhut_booking_json($1::jsonb);
```

The database registration outcome determines the next branch.

### New booking path

When the request is newly registered:

1.  Check Google Calendar availability.
2.  If the slot is available, create the Google Calendar event.
3.  Update the booking row to `booked` and store the Calendar event ID.
4.  Build the `booked` response.
5.  Respond to Vapi.
6.  Trigger the post-booking notification workflow without waiting for
    it to finish.

If the slot is unavailable, return `slot_unavailable`. Do not tell the
caller that the appointment was booked.

### Error and status responses

The workflow has response branches for validation errors, database
errors, calendar errors, unavailable slots, and registration states such
as `still_processing` or `previous_failed`.

The caller-facing response must not claim success unless the tool
returns `ok: true` and a successful status.

------------------------------------------------------------------------

## Workflow 2: Post-booking notifications

**Workflow name:**
`VORHUT Post-Booking Notifications (With Failure Alerts)`

This is a separate n8n subworkflow called after a new booking is
confirmed.

### Successful notification branches

The subworkflow starts from `When Called by Main Booking Workflow` and
runs three independent branches:

-   **Gmail -- Customer Confirmation:** sends the booking confirmation
    email to the customer.
-   **Google Sheets -- Log Booking:** appends the booking details to the
    booking log.
-   **Telegram -- Notify Owner:** sends the owner a new-booking
    notification.

Each branch uses its own connection and error output.

### Notification failure alerts

If one of the notification branches fails, its error output routes to
the corresponding Slack alert node:

-   `SLACK ALERT - Gmail failed`
-   `SLACK ALERT - Google Sheets failed`
-   `SLACK ALERT - Telegram booking message failed`

The main booking workflow calls this subworkflow with **Wait for
Sub-workflow disabled**. The caller's booking response therefore does
not wait for notification delivery.

The notification workflow receives booking information such as name,
email, phone, consultation type, clothing needs, appointment start/end,
booking ID, event ID, and status.

------------------------------------------------------------------------

## Workflow 3: Global workflow error handler

**Workflow name:** `VORHUT Error Handler`

This workflow is separate from the post-booking notification
subworkflow.

### Flow

``` text
Error Trigger
  ↓
Slack - Alert on Workflow Failure
```

The Slack error alert is designed to include:

-   Workflow name
-   Failed node
-   Error message
-   Execution ID
-   Execution mode
-   Execution link, when available

This workflow is intended to notify the owner when an n8n workflow
execution fails. It does not replace the branch-specific Slack alerts
inside the post-booking notification workflow.

------------------------------------------------------------------------

## Vapi assistant configuration

**Assistant name:** Cleo\
**Purpose:** VORHUT clothing consultation and appointment booking
assistant

### Greeting

``` text
Hello, thank you for calling VORHUT! I'm Cleo, your clothing consultation assistant. How can I help you today?
```

### Voice configuration — Savannah

- **Voice name:** Savannah
- **Purpose:** Cleo's spoken voice for the VORHUT phone assistant.
- Savannah is the selected voice; the assistant's greeting, confirmations, booking announcements, and closing are spoken in this voice.
- The voice selection is separate from the AI model and the transcriber: the voice produces Cleo's audio, the AI model generates her response, and the transcriber converts the caller's speech into text.

### AI model configuration

The model configuration captured in the call log was:

- **Model:** `gpt-4.1` (the call log also showed the dated model identifier `gpt-4.1-2025-04-14`)
- **Temperature:** `0.5`
- **Maximum tokens:** `250`

**What these settings mean:**
- The model handles the conversation, follows Cleo's prompt, interprets caller requests, and prepares tool calls.
- Temperature `0.5` keeps responses moderately controlled while allowing natural phrasing.
- Maximum tokens `250` limits the length of an individual model response. This helps keep spoken responses reasonably concise.

These values are the model settings captured during the call-log configuration; if the live assistant was subsequently changed, the live Vapi setting takes precedence.

### Transcriber configuration

The transcriber is the speech-recognition component. It converts the caller's audio into text that Cleo can interpret. It is distinct from the GPT model and from Savannah's voice.

The project used a Vapi-configured transcriber, but the exact provider/model name is not preserved in the readable configuration notes available for this README. This README therefore does not invent a provider or model name. The transcriber setting should remain as configured in the final Vapi assistant.

**Troubleshooting note:** An email can be misrecognized during transcription or can be altered later when the model generates tool arguments. To locate the source, compare the caller's spelling, Cleo's readback, the Vapi tool-call arguments, and the n8n Webhook input—in that order.

### Start Speaking Plan and turn-taking

- **Start Speaking Plan wait:** `5 seconds`
- This wait was increased to give Cleo more time before beginning a response after the caller pauses.
- The change was made because Cleo sometimes treated a short pause as the end of the caller's turn and started speaking too early.
- The Start Speaking Plan controls when Cleo begins responding; it does not by itself control speech recognition or guarantee perfect end-of-turn detection.

The Stop Speaking Plan and the other advanced turn-taking settings were left at the configured values during the later troubleshooting; they were not changed as part of the final 5-second Start Speaking Plan adjustment. Keep those settings as configured in the final assistant rather than changing them without a targeted test.

### Other advanced voice settings and call ending

- The advanced voice/turn-taking configuration was retained during the final adjustment; the specific change recorded was the Start Speaking Plan wait to 5 seconds.
- **End Call Phrase:** `Have a lovely day`
- Cleo should finish the full closing sentence and use the phrase to end the call.
- The Hang Up / EndCall tool was not used to force the call to end; the prompt directs Cleo to finish naturally with the configured phrase.

### Knowledge base

Cleo uses the VORHUT FAQ / connected knowledge base for service information. The free consultation is included with every VORHUT service; it is not a separate service.

The FAQ used for the demo was uploaded as a VORHUT clothing/consultation FAQ PDF. Keep the published knowledge base aligned with the current service list and consultation flow.

------------------------------------------------------------------------

## Booking API tool

**Tool name:** `BookVorhutConsultation`\
**Tool type:** Vapi API Request / custom tool\
**Method:** `POST`\
**Destination:** n8n production webhook for the booking workflow

The tool should be called only after Cleo has collected the required
details and the caller has explicitly confirmed the booking summary.

### Request fields

The tool request includes:

  Field                Purpose
  -------------------- --------------------------------------------------------
  `name`               Customer name
  `email`              Customer email address, preserved exactly as confirmed
  `phone`              Optional phone number
  `timeZone`           `Africa/Lagos`
  `clothingNeeds`      Caller's description of what they want help with
  `startDateTime`      ISO 8601 appointment start with timezone offset
  `durationMinutes`    `30` for the consultation
  `consultationType`   VORHUT service selected by the caller

### Tool description

The tool description should clearly state that it:

-   Books only after all required details are collected and the caller
    confirms the date/time.
-   Sends the request to the VORHUT booking workflow.
-   Must not claim success unless the response is successful.
-   Must treat `booked` and `already_booked` as different outcomes.
-   Must not create another booking when the tool returns
    `already_booked`.
-   Must communicate the original registration date/time when the
    duplicate response provides it.

### Tool configuration and response handling

`BookVorhutConsultation` is the Vapi API Request tool that sends the confirmed booking details to the n8n production webhook.

- **Tool name:** `BookVorhutConsultation`
- **Tool type:** API Request / custom tool
- **HTTP method:** `POST`
- **Request destination:** n8n production webhook for `VORHUT Voice AI Consultation Booking`
- **When to call:** only after required booking details are collected and the caller explicitly confirms the appointment date and time.
- **Request data:** caller name, confirmed email, optional phone, consultation type, clothing needs, appointment start, duration, and `Africa/Lagos` timezone.
- **Response handling:** only `ok: true` with `status: booked` or `status: already_booked` is a confirmed outcome. `already_booked` means the appointment already exists and must not be announced as newly created.
- **Duplicate response:** Cleo must use the returned `message` and `originalBookingCreatedAt` to announce the original booking registration date/time, when provided.
- **Other statuses:** `slot_unavailable`, `past_date_time`, `still_processing`, and failure/error responses must not be described as successful bookings.

The tool's description was updated to make the confirmation requirement and duplicate-booking behavior explicit. The email field description was also strengthened to instruct the model to copy the caller-confirmed email character-for-character.

### Email preservation safeguard

Email transcription was occasionally inconsistent: in a few calls Vapi
generated a tool argument with an extra `h`, even though the caller had
spelled and confirmed the intended address. The incorrect email was
visible in the Vapi tool arguments before it reached n8n.

The email field description and prompt should therefore emphasize:

-   Copy the exact caller-confirmed email.
-   Preserve every character in order.
-   Do not reconstruct, autocorrect, duplicate, add, or remove letters.
-   If the confirmed value cannot be preserved, clarify before calling
    the booking tool.

This is a safeguard, not a guarantee that voice transcription or
model-generated arguments can never contain an error. Inspect the tool
arguments in Vapi call logs when investigating an email mismatch.

------------------------------------------------------------------------

## Database and idempotency

The booking workflow uses Supabase/Postgres, including the table:

``` text
public.vorhut_bookings
```

The registration process calls the database function:

``` text
public.register_vorhut_booking_json(p_booking JSONB)
```

### Idempotency key

The workflow uses an idempotency key based on the normalized email and
appointment start time:

``` text
<normalized-email>_<start-time-in-ISO-UTC>
```

This helps identify a repeated request for the same email and
appointment start time and prevents creating a duplicate appointment.

**Important:** If the email is misspelled or changed, the generated
idempotency key changes too. The duplicate lookup may then fail to match
the original record. This is why preserving the confirmed email is
important for both booking identity and duplicate detection.

------------------------------------------------------------------------

## Duplicate booking and original registration timestamp

The duplicate branch is:

``` text
Is Booking Already Confirmed?
  ↓ already_booked
Get Original Booking Date (HTTP Request)
  ↓
Return Existing Booking (Code)
  ↓
Respond to Webhook
  ↓
Vapi / Cleo
```

### HTTP Request configuration

The HTTP Request performs a `GET` request to the Supabase REST endpoint
for `vorhut_bookings`.

It queries:

-   `select=created_at`
-   `idempotency_key=eq.<the validated idempotency key>`
-   `limit=1`

The request returns the original booking row's `created_at` value.

### Return Existing Booking

The Code node:

-   Reads the existing booking details from
    `Is Booking Already Confirmed?`
-   Reads `created_at` from the HTTP Request output
-   Converts the timestamp to `Africa/Lagos`
-   Returns `status: already_booked`
-   Includes `originalBookingCreatedAt`
-   Includes a message with the original registration date/time and
    explains that no duplicate appointment was created

The webhook response must return this JSON to Vapi.

### What the timestamp means

`created_at` is the time the booking record was created in the database.
It is **not necessarily**:

-   The exact time the caller first made the phone call
-   The exact time the confirmation email was sent

Cleo should say that the booking was **registered** at that date/time
and suggest checking the inbox around that time. Do not claim that the
email was sent at that exact time unless a separate email-sent timestamp
is stored and returned.

### Required duplicate response behavior

When `status: already_booked` and the timestamp is available, Cleo must
say both:

1.  The scheduled appointment date/time
2.  The original booking registration date/time

Do not replace the original registration date/time with vague wording
such as "it was booked earlier." The registration timestamp is essential
so the caller can search their inbox for the original confirmation.

------------------------------------------------------------------------

## Cleo master prompt

The master prompt captured from the Vapi call log on **November 24,
2026** is included below as the baseline prompt. Later changes discussed
for the duplicate timestamp and email tool argument safeguards are
documented immediately after it.

### Latest prompt amendments to keep with the master prompt

#### Mandatory duplicate registration date/time

When the booking tool returns `status: already_booked` and
`originalBookingCreatedAt` is available:

-   Cleo must explicitly tell the caller the original booking
    registration date and time.
-   Convert the timestamp to `Africa/Lagos`.
-   Do not omit it or replace it with "booked earlier."
-   Distinguish the appointment date/time from the original registration
    date/time.
-   Tell the caller to check the inbox around the registration
    date/time.
-   Do not claim that the email was sent at that exact time.

Suggested response structure:

> Your consultation is already booked for \[appointment date and time\].
> The original booking was registered on \[original registration date
> and time\] Lagos time. Please check your inbox around that date and
> time for the original confirmation email. No duplicate appointment was
> created.

#### Email tool-argument safeguard

The email field description in the `BookVorhutConsultation` schema
should instruct Cleo to use the exact email address the caller
explicitly spelled and confirmed, character-for-character. Do not
change, duplicate, autocorrect, or reconstruct characters when preparing
the tool arguments.

**Documentation status note:** The master prompt below is captured from
the call log. Confirm that the latest amendments above are present in
the currently published Vapi prompt/tool before describing the live
configuration as fully synchronized.

### Captured master prompt

``` text
# CLEO — VORHUT VOICE ASSISTANT
## Complete Master System Prompt

### 1. IDENTITY AND PURPOSE

You are Cleo, the friendly, professional, natural-sounding voice assistant for VORHUT.

Your role is to:
- Welcome callers.
- Answer questions using the VORHUT FAQ and connected knowledge base.
- Help callers choose the VORHUT service that best fits their needs.
- Explain that every VORHUT service includes a free consultation.
- Collect the required booking details.
- Book consultations using the BookVorhutConsultation tool.
- Confirm bookings only when the booking tool returns a successful result.
- End calls naturally and politely using Vapi's configured End Call Phrase.

Speak warmly and conversationally. Do not sound robotic, overly formal, or scripted.

Ask one question at a time.

---

### 2. OPENING GREETING

Begin each call with:

“Hello, thank you for calling VORHUT! I'm Cleo, your clothing consultation assistant. How can I help you today?”

Do not repeat the opening greeting later in the call.

---

### 3. NATURAL CONVERSATION

Respond naturally to what the caller says.

Do not:
- Thank the caller after every piece of information.
- Repeat the caller's name unnecessarily.
- Use the same acknowledgement repeatedly.
- Sound as though you are reading a checklist.
- Interrupt the caller while they are speaking.
- Repeat a question when the caller has already answered it clearly.
- Ask for information that the caller has already provided.

Short acknowledgements such as “Got it,” “Of course,” or “That works” may be used occasionally, but they are not required after every answer.

If the caller clearly says they did not understand a question, explain it in simpler words and ask again.

Do not assume the caller is confused merely because they pause, hesitate, or take time to respond.

If the caller begins speaking, listen to what they say instead of repeatedly saying “Whenever you're ready” or “Take your time.”

Allow the caller time to think. Do not fill every silence with repeated prompts.

When the caller gives a short or incomplete response, do not immediately repeat the same question or say “Could you please clarify” without context.

Use the conversation to understand what they are trying to communicate. If their answer is genuinely incomplete, ask one specific, simple follow-up.

---

### 4. VORHUT SERVICES AND FREE CONSULTATIONS

Every VORHUT service includes a free consultation.

The free consultation is not a separate service that the caller must choose.

The caller selects the VORHUT service they are interested in, and that selected service determines the consultation type.

Use the VORHUT FAQ and connected knowledge base to identify and describe the actual services offered.

Never invent service names or claim that a service is unavailable if it is listed in the FAQ.

If the caller already knows which service they want, continue with that service.

If the caller is unsure which service to choose:
1. Briefly explain that every VORHUT service includes a free consultation.
2. Help them understand the available services using the FAQ or knowledge base.
3. Offer the relevant service options in a natural, concise way.
4. Help them choose based on what they want.
5. Do not make the caller know the exact service name before you can help.

Once the caller selects a service, use that service as the consultationType when booking.

---

### 5. BOOKING DETAILS

To book a consultation, collect:

- Customer's name
- Email address
- Phone number, if the caller has one
- Selected VORHUT service
- What the caller wants help with during the consultation
- Preferred appointment date
- Preferred appointment time

The consultation duration is always 30 minutes.

The consultation timezone is Africa/Lagos.

Do not require a phone number. If the caller does not have one or prefers not to provide it, continue without it.

Do not confuse the selected service with the caller's clothingNeeds.

consultationType means the VORHUT service selected by the caller.

clothingNeeds means the caller's description of what they want help with during the consultation.

For example, if the caller selects personal styling and says they want to discuss a birthday outfit in general:
- consultationType: personal styling
- clothingNeeds: discuss their birthday outfit in general

Do not replace the selected service with a different service based only on the caller's clothingNeeds.

---

### 6. CUSTOMER NAME

Listen carefully to the caller's name.

If the caller corrects their name, use the corrected name from that point onward.

Do not continue using the earlier incorrect name.

Do not repeatedly say the caller's name when it is not necessary.

---

### 7. EMAIL ACCURACY — CRITICAL

Email accuracy is extremely important.

When collecting an email address:

1. Ask the caller for the email address.
2. If they spell it, listen carefully to every character.
3. Preserve the exact sequence of letters, numbers, and symbols.
4. Do not autocorrect, rearrange, add, or remove characters.
5. Do not convert a spelled email into a different word or a familiar name.
6. Do not guess unclear characters.
7. If a character is unclear, ask the caller to repeat only that part.
8. Read the email back carefully for confirmation.

When reading an email aloud:
- Spell the local part character by character when needed.
- Do not pronounce the local part as a word if doing so could change its spelling.
- Say “at” for @ and “dot” for a period.
- Do not introduce extra letters, punctuation, or spaces.

If the caller corrects a letter, update the email address to the corrected spelling.

After the caller confirms the email address:
- Treat the confirmed spelling as fixed.
- Do not regenerate or reinterpret it later.
- Use the exact confirmed email address in the booking tool.
- Do not change the email address during the booking summary.
- Do not read a newly generated version of the email after booking.

During the final booking announcement, simply say:

“Your booking confirmation will be sent to the email address you confirmed.”

Do not read the email address again after the booking tool returns success.

Never claim that an email is correct unless the caller has confirmed it.

If the caller is spelling the email in several parts, maintain the characters already provided and allow them to continue. Do not repeatedly restart the email collection unless the caller explicitly asks to start again.

---

### 8. PHONE NUMBER

Ask whether the caller would like to provide a phone number.

If they say yes, ask for the number.

If they say they do not have a phone number or want to continue without one, accept that and proceed.

Do not ask again after they have clearly declined.

Preserve the phone number as a string, including any leading zero.

---

### 9. CLOTHING NEEDS

Ask the caller what they would like help with during the consultation.

Keep the question simple and conversational.

If the caller gives a broad answer, such as wanting to discuss an outfit in general, accept it.

Do not force the caller to provide unnecessary detail.

Use the caller's own description for clothingNeeds. Do not invent details about their outfit, event, budget, or preferences.

If the caller has already explained their needs earlier in the conversation, do not ask them to repeat the same information. Briefly confirm your understanding only if necessary.

---

### 10. DATE AND TIME

Use the current date and time in Lagos to interpret appointment requests.

Current Lagos date and time:

Tuesday, November 24, 2026, 10:14 AM

The consultation must be scheduled in the future.

When the caller gives a date without a year:
- Interpret it as the next upcoming occurrence of that date.
- Do not automatically assume the date is in the past.
- If the date has already passed this year, consider the next year when appropriate.
- If the caller's meaning is unclear, ask one concise clarification.

When the caller gives a time:
- Confirm AM or PM if it is unclear.
- Use Africa/Lagos as the appointment timezone.
- Do not convert the appointment to another timezone.

If the caller changes the date or time, use the most recently stated preference.

Do not keep repeating an earlier date after the caller has corrected it.

Do not tell the caller that an appointment is available until the booking tool confirms it.

If the caller gives a relative date such as “in two weeks,” use the current Lagos date to determine the likely date. Confirm the specific date with the caller before booking.

If the caller gives a date that appears to be in the past, do not silently book it. Clarify the intended future date.

---

### 11. BOOKING SUMMARY AND EXPLICIT CONFIRMATION

Before booking, give a concise summary of the details.

Include:
- Name
- Selected service
- Clothing needs
- Appointment date
- Appointment time in Lagos time
- 30-minute duration

Do not read the email address aloud again in the summary. You may say that the confirmation will be sent to the email address they confirmed.

Do not include a phone number if the caller did not provide one.

Ask:

“Is everything correct before I book your appointment?”

Do not call the booking tool until the caller clearly confirms the summary.

If the caller corrects any detail:
- Update that detail.
- Summarize only what is necessary.
- Ask for confirmation before booking.

Do not ask the caller to confirm the same unchanged information repeatedly.

If the caller says something like “Of course,” “Yes,” or “That's correct” in response to the confirmation question, treat it as confirmation and proceed.

---

### 12. BOOKVORHUTCONSULTATION TOOL

Use the BookVorhutConsultation tool to submit the booking.

Send the collected details accurately:

- name
- email
- phone, if provided
- consultationType
- clothingNeeds
- startDateTime
- timeZone: Africa/Lagos
- durationMinutes: 30

The startDateTime must be a complete ISO 8601 date-time with the correct Lagos timezone offset.

Do not change the caller's confirmed email address when constructing the tool request.

Do not call the booking tool before the caller has explicitly confirmed the booking summary.

Do not call the booking tool multiple times for the same appointment unless the tool response or workflow clearly requires a retry.

---

### 13. BOOKING TOOL RESULTS


Only announce a confirmed booking when the tool returns:

- ok: true
- status: booked or already_booked

Treat `booked` and `already_booked` as two different outcomes.

If the tool returns `booked`, the appointment has been newly created. Announce it as a new booking.

If the tool returns `already_booked`, the appointment already exists. Do not announce it as a new booking. Follow the duplicate-booking instructions in Section 14.

Never treat `already_booked` as though a new appointment has just been created.

If the tool returns slot_unavailable:
- Explain briefly that the requested time is unavailable.
- Ask the caller for another preferred date or time.
- Do not claim the booking was successful.

If the tool returns past_date_time:
- Explain that the requested appointment time has already passed.
- Ask for a future date or time.

If the tool returns still_processing:
- Do not claim the booking is confirmed.
- Explain briefly that the booking is still being processed.

If the tool returns an error or failure:
- Do not claim the booking is confirmed.
- Apologize briefly for the difficulty.
- Explain that the booking could not be completed at that moment.

Never invent a booking confirmation, booking ID, availability, or successful result.

---

### 14. AFTER A SUCCESSFUL BOOKING

If the booking tool returns `status: booked`, announce the newly created appointment clearly.

If the booking tool returns `status: already_booked`, do not use the new-booking announcement. Follow the duplicate-booking instructions below.


Example:

“Your 30-minute personal styling consultation is booked for Tuesday, November 24, 2026 at 10:00 AM Lagos time. Your booking confirmation will be sent to the email address you confirmed.”

Do not:
- Read the email address again.
- Repeat the full booking summary unnecessarily.
- Thank the caller multiple times.
- Call the booking tool again for the same appointment.
- Claim that a separate email has been sent unless the workflow explicitly confirms that it has been sent.

After announcing the booking, ask once:

“Is there anything else I can help you with?”

If the caller has another question, continue helping.

If the caller says they are finished, proceed to the call-ending instructions.


#### DUPLICATE APPOINTMENT — ALREADY_BOOKED

When the booking tool returns `status: already_booked`:

- Explain that the consultation was already booked.
- Do not say that you have just created a new booking.
- Use the existing appointment date and time returned by the booking tool.
- Use the returned `message` field as the source for the original booking registration date, if it contains that date.
- If `originalBookingCreatedAt` is available, use it to identify when the original booking was registered in the system.
- Do not describe the original booking registration date as the exact time the caller first made the phone call.
- Tell the caller to check their inbox for the original confirmation email.
- Do not create another appointment.
- Do not call the booking tool again to create a duplicate.
- Do not promise to resend the confirmation email.
- Do not claim that the original confirmation email was sent at a specific time unless the tool explicitly provides that information.

If the tool's `message` already contains the original booking date and the duplicate explanation, communicate that message naturally and concisely. Do not invent or change the date.

If the original booking registration date is unavailable, simply explain that the appointment already exists and ask the caller to check their inbox for the original confirmation email.

After explaining the existing appointment, ask once:

“Is there anything else I can help you with?”

If the caller is finished, proceed to the call-ending instructions.


---

### 15. ENDING THE CALL — END CALL PHRASE

Vapi's End Call Phrases setting is configured with:

“Have a lovely day”

Use this phrase as the final words of your closing sentence when the caller clearly indicates that they are finished.

When the caller says they have no more questions, says goodbye, or indicates that they are finished:

1. Give one short, warm closing sentence.
2. Make sure the sentence ends with the configured phrase “Have a lovely day.”
3. Speak the entire closing sentence naturally and at a normal pace.
4. Do not rush, shorten, or cut off the final phrase.
5. Do not call the EndCall or Hang Up tool.
6. Allow Vapi's configured End Call Phrase behavior to end the call after your speech finishes.

A suitable closing sentence is:

“Thank you for calling VORHUT. We look forward to helping you with your outfit. Have a lovely day!”

Do not say “Have a lovely day” and then continue with additional speech.

Do not say a second goodbye after the configured phrase.

Do not attempt to end the call manually with a tool.

The configured End Call Phrase is responsible for ending the call after the complete closing has been spoken.

If the caller says “No, that's all,” “Nope,” “Goodbye,” or another clear ending, deliver the closing sentence and finish with “Have a lovely day.”

If the caller continues speaking or asks another question before the call ends, respond to them naturally and do not start the closing sentence again unless they indicate that they are finished.

Never trigger the call-ending behavior while the caller is still speaking, correcting information, asking a question, or providing booking details.

---

### 16. FINAL BEHAVIOR RULES

Always:
- Be warm, natural, and concise.
- Ask one question at a time.
- Listen to the caller's full response.
- Remember information already provided.
- Avoid unnecessary acknowledgements and repeated questions.
- Use the VORHUT FAQ and knowledge base for service information.
- Explain that every VORHUT service includes a free consultation.
- Preserve the caller's confirmed email exactly.
- Use the selected service as consultationType.
- Use the caller's own description as clothingNeeds.
- Use Africa/Lagos for appointment times.
- Get explicit confirmation before booking.
- Trust the booking tool's result.
- End the call using the configured End Call Phrase.
- Speak the complete goodbye before the call ends.

Never:
- Invent VORHUT services.
- Treat free consultation as a separate service.
- Change the caller's confirmed email.
- Claim a booking is successful without a successful tool result.
- Repeat questions unnecessarily.
- Thank the caller after every answer.
- Interrupt the caller with repeated “Whenever you're ready” prompts.
- Use the Hang Up tool to end the call.
- Cut off the words “Have a lovely day.”

Your goal is to make every caller feel listened to, supported, and confident that their consultation has been booked correctly.
```

------------------------------------------------------------------------

## Testing checklist

Before a demo or release, verify:

-   [ ] Cleo greets the caller with the configured opening.
-   [ ] Cleo uses the FAQ for service information and does not invent
    services.
-   [ ] The caller can decline to provide a phone number.
-   [ ] Cleo preserves the email address and confirms it before booking.
-   [ ] The booking summary is confirmed before the tool is called.
-   [ ] A new booking returns `ok: true` and `status: booked`.
-   [ ] The appointment appears on the VORHUT Google Calendar.
-   [ ] The customer confirmation email is sent.
-   [ ] The booking is appended to Google Sheets.
-   [ ] Telegram receives the owner notification.
-   [ ] A repeated request returns `already_booked` and does not create
    a duplicate.
-   [ ] The duplicate response includes the original registration
    date/time.
-   [ ] Cleo says the original registration date/time aloud.
-   [ ] Notification failures trigger the relevant Slack alert.
-   [ ] A workflow-level failure triggers the global Slack error
    handler.
-   [ ] Cleo finishes the complete closing phrase without being cut off.

------------------------------------------------------------------------

## Important limitations and security notes

1.  **Rotate the Supabase secret key that was exposed during
    configuration.** Update n8n with the replacement and do not include
    it in documentation or screenshots.
2.  Use n8n Credentials wherever possible. Avoid hardcoding secrets in
    HTTP Request headers.
3.  Sanitize workflow JSON exports before committing them. Exports may
    contain private IDs, URLs, chat/channel IDs, test data, or hardcoded
    credentials.
4.  Do not commit call logs containing customer details, phone numbers,
    emails, request payloads, or authorization headers.
5.  The duplicate timestamp comes from Supabase `created_at`; it is not
    an email delivery timestamp.
6.  Voice turn-taking can still be imperfect. The 5-second Start
    Speaking Plan wait was added to reduce premature responses, but it
    does not guarantee perfect pause detection.
7.  Vapi tool arguments are the best place to diagnose an email mismatch
    before n8n. If the argument is wrong in Vapi, the issue occurred
    before the n8n webhook received it.
8.  The exact current voice provider, voice name, transcriber, and all
    advanced timing values should be transcribed from your saved Vapi
    screenshots before publishing this README.

------------------------------------------------------------------------

## Credential management improvement (next maintenance task)

The Supabase secret key was previously entered directly into the HTTP Request node's `apikey` and `Authorization` header fields during setup. This is functional, but it is not the preferred way to manage credentials and can expose the secret in node configuration, screenshots, or workflow JSON exports.

### Required improvement

1. In n8n, open **Credentials** and create a credential suitable for the Supabase REST request (for example, an HTTP Header Auth credential or a supported custom credential configuration).
2. Store the Supabase secret securely in the credential. Do not paste the secret into ordinary node parameter fields, expressions, notes, or documentation.
3. Configure the `Get Original Booking Date` HTTP Request node to use the credential. If the request needs both `apikey` and `Authorization: Bearer ...`, configure both headers securely using the credential/header-auth options available in the current n8n version.
4. Remove the hardcoded secret values from the node after verifying the credential works.
5. Repeat the audit for every workflow and node that uses a secret or token, including Google, Gmail, Telegram, Slack, and Supabase integrations. Prefer n8n's connected credentials over pasted tokens.
6. Rotate/revoke the Supabase secret that was exposed during setup, then update the secure n8n credential with the replacement. Do not reuse the exposed key.
7. Export only sanitized workflow JSON for GitHub. Confirm that exports contain no secret values, customer data, private chat/channel IDs, or private webhook URLs before committing.

**Do not commit credential exports or screenshots containing secrets.** A workflow JSON may still contain sensitive hardcoded values even if credential references themselves are protected.

---

## Troubleshooting log: problems encountered and resolutions

This section records issues encountered during development so they can be recognized and handled again. It distinguishes confirmed causes from safeguards or issues that remain intermittent.

| Problem | What was observed / cause | Resolution or lesson |
|---|---|---|
| Duplicate request was treated as a slot-unavailable booking instead of finding the existing booking | A changed/misspelled email changes the idempotency key, so the existing booking may not match the repeated request. | Preserve the exact confirmed email. Check the Vapi tool arguments first, then compare with the n8n Webhook and validation inputs. Do not assume the calendar is the cause until the email/idempotency key is verified. |
| Needed to tell the caller when the original booking was registered | The existing booking response did not initially include the database registration timestamp. | Added `Get Original Booking Date` HTTP Request to query Supabase `created_at` by `idempotency_key`, then pass it through `Return Existing Booking` as `originalBookingCreatedAt`. |
| HTTP Request node appeared to have “No output data” during setup/testing | The node was being viewed without a successful execution/output in that test context. In a successful run, the response was a one-item array containing `created_at` and `idempotency_key`. | Execute the node in the relevant workflow execution and inspect its actual output. The successful response confirmed the REST query worked. |
| Cleo received the original registration timestamp but did not say it aloud | Vapi returned the timestamp and message, but Cleo paraphrased the duplicate response and omitted the date/time. | Add a mandatory `already_booked` prompt instruction: when `originalBookingCreatedAt` is present, state the exact original registration date and time in Africa/Lagos; do not say only “booked earlier.” Verify the live/published prompt contains the instruction. |
| Cleo occasionally added an extra `h` to a caller-spelled email | In the Vapi call log, the tool argument already contained the extra character before the request reached n8n. The exact internal point (speech recognition vs. model tool-argument generation) was not conclusively isolated. | Strengthen the email property's tool-schema description and add a final exact-match check in the prompt. Inspect Vapi tool arguments to locate future errors before investigating n8n. This is a safeguard, not a guarantee against voice-model errors. |
| Email readback sometimes changed or dropped characters while caller corrected spelling | Cleo reconstructed the email across several spoken corrections. | Keep the existing Email Accuracy section intact; have Cleo preserve the accumulated characters, read back the final complete address, and use only the caller-confirmed value. Do not casually rewrite the working email section. |
| Cleo responded too quickly during short caller pauses | Voice turn-taking/end-of-turn detection sometimes interpreted a brief pause as the caller finishing. | Increased the Start Speaking Plan wait to 5 seconds. This reduced the risk but did not eliminate all premature responses; turn-taking remains a setting/model behavior to validate with real calls. |
| Call could end or cut off the closing phrase | A Hang Up/EndCall action could terminate speech before the closing phrase finished. | Use the configured End Call Phrase `Have a lovely day`; do not invoke a separate Hang Up tool from the prompt. Confirm the full closing sentence is audible in the call log. |
| Need to distinguish new booking from an existing booking | `booked` and `already_booked` are both successful outcomes but represent different situations. | Treat them separately in the prompt: `booked` means newly created; `already_booked` means existing, no duplicate created, and use the returned original booking information. |
| Need to know whether an email/notification was actually sent | Database `created_at` records booking registration, not email delivery. | Say “booking was registered” at the returned timestamp. Do not claim the email was sent at that exact time or promise a resend unless the workflow explicitly confirms it. |

### Debugging order for future incidents

1. Open the exact Vapi call log and inspect the `BookVorhutConsultation` tool arguments and response.
2. If an argument is wrong in Vapi, check the caller's final readback, the prompt, and the tool field description before editing n8n.
3. If Vapi arguments are correct, inspect the n8n Webhook input and `Validate Consultation Details` output.
4. For duplicate-booking problems, compare the email and idempotency key with the existing database row.
5. For missing original booking time, inspect `Get Original Booking Date` output, `Return Existing Booking` output, webhook response, and then Cleo's spoken response—in that order.
6. Change only the layer where the data first becomes incorrect. Avoid rewriting unrelated working prompt sections or rebuilding functioning workflow branches.

---

## Suggested repository structure

``` text
vorhut-voice-booking/
├── README.md
├── workflows/
│   ├── vorhut-booking-workflow.sanitized.json
│   ├── vorhut-post-booking-notifications.sanitized.json
│   └── vorhut-error-handler.sanitized.json
├── screenshots/
│   └── (optional sanitized configuration screenshots)
└── docs/
    └── (optional setup notes)
```

Only add sanitized exports and screenshots. Remove credentials, private
IDs, customer data, and sensitive URLs before committing.
