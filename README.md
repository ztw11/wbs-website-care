# WBS Website Care — Landing Page

Self-service signup landing page for the WBS Website Care managed WordPress service.

**Live URL:** https://care.woods.consulting/
**Repo:** https://github.com/ztw11/wbs-website-care

---

## Form submission setup (current — stopgap via Formsubmit.co)

The signup form is wired to **[Formsubmit.co](https://formsubmit.co)**, a free no-signup form-to-email service. Submissions email **zach@woods.consulting**.

### Configuration in `index.html`

```html
<form action="https://formsubmit.co/zach@woods.consulting" method="POST">
  <input type="hidden" name="_subject" value="Neue WBS Website Care Anfrage">
  <input type="hidden" name="_template" value="table">
  <input type="hidden" name="_captcha" value="true">
  <input type="hidden" name="_next" value="https://care.woods.consulting/?submitted=1">
  <input type="text" name="_honey" style="display:none" tabindex="-1" autocomplete="off">
  ...
</form>
```

| Hidden field | Purpose |
|---|---|
| `_subject` | Email subject line |
| `_template` | `table` = nicely-formatted HTML table of submitted fields |
| `_captcha` | reCAPTCHA enabled (recommended; bot protection) |
| `_next` | Redirect URL after submission. Triggers thank-you banner via `?submitted=1` query param |
| `_honey` | Honeypot field (hidden from humans, bots fill it = submission rejected) |

Hidden state-mirroring fields `paket` and `zahlungsart` capture the user's tier and payment method clicks (the visual cards aren't real radios).

### One-time activation (required on first deployment or recipient change)

1. Submit the form once with any test data
2. Formsubmit sends a confirmation email to `zach@woods.consulting`
3. Click the activation link in that email
4. From then on, all submissions email through automatically

If you change the action URL recipient (e.g., to a different email), repeat the activation.

### What gets emailed

A nicely-formatted table with these fields:

- `vorname`, `nachname`, `unternehmen`
- `email` (also used as Reply-To automatically)
- `telefon`, `strasse`, `plz`, `ort`
- `paket` (e.g., `Care+`)
- `benachrichtigungskanal` (multi-value: `E-Mail`, `WhatsApp`, etc.)
- `zahlungsart` (`SEPA-Lastschrift` or `Rechnung & Überweisung`)
- `kontoinhaber`, `iban`, `bic` (only if SEPA selected and filled)

### Why Formsubmit.co (not the eventual plan)

This is a **stopgap** until the Power Automate flow described in the project's
"Signup Ingest" milestone (Flow 1) is built. That flow will:

1. Receive the form payload via HTTP webhook trigger
2. Create `wbs_signupsubmission`, `wbs_client`, `wbs_contact`, `wbs_contract`,
   `wbs_managedwebsite`, `wbs_notificationpreference` records in the WBS Production
   Dataverse environment
3. Send confirmation email to customer + internal alert to Zach
4. Create a D365 follow-up task

When that flow exists, swap the `action` attribute on the `<form>` to point at
the Power Automate webhook URL, remove the Formsubmit `_*` hidden fields, and
the rest of the form stays the same.

### Privacy note

Formsubmit.co is US-based and form data transits via Cloudflare Workers. For a
limited-volume stopgap with the existing recipients this is acceptable. Once the
Power Automate flow is live (data stays inside the EU-region Microsoft tenant),
migrate before any volume.

---

## Hosting

GitHub Pages from the `main` branch root:

- Repo Pages config enabled with `cname: care.woods.consulting`
- TLS via Let's Encrypt (auto-renewed by GitHub)
- HTTP → HTTPS enforced
- DNS: CNAME `care.woods.consulting → ztw11.github.io.` at Strato

If the cert ever fails to renew or you change the custom domain, the nudge that
forces re-provisioning is:

```powershell
gh api --method PUT repos/ztw11/wbs-website-care/pages -f "cname="
gh api --method PUT repos/ztw11/wbs-website-care/pages -f "cname=care.woods.consulting"
```

---

## File map

| File | Purpose |
|---|---|
| `index.html` | The landing page (single file: HTML + CSS + JS inline) |
| `CNAME` | Custom domain config for GitHub Pages |
| `README.md` | This file |
