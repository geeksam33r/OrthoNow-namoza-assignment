# Task 01 — GTM Event Schema: OrthoNow

## Why this architecture

GTM listens for two kinds of signals:
1. **Native browser events** it can detect on its own — page loads, link clicks, form `submit` events, scroll position, visibility.
2. **Custom events pushed into `dataLayer`** by the front-end — anything that happens *inside* an app state (a multi-step form advancing, a modal opening, an SPA route change) that never triggers a real page load or a native DOM event GTM recognizes.

The 3-step booking form is case #2. Steps 2 and 3 happen without a URL change or full page reload, so GTM has no native hook. The front-end dev must call `dataLayer.push()` at the moment each step's validation passes. GTM's job is only to listen for that push via a **Custom Event trigger** and forward it to GA4. This is the single most important design decision in this schema — flagged explicitly below.

---

## Full Event Schema

| Event Name | Trigger Type | Key Parameters | Feeds Into (GA4) |
|---|---|---|---|
| `clinic_page_view` | Page View — trigger fires on Page Path matching `/clinics/*` | `clinic_name`, `clinic_city`, `page_path` | Engagement > Pages and Screens report; remarketing audience "Viewed [City] Clinic" |
| `call_now_click` | Click – Just Links, Trigger ID `call-now-btn` (fires on all instances: homepage, 9 clinic pages, landing page) | `click_location` (homepage / clinic_page / landing_page), `clinic_name`, `phone_number_clicked` | Events report; marked as GA4 Key Event (conversion); feeds "High Intent — Call Click" audience |
| `whatsapp_click` | Click – Just Links, Trigger fires when Click URL contains `wa.me` | `click_location`, `page_path`, `clinic_name` (if applicable) | Events report; secondary conversion; excluded from primary Ads conversion (see reasoning below) |
| `guide_lead_captured` | Custom Event — `dataLayer.push` fired on successful validation of the gated name+phone form (before PDF unlocks) | `form_location`, `guide_topic`, `lead_source` | Events report; feeds CRM-bound lead audience; **not** the file download itself |
| `guide_file_download` | GA4 Enhanced Measurement "File Download" (native, auto-fires on the PDF link click once the gate is passed) | `file_name`, `file_extension`, `link_url` | Engagement > File Downloads; confirms actual content consumption post-lead-capture |
| `booking_step_complete` (step 1) | Custom Event — `dataLayer.push` on step 1 validation pass | `step_number: 1`, `step_name: location_specialty_selected`, `clinic_location`, `specialty` | Funnel Exploration (step 1); GA4 Key Event NOT set here — too early-funnel |
| `booking_step_complete` (step 2) | Custom Event — `dataLayer.push` on step 2 validation pass | `step_number: 2`, `step_name: contact_info_entered`, `clinic_location`, `preferred_date` | Funnel Exploration (step 2) |
| `booking_step_complete` (step 3 / confirmation) | Custom Event — `dataLayer.push` on booking confirmation from server response | `step_number: 3`, `step_name: booking_confirmed`, `clinic_location`, `specialty`, `booking_id` | Funnel Exploration (step 3); **primary GA4 Key Event**; imported into Google Ads (see below) |
| `blog_scroll_depth` | Scroll Depth trigger (native GTM trigger type), thresholds set at 25/50/75/90% | `percent_scrolled`, `page_path`, `article_title` | Engagement report; feeds "Engaged Reader" audience for blog-to-consultation remarketing |

**Note on PII:** none of the above ever pass `name` as a GA4 event parameter. Name is PII and GA4's terms prohibit sending it. Phone number is also excluded from GA4-bound parameters — it lives in HubSpot only (see Task 3).

---

## Booking Funnel: Drop-off Tracking

**What fires at each step, and who fires it:**

- Steps 1–3 all use the *same* custom event name, `booking_step_complete`, differentiated by a `step_number` parameter. This keeps the GTM setup to a single Custom Event trigger rather than three separate ones.
- The front-end dev pushes to `dataLayer` at three moments:
  - After step 1 form fields (clinic + specialty) pass client-side validation and the user clicks "Next"
  - After step 2 fields (name/phone/date) pass validation and the user clicks "Next"
  - After the server confirms the booking was actually created (not just "form submitted" — you want a *confirmed* booking, since step 3 can fail server-side even if the client-side form looks fine)

**Actual dataLayer JSON:**

```json
{
  "event": "booking_step_complete",
  "step_number": 1,
  "step_name": "location_specialty_selected",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}"
}
```

```json
{
  "event": "booking_step_complete",
  "step_number": 2,
  "step_name": "contact_info_entered",
  "clinic_location": "{{clinic name}}",
  "preferred_date": "{{preferred date}}"
}
```

```json
{
  "event": "booking_step_complete",
  "step_number": 3,
  "step_name": "booking_confirmed",
  "clinic_location": "{{clinic name}}",
  "specialty": "{{specialty selected}}",
  "booking_id": "{{booking id}}"
}
```

**GTM setup:**
1. One Custom Event trigger: fires when `event` equals `booking_step_complete`.
2. Three Data Layer Variables: `DLV - step_number`, `DLV - step_name`, `DLV - clinic_location` (plus `specialty`, `preferred_date`, `booking_id` as needed per step).
3. One GA4 Event tag using this trigger, with the GA4 event name set dynamically — e.g. `booking_step_{{DLV - step_number}}` — so GA4 receives three distinct event names: `booking_step_1`, `booking_step_2`, `booking_step_3`.

**Surfacing drop-off in GA4 Funnel Exploration:**
- Build a Funnel Exploration with three steps: `booking_step_1`, `booking_step_2`, `booking_step_3`, in that order.
- Set it as an **open funnel** first (don't require the user's *first* action in session to be step 1) — this shows true drop-off without artificially inflating step 1 numbers by excluding people who scrolled around before starting.
- GA4 will show the % of users who completed step 1 but never reached step 2, and step 2 but never reached step 3 — this is your abandonment-by-step view, and it's what tells the marketing team whether the leak is at "give us your info" (step 2, most common leak in healthcare forms) or somewhere else.
- Break the funnel down by `clinic_location` as a secondary dimension to catch if one clinic's flow is disproportionately leaky (e.g. a broken date-picker on one location page).

---

## Conversion Action to Import into Google Ads

**Import: `booking_step_3` (booking_confirmed).**

Not `call_now_click`, not `whatsapp_click`, not `guide_lead_captured`. Reasoning:

- Google Ads Smart Bidding optimizes toward whatever conversion action you feed it. Feed it a soft/proxy signal (like a call button click, which doesn't confirm a call actually connected) and the algorithm will spend budget acquiring more clicks — not more patients.
- `booking_step_3` is the only event in this schema that represents a **confirmed appointment**, tied directly to the business outcome OrthoNow actually wants.
- Call clicks and WhatsApp clicks are useful as *secondary* conversions for reporting and audience-building, but importing them as the primary optimization target would mean Ads optimizes for top-of-funnel intent signals instead of completed bookings — exactly the kind of vanity-metric trap that got this account into a 2.1% conversion rate in the first place.
