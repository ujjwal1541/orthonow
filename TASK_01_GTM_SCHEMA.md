# Task 01 – GTM Event Schema for OrthoNow

## 1. Full event schema

All events are pushed to `window.dataLayer` and picked up by a single GTM container that forwards them to GA4 (and selectively to Google Ads).

| # | Event Name | Trigger Type (in GTM) | Key Parameters (≥3) | GA4 destination |
|---|---|---|---|---|
| 1 | `page_view` | History Change + Initialisation (built-in GA4 config tag) | `page_path`, `page_location`, `page_type` (home / clinic / landing / blog), `city` | GA4 default — Pages and screens report |
| 2 | `clinic_page_view` | Custom Event trigger, fires after `page_view` when `page_type = clinic` | `clinic_name`, `clinic_city`, `clinic_id`, `page_path` | GA4 audience: *Viewed a clinic page* → remarketing in Google Ads |
| 3 | `click_call` | Click trigger on `a[href^="tel:"]` + dataLayer push from the front-end | `source` (header / sticky_mobile / clinic_card), `page_type`, `clinic_name` (if applicable), `phone_number` | GA4 Key Event → Google Ads secondary conversion |
| 4 | `whatsapp_click` | Click trigger on `a[href*="wa.me"]` | `source`, `page_type`, `clinic_name` | GA4 Key Event; feeds *WhatsApp leads* audience |
| 5 | `booking_step_view` | Custom Event – pushed by the front-end when each step renders | `step_number`, `step_name`, `clinic_location`, `specialty` | GA4 Funnel Exploration (step entries) |
| 6 | `booking_step_complete` | Custom Event – pushed by the front-end on step-forward click | `step_number`, `step_name`, `clinic_location`, `specialty`, `time_on_step_sec` | GA4 Funnel Exploration (step exits) |
| 7 | `booking_confirmed` | Custom Event – pushed by the front-end on final confirmation API success | `clinic_location`, `specialty`, `appointment_date`, `booking_id`, `value`, `currency` | **Primary** GA4 Key Event → **Google Ads primary conversion** |
| 8 | `consultation_form_submitted` | Custom Event – pushed by the landing page on form submit | `form_name`, `clinic_preference`, `pain_area`, `lead_reference_id`, `campaign` | GA4 Key Event → Google Ads conversion for the LP campaign |
| 9 | `patient_guide_form_submitted` | Custom Event – pushed on gated form submit | `guide_topic`, `clinic_preference` (optional), `lead_reference_id` | GA4 Key Event; feeds *Top-of-funnel leads* audience |
| 10 | `patient_guide_downloaded` | Click trigger on the actual PDF link AFTER form submit | `guide_topic`, `file_name`, `lead_reference_id` | GA4 Event; confirms guide delivery (separates form-only vs full download) |
| 11 | `blog_scroll_depth` | Built-in Scroll Depth trigger (25 / 50 / 75 / 100) limited to `page_type = blog` | `article_slug`, `article_category`, `scroll_percent`, `read_time_sec` | GA4 audience: *Engaged readers* (≥75%) for remarketing |
| 12 | `blog_article_complete` | Custom Event fired when `scroll_percent = 100 AND read_time_sec ≥ 60` | `article_slug`, `article_category`, `read_time_sec` | GA4 Key Event – content effectiveness report |
| 13 | `form_field_error` | Form trigger – pushed on inline validation failure | `form_name`, `field_name`, `error_type` | GA4 exploration – diagnose form drop-off causes |

Notes on parameter strategy:
- `clinic_location` / `clinic_name` / `clinic_id` are normalised – the front-end always uses the same canonical names (e.g. `"Indiranagar, Bengaluru"`) so GA4 dimensions stay clean.
- `lead_reference_id` is generated client-side and reused server-side so a lead can be stitched across GA4 ↔ HubSpot ↔ Google Ads (used as the Ads `order_id` for de-duplication).
- All custom parameters are registered as **GA4 Custom Dimensions** in the Admin UI before traffic starts – otherwise GA4 silently drops them from reports.

---

## 2. The 3-step booking form – funnel tracking

### Who writes the dataLayer push?
**The front-end developer writes the pushes; I (the GTM owner) write the spec, the triggers, the tags and the GA4 funnel.** GTM cannot natively see internal state changes in a multi-step React/JS form – the only thing the DOM exposes is a click on a "Next" button, and that click doesn't tell GTM which step was completed, which clinic was chosen, or whether the step actually validated. So the front-end must explicitly emit a `dataLayer.push` at three moments per step:

1. When the step **renders** → `booking_step_view`
2. When the user successfully **completes** the step (validation passed, moving forward) → `booking_step_complete`
3. On final API success → `booking_confirmed`

### Brief to the front-end dev (for Step 2 specifically)
> "On Step 2 (name / phone / preferred date), please add two `window.dataLayer.push` calls:
> - One when the step component **mounts** (use `useEffect` / `mounted`) → `booking_step_view` with `step_number: 2`.
> - One inside the **submit handler, AFTER your client-side validation passes** and BEFORE the navigation to Step 3 → `booking_step_complete` with `step_number: 2`.
> Do **not** fire on button click directly – that would log even invalid attempts and inflate the step-2 completion count. Carry forward `clinic_location` and `specialty` from Step 1 so every step has the full context. Initialise `window.dataLayer = window.dataLayer || []` once in the app shell before any other script runs."

### Actual dataLayer pushes

```json
{
  "event": "booking_step_view",
  "step_number": 1,
  "step_name": "location_specialty",
  "page_type": "booking_form"
}
```

```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "Indiranagar, Bengaluru",
  "specialty": "Knee & Joint",
  "time_on_step_sec": 14
}
```

```json
{
  "event": "booking_step_view",
  "step_number": 2,
  "step_name": "patient_details",
  "clinic_location": "Indiranagar, Bengaluru",
  "specialty": "Knee & Joint"
}
```

```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "patient_details_entered",
  "clinic_location": "Indiranagar, Bengaluru",
  "specialty": "Knee & Joint",
  "preferred_date": "2026-07-05",
  "lead_reference_id": "ON-LX9F2A",
  "time_on_step_sec": 38
}
```

```json
{
  "event": "booking_step_view",
  "step_number": 3,
  "step_name": "confirm_booking",
  "clinic_location": "Indiranagar, Bengaluru",
  "specialty": "Knee & Joint",
  "preferred_date": "2026-07-05"
}
```

```json
{
  "event": "booking_confirmed",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "Indiranagar, Bengaluru",
  "specialty": "Knee & Joint",
  "appointment_date": "2026-07-05",
  "booking_id": "ON-LX9F2A",
  "lead_reference_id": "ON-LX9F2A",
  "value": 500,
  "currency": "INR"
}
```

### GTM wiring
- **Variables** (Data Layer Variable): `dlv.step_number`, `dlv.step_name`, `dlv.clinic_location`, `dlv.specialty`, `dlv.booking_id`, `dlv.lead_reference_id`, `dlv.value`, `dlv.currency`.
- **Triggers**: one Custom Event trigger per event name (`booking_step_view`, `booking_step_complete`, `booking_confirmed`).
- **Tags**: GA4 Event tag per trigger, mapping the variables above as event parameters. `booking_confirmed` also fires a Google Ads Conversion tag (`order_id` = `lead_reference_id` for de-duplication).

### Surfacing drop-off in GA4 Funnel Exploration
In GA4 → **Explore → Funnel exploration**, build a 4-step **open funnel**:

1. Step 1 – Event = `booking_step_view` AND `step_number = 1`
2. Step 2 – Event = `booking_step_complete` AND `step_number = 1`
3. Step 3 – Event = `booking_step_complete` AND `step_number = 2`
4. Step 4 – Event = `booking_confirmed`

Breakdown dimension: `clinic_location`. Segment: device category. This shows absolute drop-off between every step *and* lets us see, for example, that the Whitefield clinic loses 70% at Step 2 on mobile – which is a UX brief, not a marketing one.

---

## 3. The one conversion I would import into Google Ads

**`booking_confirmed`** – imported as the **primary** conversion action in Google Ads, with category *Submit Lead Form* and a value of ₹500 (avg. consultation revenue, refined later with offline conversions for booked-and-attended visits).

Why this one over the others:

- It is the **only event that represents real business value** – an appointment that the clinic can actually staff. `consultation_form_submitted`, `click_call` and `whatsapp_click` are upper-funnel signals, valuable but noisy (spam, mis-clicks, drop-offs after the call).
- Smart Bidding optimises toward whatever signal you give it. If you let it optimise toward `consultation_form_submitted`, you'll get lots of cheap form fills but no bookings. Optimising toward `booking_confirmed` forces the algorithm to find users who actually convert at the bottom of the funnel.
- Because we pass `lead_reference_id` as the Ads `order_id`, it de-duplicates cleanly with the offline conversion we'll later upload from HubSpot ("Booked & Attended"), letting us move to **value-based bidding** in month 2 without rebuilding the conversion.

The other events stay in Google Ads as **secondary ("Observation-only") conversions** so we can monitor them, but they do not influence bidding.
