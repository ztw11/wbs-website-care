# Wordfence Premium auf woods.consulting

Hey Gaurav,

quick task — install and configure Wordfence Premium on woods.consulting. ~2-3 hours total, mostly waiting on the first scan to finish.

I'll DM you the **Premium license key** separately (1Password). Don't proceed without it.

## Backup before you touch anything

Strato snapshot via the hosting panel if that's available. Otherwise SSH in:

```bash
tar -czf wp-backup-pre-wordfence-$(date +%Y%m%d).tar.gz public_html/
mysqldump -u <db_user> -p <db_name> > db-backup-pre-wordfence-$(date +%Y%m%d).sql
```

Confirm both files are non-zero before continuing.

## Install + license

In WP admin → **Plugins → Add New → search "Wordfence" → Install + Activate**. Or `wp plugin install wordfence --activate` if you've got CLI.

Then **Wordfence → Dashboard → Upgrade to Premium** and paste the license key. Should show "Premium" with a 12-month expiry.

## Onboarding wizard

The two answers that matter:

- **Alert email:** `zach@woods.consulting`
- **Auto-update Wordfence:** Yes

Decline the marketing emails.

## Switch the WAF to Extended Protection

This is the step everyone skips and shouldn't.

**Wordfence → Firewall → All Firewall Options → Optimize the Wordfence Firewall**.

Walk the wizard, accept the suggested server config (it'll detect Apache/Nginx). Take the `.htaccess` / `.user.ini` backups it offers.

When done, the dashboard should show **Extended Protection**, not **Basic**.

While you're in firewall settings:

- Real-Time IP Blocklist → enabled
- Brute Force Protection → enabled (defaults are fine)
- Block fake Google crawlers → enabled

## Login security & 2FA

**Wordfence → Login Security → Two-Factor Authentication**.

Enable 2FA on the admin account. I'll use Microsoft Authenticator — same as the rest of my stack. **Save the recovery codes in 1Password and share them with me before you log out.** Test the QR code in a private/incognito window first so you don't lock yourself out mid-setup.

Under **Settings**:

- Require 2FA: Administrator + Editor
- Allow remembered devices: 30 days
- Disable XML-RPC authentication: Yes

Then **Firewall → Brute Force Protection**:

- Lock out after 10 failed logins / 5 forgot-password attempts
- Counted over 5 minutes
- 30-minute lockout
- Immediately block usernames: `admin`, `administrator`, `root`, `test`, `user`

## Country-block the login form

**Wordfence → Blocking → Country Blocking → Block access to login form**. Add: CN, RU, KP, IR, BY.

Don't block the entire site — just the login form. The site stays publicly accessible from those countries; only `/wp-admin` is blocked.

## Daily scans

**Wordfence → Scan → Scan Options**:

- High Sensitivity
- Daily, scheduled between 03:00 and 05:00 server time (avoid daytime — scans hammer CPU)
- Toggle on everything: malware, file changes, posts/comments, files outside WP, password strength, abandoned plugins

Run a manual scan now and review findings before signing off. Address anything High or Critical.

## Alert preferences

**Wordfence → All Options → Email Alert Preferences**:

- Recipient: `zach@woods.consulting`
- Cap at 30 emails/hour so a brute-force flood doesn't fill my inbox
- Enable: admin sign-ins, critical problems, warnings, IP blocks, lockouts, lost-password attempts, increased attack rate
- Disable: non-admin sign-ins (way too noisy)

## Smoke test

From your phone (different IP from the server), try to log into `woods.consulting/wp-admin` with a deliberately wrong password 11 times. Should:

- Lock you out
- Trigger an alert email at `zach@woods.consulting` within ~30 seconds

If both happen, you're done.

## When you're done

Quick recap message back to me with:

- Plugin version installed
- License expiry date
- Confirmation: WAF mode = Extended Protection
- Confirmation: smoke test triggered the alert email
- 2FA recovery codes → 1Password
- Backup file paths

Anything weird (Strato perms issues, WAF wizard fails, scan timeouts on shared hosting), ping me before improvising.

Cheers,
Zach
