# Task 01 — GTM Event Schema for OrthoNow

## Context

OrthoNow currently has GA4 installed with pageview tracking only — there's no GTM container, no event-level visibility, and no way for the performance marketing team to know which interactions are actually driving bookings. Before any paid spend goes live, we need a tracking layer that captures every meaningful interaction on the site and maps it to something usable in GA4 and, eventually, Google Ads.

Below is the full event schema, followed by a deep-dive on the booking funnel (since that's the highest-value and most fragile part of this setup), and a recommendation on which single event to import as a Google Ads conversion.

---

## 1. Full Event Schema

| Event Name | Trigger Type | Key Parameters | GA4 Report / Audience It Feeds |
|---|---|---|---|
| `call_now_click` | Click trigger, fires on all elements matching `.call-now-btn` (homepage, clinic pages, landing page) | `page_location`, `clinic_name` (blank if not on a clinic page), `button_position` (header / footer / sticky-bar) | Engagement → Events report. Feeds a "Call Intent" audience for remarketing/lookalikes. |
| `whatsapp_widget_open` | Click trigger on the floating WhatsApp button (before the `wa.me` redirect fires) | `page_location`, `device_category`, `scroll_depth_at_click` | Engagement → Events. Feeds a "WhatsApp Engaged" remarketing audience. |
| `patient_guide_download` | Form submission trigger on the gated PDF form (name + phone) | `form_id`, `pdf_name`, `lead_source` | Conversions report. Feeds a top-of-funnel lead magnet audience for nurture campaigns — not the primary Ads conversion. |
| `clinic_page_view` | Page view trigger filtered by URL path `/clinics/*` (fires once per of the 9 clinic pages) | `clinic_name`, `clinic_city`, `page_referrer` | Pages and Screens report. Feeds city-specific interest audiences (Bengaluru / Hyderabad / Chennai). |
| `blog_scroll_75` | Scroll depth trigger fired at 75% on blog article templates | `article_title`, `article_category`, `time_on_page` | Engagement report. Feeds a "Content Engaged" audience used for retargeting toward the booking page. |
| `booking_step_1_complete` | Custom event trigger listening for a dataLayer push (see Section 2) | `step_number`, `clinic_location`, `specialty` | Funnel Exploration — Step 1 node. |
| `booking_step_2_complete` | Custom event trigger listening for a dataLayer push | `step_number`, `contact_details_filled` (boolean — never push raw PII into GTM/GA4), `preferred_date` | Funnel Exploration — Step 2 node. |
| `booking_confirmed` | Custom event trigger listening for a dataLayer push on final confirmation | `step_number`, `clinic_location`, `specialty`, `booking_id` | Conversions report — this is the primary conversion event, and the one imported into Google Ads (see Section 3). |

A note on parameters: for `booking_step_2_complete` I'm deliberately not pushing the patient's actual name or phone number into the dataLayer. GTM containers and GA4 are not PII-safe by default, and pushing raw contact details risks both a Google Ads policy violation and a compliance issue given this is healthcare data. I'd push a boolean flag instead and let the CRM (HubSpot, covered in Task 3) hold the actual PII.

---

## 2. Booking Funnel — Step-by-Step Tracking

This is the part of the schema that actually needs engineering effort, not just configuration.

GTM's built-in triggers (Click, Form Submission, Page View, Scroll Depth) only fire off real DOM events or page-load events. A 3-step booking form that switches steps via JavaScript — without a page reload or a real form submission between steps — gives GTM nothing native to listen for. There is no "step changed" trigger in GTM out of the box. The only way to track step-level progression is for the **front-end developer** to manually fire a `dataLayer.push()` at the moment each step completes, and for GTM to then catch that push using a Custom Event trigger.

So the division of responsibility is: I define the event names, parameters, and exactly when each push should fire. The front-end dev is the one who wires the actual `dataLayer.push()` call into the step-transition logic (e.g., inside the "Next" button's click handler, after validation passes).

Here's how I'd brief that to the front-end team, step by step:

**Step 1 — Location + Specialty selected**, fires when the user clicks "Next" after selecting both fields:

```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}"
}
```

**Step 2 — Contact details entered**, fires when the user clicks "Next" after name, phone, and preferred date pass validation:

```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_details_entered",
  "clinic_location": "{{clinic name}}",
  "preferred_date": "{{preferred date}}"
}
```

**Step 3 — Booking confirmed**, fires on the final "Confirm Booking" click, after the booking is actually accepted by the backend (not just on button click — this should wait for a success response, otherwise we'd count failed submissions as conversions):

```json
{
  "event": "booking_confirmed",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}",
  "booking_id": "{{booking id from backend response}}"
}
```

In GTM, I'd set up three Custom Event triggers — one per event name/step combination — each firing a corresponding GA4 event tag. Each tag passes through the relevant parameters as event-scoped dimensions.

To surface drop-off in **GA4 Funnel Exploration**: I'd build a funnel with three open steps — `booking_step_complete` (step_number = 1) → `booking_step_complete` (step_number = 2) → `booking_confirmed`. GA4's funnel visualization will show the percentage of users who completed each step and dropped at each transition, which tells the marketing team exactly where the form is leaking — e.g., if most users complete step 1 (low-commitment, just picking a clinic) but drop heavily before step 2 (entering personal contact details), that's a strong signal the form is asking for too much too early, or that there's friction/trust issue at that exact point.

---

## 3. Which Conversion to Import into Google Ads

I'd import **`booking_confirmed`** as the Google Ads conversion action — not `call_now_click`, not `whatsapp_widget_open`, and not `patient_guide_download`.

Reasoning: Google Ads' bidding algorithms (Target CPA, Maximise Conversions) optimise toward whatever signal you feed them. Call clicks and WhatsApp opens are *intent* signals — a meaningful chunk of those never convert into an actual booking, and treating them as the conversion would train the algorithm to chase cheap clicks rather than real patients. `patient_guide_download` is even further upstream — it's a lead magnet for nurture, not a booking signal at all.

`booking_confirmed` is the only event in this schema that represents a completed, backend-confirmed action with real commercial value to OrthoNow. It's the cleanest, least noisy signal to optimise spend against. Once volume is high enough, I'd consider layering in a secondary "micro-conversion" (like `booking_step_2_complete`) as an *observation-only* signal for Smart Bidding — but the primary, bid-optimising conversion should stay `booking_confirmed`.