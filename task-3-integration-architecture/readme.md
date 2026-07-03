# Task 03 — Integration Design: Landing Page → HubSpot → WhatsApp → Google Ads

## Architecture: end-to-end

I'd use a **direct server-side API call from the landing page's form handler to HubSpot**, not a native HubSpot embed and not a no-code layer like Zapier or Make, for one core reason: the form fires a GTM dataLayer event *and* needs to trigger a WhatsApp send within a 2-minute SLA. Native HubSpot embeds hand control of the submission to HubSpot's own script, which makes it harder to guarantee the dataLayer push fires reliably and adds a dependency on HubSpot's uptime for the very first step of the funnel. Zapier/Make are convenient but add 10–30+ seconds of polling latency per hop, which eats directly into a 2-minute WhatsApp SLA when you're chaining two hops (HubSpot → WhatsApp).

**Flow:**
1. Form submits to our own lightweight backend endpoint (not directly to HubSpot from the browser — keeps the API key server-side).
2. Backend calls the **HubSpot Contacts API** (`PATCH /crm/v3/objects/contacts` with an `idProperty` search-and-upsert on phone) to create or update the contact with Name, Phone, Clinic Preference, Source, and Lead Status.
3. In parallel (not sequentially, to protect the SLA), the backend calls **Karix's WhatsApp Business API** directly to send the confirmation template message.
4. The backend also fires the **Google Ads Conversion API** (server-side, via Enhanced Conversions) rather than relying solely on the client-side GTM tag, so the conversion still gets recorded even if the user closes the tab before GTM's browser-side hit completes.

## Biggest failure point

**Phone number deduplication.** HubSpot's default dedup key is email, not phone — and this form never collects email. If two different patients submit with the same phone number (a shared family phone is common in Indian healthcare lead gen) but different names, without an explicit dedup rule HubSpot will either create two separate contacts (breaking a single patient view) or, if I set phone as a custom unique identifier, silently overwrite the first patient's name with the second one's.

**Fallback:** treat phone as the unique key, but on a name mismatch, don't overwrite — instead log it as a new "Enquiry" activity on the existing contact record and flag it for manual review, rather than blindly merging two different people into one record.

## Protecting the WhatsApp 2-minute SLA

What could break it: Karix API rate limits or downtime, HubSpot API latency blocking a sequential call, or WhatsApp template approval issues. I'd monitor this with a lightweight logging layer that timestamps each step (form submit → HubSpot write → WhatsApp send) and alerts if the gap exceeds 90 seconds, giving a buffer before the SLA is actually breached — plus a daily dashboard of P50/P95 send times.

