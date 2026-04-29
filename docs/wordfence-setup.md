# Wordfence Premium — Setup Guide for `woods.consulting`

**Internal SOP — WBS Website Care**
**Audience:** Gaurav (developer)
**Requested by:** Zachary Woods, WOODS Business Solutions
**Stand:** April 2026

---

## Why we're doing this

WOODS Business Solutions is rolling out a new productized managed service called **WBS Website Care**. Zach is running the full **Care+** tier on `woods.consulting` first — as his own customer-zero test — before onboarding paying clients. This guide covers **just the Wordfence Premium piece**. Other components (uptime monitoring, the monthly maintenance script, multi-channel notifications) are wired up in separate work and don't block this task.

When you're done with this guide, `woods.consulting` should have:

- Wordfence Premium installed and licensed
- Real-time WAF (Web Application Firewall) in extended protection mode
- Daily automated security scans
- 2FA enforced for all admin-level users
- Brute-force / login-attempt protection
- Country blocking for the highest-risk regions
- Email alerts going to **zach@woods.consulting** for every notable event
- Documentation of credentials/configuration handed back to Zach

Total time estimate: **2–3 hours** if everything is straightforward.

---

## What you need before you start

| Item | Where to get it | Status |
|---|---|---|
| WP admin login on `woods.consulting` | Zach (1Password / Bitwarden / wherever he keeps it) | ⬜ confirm before starting |
| SFTP/SSH access to the hosting | Zach (Strato hosting credentials) | ⬜ confirm before starting |
| Strato hosting panel login | Zach | ⬜ confirm before starting |
| **Wordfence Premium license key** | Zach — buy at https://www.wordfence.com/products/wordfence-premium/ if not yet purchased (€99/year per site) | ⬜ confirm before starting |
| Recipient email for security alerts | `zach@woods.consulting` | ✅ |

> **STOP if WordPress isn't actually installed on `woods.consulting` yet** — that's a separate prerequisite. Flag it back to Zach and stop here.

---

## Step 1 — Take a backup before touching anything

Even though we're "only" installing a plugin, you don't install a security plugin without a rollback path. Three options, in order of preference:

1. **If Strato has a backup feature** in their hosting panel: trigger a manual snapshot, note the timestamp, confirm in their UI it completed.
2. **If you have SSH access**: tar the entire WP install + dump the DB:
   ```bash
   ssh <user>@<strato-host>
   cd ~                                    # or wherever the WP root sits
   tar -czf wp-backup-pre-wordfence-$(date +%Y%m%d).tar.gz public_html/
   mysqldump -u <db_user> -p <db_name> > db-backup-pre-wordfence-$(date +%Y%m%d).sql
   ```
3. **If only FTP**: download a full copy of the WP root directory locally + export the DB via phpMyAdmin in the Strato panel.

Confirm the backup file is not zero-byte before continuing. **Do not skip this step** — Zach's site is the production face of his business right now.

---

## Step 2 — Install the Wordfence plugin

**Method A — via WP admin UI (preferred for visibility):**

1. Log into `https://woods.consulting/wp-admin`
2. **Plugins → Add New**
3. Search for **"Wordfence"** (the publisher is "Wordfence" / Defiant Inc.)
4. Click **Install Now**, then **Activate**
5. You'll be redirected to a Wordfence onboarding screen — *don't* run through it yet, we'll do that explicitly in Step 4

**Method B — via WP-CLI** (if you have SSH):

```bash
cd /path/to/wordpress/root
wp plugin install wordfence --activate
```

After install, verify in WP admin: **Plugins → Installed Plugins** shows "Wordfence Security – Firewall, Malware Scan, and Login Security" as active.

---

## Step 3 — Activate Wordfence Premium

The free plugin is fine but the Care+ tier is sold to customers as Wordfence **Premium**, so we run Premium from day one.

1. **Wordfence → Dashboard** in the WP admin sidebar
2. Click **Upgrade to Premium** *(or "License" if you've already linked an account)*
3. When prompted, paste the **license key** Zach gave you
4. Confirm the dashboard now shows: `License: Premium` and the expiry date is ~12 months out

If license activation fails:
- Double-check there are no hidden whitespace characters around the key
- Make sure the license isn't already bound to another site (Wordfence licenses are per-site; if it's been used elsewhere, ask Zach to detach it from his Wordfence account at wordfence.com/console)

---

## Step 4 — Run through the initial Wordfence setup

Wordfence's onboarding wizard handles a chunk of the setup for us. Click through it carefully:

| Prompt | What to choose |
|---|---|
| Email for alerts | `zach@woods.consulting` |
| Wordfence promotional emails? | **No / Decline** |
| Auto-update Wordfence? | **Yes** — patches matter for a security plugin |
| Privacy Policy / Terms | Accept |

When the wizard finishes, you'll land on the Wordfence Dashboard. Don't celebrate yet — we still have manual config to do.

---

## Step 5 — Configure the Firewall (WAF) for extended protection

The default Wordfence install runs the WAF in "Basic mode," which means it only sees traffic *after* WordPress has loaded. We want **Extended Protection mode** — it puts the WAF in front of WordPress entirely (loads as a PHP `auto_prepend_file`).

1. **Wordfence → Firewall → All Firewall Options**
2. Under **Web Application Firewall Status**, click **Optimize the Wordfence Firewall**
3. The wizard auto-detects the server type (Apache/Nginx) — confirm the path it suggests, click **Continue**
4. **Important:** When it asks "Have you backed up your `.htaccess` and `.user.ini` files?" — say yes only after confirming the backup we made in Step 1 covers them. Wordfence will also offer to download backups inline; do that as a second safety net.
5. Wait for the verification step to complete. The dashboard should now show **"Extended Protection"** instead of **"Basic"**.

After it's set up:

- **Firewall Status:** Set to **Enabled and Protecting**
- **Block fake Google crawlers:** **Enabled**
- **Real-Time IP Blocklist** *(Premium feature):* **Enabled**
- **Brute Force Protection:** **Enabled** (defaults are OK)
- **Rate Limiting:** **Enabled** (defaults are OK)

---

## Step 6 — Login Security (2FA + brute force)

1. **Wordfence → Login Security**
2. Click the **Two-Factor Authentication** tab
3. **Enable 2FA for the admin account first** — scan the QR code with Zach's authenticator app (Authy, 1Password, Google Authenticator, Microsoft Authenticator — Zach picks). **Save the recovery codes somewhere safe and send them to Zach.**
4. Under **Settings**, set:
   - **Require 2FA for these roles:** Administrator, Editor (everyone with privilege to do damage)
   - **Allow remembered devices:** 30 days (good UX/security balance)
   - **Disable XML-RPC authentication:** **Yes** (legacy attack surface; if anything legitimately needs XML-RPC, Zach can re-enable later)

Then back to **Wordfence → Firewall → Brute Force Protection**:

- Lock out after **10** login failures
- Lock out after **5** forgot-password attempts
- Count failures over **5 minutes**
- Lock out for **30 minutes**
- Immediately lock out invalid usernames: **Yes**
- Immediately block the IP of users who try to sign in as: `admin`, `administrator`, `root`, `test`, `user` (add these to the username block list)

---

## Step 7 — Country blocking (Premium only)

This is one of the differentiators we sell on the Care+ tier landing page. Configure it:

1. **Wordfence → Blocking → Country Blocking**
2. **Block access to login form:** Add countries that have *zero* legitimate reason to log into a Würzburg-based business website. Conservative starting list:
   - China (CN)
   - Russia (RU)
   - North Korea (KP)
   - Iran (IR)
   - Belarus (BY)
3. **Block access to entire site:** Leave at default (none) for now — too aggressive for a marketing site that wants discoverability.
4. **Bypass URL** *(important):* Leave blank. Adding bypass URLs weakens the protection.
5. Save.

If Zach later needs to log in from a blocked country (travel), he can either disable country blocking temporarily from a different IP, or use a VPN to a permitted country. Document this in the handoff so he isn't surprised.

---

## Step 8 — Schedule automatic scans

1. **Wordfence → Scan → Scan Options and Scheduling**
2. **Scan type:** **High Sensitivity** (the Care+ tier promises this; the free tier defaults to "Standard")
3. **Scheduled scans:** Premium feature — set to run **daily, between 03:00 and 05:00 server local time**. Avoid daytime scans; they spike CPU and can affect site responsiveness.
4. Make sure these scan toggles are all on:
   - Scan core, theme, plugin files for malware
   - Scan registered files for changes
   - Scan posts and comments for malicious content
   - Scan files outside WordPress' installation
   - Scan for the HeartBleed vulnerability
   - Scan for publicly accessible config files, backup files, etc.
   - Check the strength of passwords
   - Scan for out-of-date, abandoned, and vulnerable plugins
5. Save and run a **first manual scan now** — review the results before walking away. Address any High/Critical findings before considering setup complete.

---

## Step 9 — Configure email alerts

1. **Wordfence → All Options → Email Alert Preferences**
2. **Where to email alerts to:** `zach@woods.consulting`
3. **Enable** these alerts (everything actionable):
   - Alert when an administrator signs in *(catches account compromises)*
   - Alert on critical problems
   - Alert on warnings
   - Alert when an IP address is blocked
   - Alert when someone is locked out from login
   - Alert when the lost password form is used for a valid user
   - Alert when someone with admin access signs in
   - Alert on increased attack rate
4. **Disable** these (too noisy):
   - Alert when "non-admin users sign in" — daily noise from any logged-in customer/user
   - Wordfence promotional emails
5. **Maximum email alerts per hour:** Set to `30` (default is unlimited; cap it so a brute-force flood doesn't fill Zach's inbox).
6. Save.

> **Note for Gaurav:** Zach is also building a Power Automate flow that will eventually take over multi-channel notifications (Email + WhatsApp + SMS). For now Wordfence emails to `zach@woods.consulting` are the source of truth. Don't try to wire Power Automate now — that's a separate work item.

---

## Step 10 — Document and hand back to Zach

When everything above is green, send Zach a short handoff email at `zach@woods.consulting` with these details:

```
Subject: Wordfence Premium installation on woods.consulting — completed

Hi Zach,

Wordfence Premium is installed and configured on woods.consulting per the
WBS Website Care setup guide. Summary:

- Plugin version: <e.g. 8.0.x>
- License: Premium, expires <date from dashboard>
- WAF mode: Extended Protection
- 2FA: enabled and required for Admin and Editor roles
- Daily scans: scheduled for <e.g. 04:00 server time>
- Country blocking: login blocked for CN, RU, KP, IR, BY
- Alerts: routed to zach@woods.consulting

Backups taken before any changes:
- Strato snapshot: <timestamp / location>
- File archive:    <path/to/wp-backup-pre-wordfence-YYYYMMDD.tar.gz>
- DB dump:         <path/to/db-backup-pre-wordfence-YYYYMMDD.sql>

2FA recovery codes for the admin account: <attach as separate password-
protected file or share via 1Password / Bitwarden — DO NOT paste in email>

First scan results: <clean / N findings, all addressed>

Outstanding follow-ups: <none / list anything>

Thanks,
Gaurav
```

---

## Verification Checklist

Before you call it done, walk through this list. All items must be ✅:

- ⬜ Wordfence Premium plugin active, license shows "Premium"
- ⬜ WAF status: **Enabled and Protecting**, mode: **Extended Protection**
- ⬜ Real-Time IP Blocklist: **Enabled**
- ⬜ Brute Force Protection: **Enabled**, sensible thresholds
- ⬜ Country blocking active for at least the 5 countries listed in Step 7
- ⬜ 2FA enabled and required for Administrator + Editor roles
- ⬜ XML-RPC authentication: **Disabled**
- ⬜ Daily scans scheduled in the 03:00–05:00 window
- ⬜ First manual scan completed; no unaddressed High/Critical findings
- ⬜ All notification emails arrive at `zach@woods.consulting` (test by triggering a deliberate failed-login lockout from your phone — you should get an alert email within ~30 seconds)
- ⬜ Pre-install backup taken AND verified non-zero-byte
- ⬜ Handoff email sent to Zach with all the items above

---

## Common gotchas

- **Wordfence "Optimize Firewall" wizard fails** — usually a permission issue on `wp-content/wflogs/` or the inability to write to `.htaccess`. Check ownership/perms; on Strato it's often `nobody:web` or similar.
- **Cache plugins clash with Wordfence** — if there's a page-cache plugin (W3 Total Cache, WP Rocket, etc.), make sure it's configured to bypass cache for logged-in users. Otherwise admins get stale Wordfence dashboards.
- **2FA locks YOU out during setup** — always confirm the QR scan worked by testing a logout/login cycle in a private/incognito window *before* logging out of your main session. Have the recovery codes ready.
- **Email alerts not arriving** — this is almost always a WP `wp_mail()` issue rather than Wordfence's fault. If you don't see a test email after the failed-login test, check the host's mail logs and consider installing **WP Mail SMTP** plugin pointed at a transactional email service (or Strato's SMTP relay).
- **Strato + Wordfence specifics** — Strato's shared hosting plans cap PHP execution time around 60s; full scans on a large site can timeout. If that happens, switch the scan from "High Sensitivity" to "Standard" and run scans more frequently as a workaround. Premium hosts (V-Server / dedicated) don't have this issue.

---

## References

- Wordfence official docs: https://www.wordfence.com/help/
- Initial setup walkthrough: https://www.wordfence.com/help/dashboard/options/
- Country blocking how-to: https://www.wordfence.com/help/firewall/country-blocking/
- 2FA guide: https://www.wordfence.com/help/tools/two-factor-authentication/
- Strato WordPress documentation: https://www.strato.de/faq/hosting/wordpress/

---

If anything in this guide is ambiguous or breaks against the actual hosting environment, **stop and message Zach before improvising**. He'd rather get a clarifying question than discover a creative configuration choice three months later.
