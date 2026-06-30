# Task 03 — Integration Design: Landing Page to HubSpot, WhatsApp, and Google Ads

## End-to-end architecture

I'd skip a no-code middle layer like Zapier or Make here and go with a **direct API call from the landing page form to HubSpot's Forms Submission API**, triggered server-side rather than client-side. Here's why: Zapier/Make add a polling or webhook-trigger delay that can easily eat into the 2-minute WhatsApp SLA, and they introduce a third-party dependency for something that's a simple, well-documented API call. A native HubSpot embed is even worse here — it gives us no control over what happens after submission, and we need to fire two more downstream actions (WhatsApp, Ads conversion) immediately after the contact is created.

So the flow looks like this: the landing page form posts to a small serverless function (e.g., a single endpoint on Vercel or AWS Lambda) instead of directly to HubSpot from the browser. That function does three things in sequence:

1. Calls the **HubSpot Contacts API** (not the public Forms API) to create or update the contact, searching first by phone number before deciding create vs. update.
2. On success, immediately calls the **Karix WhatsApp Business API** to send the confirmation message.
3. Fires the **Google Ads conversion** via the dataLayer push already wired on the front end (covered in Task 2), which GTM picks up and forwards via the Enhanced Conversions / offline conversion path if needed.

Routing through a backend function instead of three separate client-side calls means we control retries and logging in one place, and the browser doesn't need direct, exposed access to HubSpot or Karix credentials.

## The phone deduplication trap

This is the part most integrations get wrong: **HubSpot's default deduplication key is email, not phone**. Since this form deliberately doesn't collect email, two different patients submitting from the same phone number (a very real scenario in Indian healthcare lead gen — shared family phones are common) won't get deduplicated by default; HubSpot will happily create two separate contact records, or worse, silently overwrite one patient's name with another's if we naively "update by phone" without checking.

My fix: before calling the create/update endpoint, I'd run a **search query against HubSpot using the phone property as a custom unique identifier**, and only update an existing record if the name also reasonably matches (or flag it for manual review in a "Possible Duplicate" list view if it doesn't). New, mismatched-name submissions on an existing phone number get created as a fresh contact with a note linking the two, rather than overwriting silently.

## Failure point and SLA monitoring

The single biggest failure point is the **WhatsApp send via Karix failing silently** (rate limits, template approval issues, or Karix downtime) — since it's the last step and depends on two prior API calls succeeding. My fallback: queue the WhatsApp send as a retryable job rather than a synchronous call, with 2-3 retries over 90 seconds, and if it still fails, auto-trigger an SMS fallback and alert the front-desk team via Slack so a human can call the patient directly. To monitor the SLA itself, I'd log a timestamp at form submission and at WhatsApp delivery confirmation (Karix returns delivery webhooks), then run a simple dashboard or alert if the gap exceeds 2 minutes on more than a small percentage of daily submissions.