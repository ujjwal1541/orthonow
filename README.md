# OrthoNow – Namoza Developer Assignment

Three deliverables for OrthoNow's first-month engagement:

1. **Task 01 – GTM Event Schema** → see [`TASK_01_GTM_SCHEMA.md`](./TASK_01_GTM_SCHEMA.md)
2. **Task 02 – Landing Page** → see [`index.html`](./index.html) (single self-contained file)
3. **Task 03 – Integration Design** → see [`TASK_03_INTEGRATION.md`](./TASK_03_INTEGRATION.md)

---

## How to run the landing page

It is a single static HTML file – no build, no server, no dependencies.

**Option A – Double-click**
Just open `index.html` in any modern browser.

**Option B – Local server (recommended for PageSpeed/Lighthouse)**

```bash

python3 -m http.server 8080

```

### Verifying the GTM dataLayer push

1. Open `index.html` in Chrome.
2. Open DevTools → **Console**.
3. Type `window.dataLayer` and press Enter – you'll see an empty array on page load (intentional: no `page_view` here, GTM container handles that).
4. Fill in the form and submit. The console will show a new entry:

```js
{
  event: "consultation_form_submitted",
  form_name: "lp_book_consultation",
  clinic_preference: "Indiranagar, Bengaluru",
  pain_area: "Knee pain",
  lead_reference_id: "ON-XXXXXX",
  page_type: "landing_page",
  campaign: "gads_consultation_lp"
}
```

The thank-you state appears without a page reload.

### PageSpeed notes

The file ships with **no web fonts, no external CSS, no images over the wire** (inline SVG/text only) and inline critical CSS, so first paint is effectively immediate. When you add the real GTM container snippet, load it with `async` and place it just before `</body>` to keep the mobile score in the 90+ range.

---

## File tree

```

```
