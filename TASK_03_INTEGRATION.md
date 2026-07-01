# Task 03 – Integration Design (Landing Page → HubSpot + Karix + Google Ads)

## Goal

When a patient submits the consultation form on the landing page, three things must happen automatically, reliably, and within ~2 minutes:

1. Create or update a contact in **HubSpot CRM**.
2. Send a **WhatsApp confirmation** via **Karix** within 2 minutes.
3. Fire the `consultation_form_submitted` **Google Ads conversion** (server-side, de-duped with the client-side push).

## High-level architecture

```
```

The endpoint is the single source of truth. The browser only collects the data and hands it off – every external system is called from the server, where API keys live safely and retries are possible.

## End-to-end flow

1. **Client.** User submits the form. We:
   - generate `lead_reference_id` (client-side, e.g. `ON-LX9F2A`)
   - read `gclid` from the URL / `_gcl_aw` cookie (set by Google Ads auto-tagging)
   - push `consultation_form_submitted` to `dataLayer` (browser-side Ads/GA4 conversion)
   - `POST /api/lead` with the full payload, including `gclid` and `lead_reference_id`
   - render the thank-you state immediately (do not block UI on the API)

2. **Lead Intake API (server).** Runs in this order, with each step independently retried:
   1. **Validate** payload (Zod): phone is 10–13 digits, clinic is one of the 9, name length ok.
   2. **HubSpot upsert** – `PATCH /crm/v3/objects/contacts/{id}?idProperty=phone` (HubSpot supports lookup-by-phone). If 404, fall back to `POST /crm/v3/objects/contacts`. Body:
      ```json
      {
        "properties": {
          "firstname": "Rahul",
          "lastname": "Sharma",
          "phone": "+919812345678",
          "clinic_preference": "Indiranagar, Bengaluru",
          "hs_lead_status": "NEW",
          "lifecyclestage": "lead",
          "lead_source": "Google Ads - Consultation Landing Page",
          "pain_area": "Knee pain",
          "lead_reference_id": "ON-LX9F2A",
          "gclid": "Cj0KCQjw..."
        }
      }
      ```
      `clinic_preference`, `pain_area`, `lead_reference_id` and `gclid` are **custom contact properties** created once in HubSpot. We store `gclid` on the contact so HubSpot workflows can later upload **offline conversions** (booked / attended) back to Google Ads.

   3. **Queue WhatsApp send.** Push a job onto a lightweight queue (e.g. Cloudflare Queue / Upstash QStash) so the HTTP response stays under 200 ms and the WhatsApp call has its own retry surface. Worker picks the job and calls Karix:
      ```
      POST https://api.karix.com/v3/whatsapp/messages
      {
        "to": "+919812345678",
        "type": "template",
        "template": {
          "name": "orthonow_consultation_confirm",
          "language": "en",
          "components": [
            { "type":"body",
              "parameters":[
                {"type":"text","text":"Rahul"},
                {"type":"text","text":"Indiranagar, Bengaluru"},
                {"type":"text","text":"ON-LX9F2A"}
              ]}
          ]
        }
      }
      ```
      The template is pre-approved with Meta (mandatory for business-initiated WhatsApp messages). 2-minute SLA is comfortable because the queue + Karix typically deliver in seconds.

   4. **Fire Google Ads server-side conversion.** Use the **Google Ads API – ClickConversion upload** (or Enhanced Conversions for Leads with the hashed phone if `gclid` is missing):
      ```json
      {
        "conversions": [{
          "conversionAction": "customers/123/conversionActions/456",
          "conversionDateTime": "2026-06-29 14:02:11+05:30",
          "conversionValue": 500,
          "currencyCode": "INR",
          "orderId": "ON-LX9F2A",
          "gclid": "Cj0KCQjw..."
        }]
      }
      ```
      `orderId = lead_reference_id` is the **same value** the browser-side Ads tag sent, so Google de-duplicates – we don't double-count.

3. **HubSpot workflow (downstream, no code).** When `hs_lead_status` flips to *Qualified* or *Booked*, a HubSpot workflow uploads the offline conversion back to Google Ads using the stored `gclid`. This is what unlocks value-based bidding in month 2.

## Why this shape

- **Single intake endpoint.** Every system (HubSpot, Karix, Ads) gets the same canonical payload, so a failure in one doesn't break the others. The browser never holds an API key.
- **Queue the WhatsApp call.** Karix occasionally rate-limits or returns transient 5xx. A queue gives us free retries and the user never waits.
- **Server-side Ads conversion + client-side tag, de-duped by `order_id`.** Catches users who churn before the GTM tag fires (ad blockers, ITP, mobile network drops) without inflating counts. This is Google's recommended pattern for lead-gen.
- **`gclid` on the contact.** Without it, you can't upload qualified/booked offline conversions later, and you'll be stuck optimising on form fills forever.
- **Phone as the dedupe key in HubSpot.** Patients re-enquire from different devices; email is often missing in healthcare lead-gen, phone is always there.

## Failure modes & guardrails

| Failure | What happens | Mitigation |
|---|---|---|
| HubSpot 5xx | API returns 502 to the queue worker | Exponential retry up to 1 hour; on final failure, write the raw payload to a `failed_leads` log + Slack alert to ops |
| Karix template rejected / phone invalid | WhatsApp not delivered | Fallback SMS via the same intake endpoint after 5 min if no delivery callback |
| Google Ads API token expired | Conversion not uploaded | Refresh token rotation cron; the client-side GTM tag still fires, so Ads still sees the conversion |
| Duplicate submits (user double-clicks) | 2 contacts, 2 WhatsApps | Client disables the button on submit; server de-dupes on `lead_reference_id` for 24h in KV |
| Spam / bot fills | Junk in CRM | hCaptcha invisible on the form + simple honeypot field; server rejects if either fails |

## Secrets

- `HUBSPOT_PRIVATE_APP_TOKEN`
- `KARIX_API_KEY`
- `GOOGLE_ADS_DEVELOPER_TOKEN`, `GOOGLE_ADS_CLIENT_ID`, `GOOGLE_ADS_CLIENT_SECRET`, `GOOGLE_ADS_REFRESH_TOKEN`, `GOOGLE_ADS_CUSTOMER_ID`, `GOOGLE_ADS_CONVERSION_ACTION_ID`

All live as server-side env vars / secret manager entries. None ever reach the browser.
