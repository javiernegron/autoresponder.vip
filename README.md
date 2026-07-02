# [Autoresponder.vip](https://autoresponder.vip "Autoresponder.vip")

A lightweight PHP autoresponder designed for shared hosting, free hosting plans, and simple deployments without a database.

## Overview

Autoreponder.vip is the world's first portable autoresponder system, ultra-lightweight and easy to configure, that helps you manage subscription lists, send scheduled emails, global messages, and store subscriber data using only simple JSON files and PHP.

### Key features

- No database required
- Admin panel for managing lists, subscribers, and messages
- Sequential email delivery with delays between messages
- Send a global message to one, several, or all of your lists
- Public subscription forms with anti-bot protection
- Per-list public visibility controls for the public home page
- SMTP support
- Daily SMTP quota guard with persistent deferred queue for overflow sends
- CSV export and ZIP backups
- Daily logs for troubleshooting and monitoring
- Plain-text email by design — native `text/plain` delivery for maximum deliverability, universal compatibility, and zero HTML attack surface

## Project structure

- `autoresponder/admin.php` — administration panel
- `autoresponder/admin.js` — administration panel JavaScript (confirm dialogs, clipboard, form helpers)
- `autoresponder/index.php` — public signup page
- `autoresponder/subscribe.php` — subscription handler
- `autoresponder/unsubscribe.php` — unsubscribe handler
- `autoresponder/cron.php` — delivery runner for scheduled sends
- `autoresponder/logout.php` — legacy redirect shim; all logout logic lives in `admin.php?logout=1`
- `autoresponder/lib.php` — shared logic and helpers
- `autoresponder/style.css` — front-end stylesheet
- `autoresponder/config.example.php` — configuration template (safe to share)
- `autoresponder/config.php` — your local configuration file (create from the example; not included in git)
- `autoresponder/data/` — lists, subscribers, and runtime data
- `autoresponder/data/backups/` — backup archives
- `autoresponder/data/locks/queue.json` — persistent outbound mail queue and daily quota state
- `autoresponder/logs/` — daily log files
- `autoresponder/data/mail.log` — SMTP errors, send failures, bounces, and rate-limit events

## Requirements

- PHP 7.4+ (recommended PHP 8.x)
- Writable folders for `data/`, `logs/`, and `backups/`
- Optional (Recommended): SMTP Access.

## Installation

1. Upload the `autoresponder/` folder to your hosting account.
2. Make sure the project is accessible under `/autoresponder/`.
3. Create your configuration file:
   - Copy `autoresponder/config.example.php` to `autoresponder/config.php`.
   - Open `config.php` and replace every `CHANGE_ME` placeholder with your own values. Placeholders may appear in a descriptive form (e.g. `CHANGE_ME_SECRET_TOKEN`, `CHANGE_ME_PUBLIC_TRIGGER`) — replace them all:
     - `admin_password`
     - `unsubscribe_secret` — the security root for unsubscribe links and signup form tokens. Set it once, **before launch, to a long random string**, and never change it afterward: changing it invalidates every unsubscribe link already sent to your subscribers. Example: `a3f0c1b2e7d49...` (64 hex characters, 32 random bytes).
     - `public_trigger_key` — only needed if you use an external web cron (Option B in the Cron Setup tab). Set it to a long random alphanumeric string, e.g. `trig_9f2a7c4b1e8d...`. Keep it stable once your remote cron service is configured with it.
     - `from_name` and `from_email`
     - `base_url` — optional, but recommended when the script is installed in a subdirectory or automatic URL detection generates incorrect links. Example: `https://autoresponder.vip/autoresponder`
     - SMTP credentials (`smtp_username`, `smtp_password`) if `smtp_enabled` is `true`
   - Set `time_zone` if needed.

To generate a strong random value for `unsubscribe_secret` or `public_trigger_key`, run this once on your server or terminal and paste the output:
```bash
php -r "echo bin2hex(random_bytes(32)), PHP_EOL;"
```
4. Ensure the storage folders are writable.
5. Ensure `config.php` is writable by PHP (recommended). On the first successful admin login, the system automatically replaces your plain-text `admin_password` with a secure one-way hash in `config.php`.

The admin panel, cron runner, and public pages (index, subscribe, unsubscribe) refuse to run while required secrets are still set to `CHANGE_ME` (public pages display a clean, user-friendly maintenance page), ensuring you do not accidentally expose or use the script with the default placeholders.

The script is fully portable in any subdirectory. All asset references (stylesheet, favicon, icons, and web manifest) are resolved dynamically from the installation path, so no additional configuration is needed when deploying under a subdirectory such as `/tools/autoresponder/`.

If you use git, keep `config.php` out of the repository. The included `.gitignore` already excludes it.

## Configuration

The following settings are important for performance and delivery reliability:

- `max_sends_per_run` — maximum number of messages sent per cron run
- `send_delay_seconds` — pause between individual sends
- `cron_public_interval_minutes` — minimum time in minutes allowed between public web cron trigger runs (default: 1)
- `smtp_daily_limit` — maximum number of emails allowed per day by your SMTP provider; remaining messages are deferred to the next day in `data/locks/queue.json`
- `mail_queue_max_items` — maximum number of items allowed in the persistent outbound queue at any one time (default: 10000). Protects against unbounded disk and memory growth if `send_global` is triggered repeatedly before the cron drains the queue. Set to `0` to disable the cap.
- `bounce_threshold` — consecutive failures before a subscriber is marked as bounced
- `rate_limit_max` — max subscription attempts per IP
- `rate_limit_window` — time window for rate limiting
- `smtp_enabled` — enables SMTP transport when set to `true`
- `smtp_host` — SMTP host name
- `smtp_port` — SMTP port
- `smtp_username` — SMTP username
- `smtp_password` — SMTP password
- `smtp_secure` — `tls` or `ssl`
- `backup_dir`, `logs_dir`, `mail_log` — storage locations
- `admin_login_max_attempts` — failed admin logins before lockout (default: 5)
- `admin_login_lock_minutes` — lockout duration in minutes (default: 15)
- `base_url` — override the auto-detected base URL (useful when the script runs behind a reverse proxy or in a non-standard subdirectory)
- `default_thank_you` — default redirect URL after a successful subscription; leave blank to show the built-in thank-you page

## SMTP setup

To use your SMTP provider, configure the following values in `autoresponder/config.php`:

[Recommended SMTP Here](https://autoresponder.vip/blog/recommended-smtp)

- `smtp_enabled` => `true`
- `smtp_host` => `smtp.example.com`
- `smtp_port` => `587`
- `smtp_username` => your SMTP login
- `smtp_password` => your SMTP key/password
- `smtp_secure` => `tls`

This allows the system to send emails through your SMTP provider instead of relying only on the hosting server's default mail handler.

If your provider enforces a daily cap such as Brevo's 300 emails/day plan, set `smtp_daily_limit` to that value. The system will stop consuming SMTP capacity once the local limit is reached, store the remaining outbound messages in `data/locks/queue.json`, and resume them on the next day without marking those subscribers as bounced for quota exhaustion.

## Recommended settings by hosting type

[Recommended web hosting here](https://autoresponder.vip/blog/recommended-hosting/)

### Free or very limited hosting

- `max_sends_per_run`: 20
- `send_delay_seconds`: 2
- `bounce_threshold`: 2
- `rate_limit_max`: 3
- `rate_limit_window`: 3600

### Typical shared hosting

- `max_sends_per_run`: 40
- `send_delay_seconds`: 1
- `bounce_threshold`: 3
- `rate_limit_max`: 5
- `rate_limit_window`: 3600

### VPS or dedicated server

- `max_sends_per_run`: 100
- `send_delay_seconds`: 0
- `bounce_threshold`: 5
- `rate_limit_max`: 10
- `rate_limit_window`: 3600

## Usage

### Admin panel

Open `autoresponder/admin.php` and use the dashboard to:

- create and edit lists
- show or hide a specific list on the public home page
- view subscribers and export them as CSV (the admin panel does not support editing or deleting individual subscribers directly)
- add and schedule messages
- send global emails
- export subscribers as CSV
- create and restore ZIP backups
- review recent logs

### Public subscription form

The generated form can be pasted into any page. The form includes:

- a hidden honeypot field (`website`)
- a per-list token (`form_token`)
- IP-based rate limiting

The embed code generated by the admin panel includes a stylesheet loaded from an external URL (`https://javiernegron.github.io/autoresponder.vip/css/forms.css`). If you prefer not to depend on an external resource, copy the relevant CSS rules into your own stylesheet and remove the `<link>` tag from the embed code before pasting it into your site.

The `form_token` embedded in the generated code is an HMAC derived from your `unsubscribe_secret`. If you ever rotate that secret, regenerate the embed code from the admin panel — forms already published on external pages will reject new subscriptions until the updated code replaces them.

Each list can also be marked as visible or hidden on the public home page from its Settings screen in the admin panel. This only controls whether the list appears in the public list directory; it does not disable the list, its subscribers, or its email delivery.

### Sending messages

Emails are not sent in real time when someone subscribes. Instead, subscription is handled asynchronously: `subscribe.php` only **enqueues** the new subscriber (setting `next_send_at` to the current timestamp and `next_message_index` to `0`) and redirects immediately. The actual first message — and all subsequent ones — is delivered exclusively by the cron runner (`cron.php`) on its next execution. This design eliminates the email-bombing risk on the public subscription endpoint, reduces perceived latency for the subscriber, and respects your SMTP quota.

A delivery script (`cron.php`) must therefore be triggered periodically to process and send queued messages. You need to set this up once after installation.

There are three ways to trigger delivery, ordered from most to least recommended:

1. **Native CLI Cron Job (Recommended — no keys required)**: Schedule the script to run every minute directly on the server via command line. This is the most secure method: local execution is trusted implicitly and requires no secret keys or exposed URLs.
   ```bash
   php /absolute/path/to/autoresponder/cron.php
   ```
   Add this to your server's crontab with the schedule `* * * * *`. Use this method whenever your hosting supports SSH or cPanel cron jobs.

2. **Public Web Cron (Fallback — for hosting without CLI access)**: If your hosting does not support cron jobs or SSH, you can trigger delivery via an HTTP request using a secret key in the URL. Set up an external cron service (such as [cron-job.org](https://cron-job.org/en/)) to call this URL every minute:
   ```http
   https://yourdomain.com/autoresponder/cron.php?public_trigger=YOUR_PUBLIC_TRIGGER_KEY
   ```
   Replace `YOUR_PUBLIC_TRIGGER_KEY` with the value set in `config.php`. To prevent abuse, this endpoint includes an automatic anti-bot shield: it records the last execution timestamp in `data/cron_status.json` and silently aborts (returning `204 No Content`) if called more frequently than `cron_public_interval_minutes` (default: 1 minute). Successful executions also return `204 No Content` — no system information is exposed to the external cron service. Only configure `public_trigger_key` in `config.php` if you actually use this method.

3. **Admin Panel (Manual — for testing only)**: Run a single delivery cycle immediately by clicking the "Run Cron Manually" button in the admin panel under the Cron Setup tab. The page must remain open until the process finishes. Do not use this as your regular delivery method.

## Global send and dry-run

The admin panel includes a global send section that lets you:

- target one or more lists
- send to all lists
- preview the recipient count with `dry-run`
- use message templates such as:
  - `{{NAME}}`
  - `{{FIRSTNAME}}`
  - `{{EMAIL}}`
  - `{{LIST}}`
  - `{{UNSUBSCRIBE_URL}}`
  - `{{SITE_URL}}`

The global send flow is queue-aware: it enqueues recipients, processes them up to the configured daily SMTP limit, and leaves the remainder deferred until the next day if the quota is exhausted. This behavior is also visible in the `Backups & Logs` tab, which shows the queue state, the daily limit, how many messages were sent today, and how many remain pending.

## Security notes

Before publishing the project, make sure to:

- copy `config.example.php` to `config.php` and replace all `CHANGE_ME` values
- never commit `config.php` to a public repository
- change `admin_password`, `unsubscribe_secret`, and `public_trigger_key` to unique, long, random values
- **set `unsubscribe_secret` once, before launch, and never change it in production.** It is the HMAC key behind every unsubscribe link and public form token; changing it instantly breaks every link already sitting in your subscribers' inboxes
- only configure `public_trigger_key` if you actually use an external web cron (Option B). Keep it stable once your remote cron service is configured with it, or update that service to match if you rotate it
- use HTTPS when possible
- restrict access to the admin area
- **restrict thank-you redirects**: all custom thank-you URLs are validated to prevent Open Redirect attacks. They must be relative paths or absolute URLs belonging to the same host/domain (matching your active host name or the configured `base_url`). External redirect URLs are rejected.
- the script sets HTTP security headers automatically on every response (`Content-Security-Policy`, `X-Frame-Options: DENY`, `X-Content-Type-Options: nosniff`, `Referrer-Policy`). No server-level configuration is needed, but adding equivalent headers at the web server level provides an additional layer of protection.

### Admin password auto-hashing

You can set `admin_password` in plain text in `config.php` when you first install the script (for example `'my_secret_password'`). After the **first successful login**, the system detects that the value is not yet a PHP password hash and automatically rewrites `config.php`, replacing the plain-text password with a secure hash generated via `password_hash()`. Later logins use `password_verify()` against that hash.

For this one-time upgrade to work, `config.php` must be writable by the web server. If it is not writable, login still works, but the password remains in plain text until permissions are fixed and you log in again.

Failed admin login attempts are tracked per IP in `data/admin_login.json`. After `admin_login_max_attempts` failures (default: 5), login is blocked for `admin_login_lock_minutes` (default: 15 minutes).

## Backups and logs

- Backups are stored in `autoresponder/data/backups/`. Note: the `backups/` subfolder itself is excluded from ZIP archives to avoid recursive nesting. Individual ZIP entries larger than 50 MB are skipped during restore and logged as warnings — this protects against memory exhaustion on shared hosting from oversized or corrupted archives.
- The outbound email queue and daily quota state are stored in `autoresponder/data/locks/queue.json` and are displayed in the admin panel under `Backups & Logs`.
- Daily activity logs are stored in `autoresponder/logs/YYYY-MM-DD.log` — general activity: successful sends, cron runs, logins, subscriptions.
- Mail errors are recorded in `autoresponder/data/mail.log` — SMTP failures, send errors, bounce events, and rate-limit blocks.

## Troubleshooting

If emails are not arriving:

- check `autoresponder/data/mail.log`
- check the daily log file in `autoresponder/logs/`
- verify SMTP settings in `autoresponder/config.php`
- reduce `max_sends_per_run` if the hosting provider limits outbound mail

## FAQ

### Does the project require a database?
No. All data is stored in JSON files.

### Can I use SMTP instead of PHP mail()?
Yes. Set `smtp_enabled` to `true` and provide the SMTP details.

### What is `dry-run`?
It counts how many recipients would receive a message without actually sending the emails.

### What is a bounced subscriber?
A subscriber is marked as bounced after repeated send failures reach the configured threshold.

### How does the unsubscribe token work?
The unsubscribe token is a keyed HMAC-SHA256 hash derived from the subscriber's email address and the `unsubscribe_secret` value in `config.php`. It does not include the list slug, so the same token is valid across all lists that address is subscribed to. The list slug is still required in the unsubscribe URL to locate and update the correct record — it is not a security parameter.

Because `unsubscribe_secret` is the HMAC key, **rotating it invalidates every unsubscribe link you have already sent** (and every public form token, since they share the same key). Set it to a long random string before launch and leave it untouched for the lifetime of the installation. If you are ever forced to rotate it (suspected leak), expect old unsubscribe links and embedded form tokens to stop working until subscribers re-subscribe or you re-embed forms.

### Why does the system send plain-text emails only? Can I use HTML?
This is an intentional design decision, not a limitation. All messages are delivered with a native `Content-Type: text/plain` header. No HTML markup, `nl2br()` conversion, or `htmlspecialchars()` encoding is applied to the message body before dispatch — the text you write is exactly what reaches the recipient's inbox. This brings four concrete advantages:

- **Maximum deliverability.** Pure `text/plain` messages bypass aggressive spam filters organically and rarely land in Gmail's Promotions tab. They read like a one-to-one conversation, which is exactly what inbox algorithms reward.
- **Universal compatibility.** Plain text renders perfectly on every device — desktop, mobile, and smartwatch — with zero layout distortion and no dependency on an email client's HTML renderer.
- **Higher engagement.** Without visual clutter, readers focus entirely on the content. Full URLs such as `https://example.com` are auto-linked natively by every major email client (Gmail, Outlook, Apple Mail), so clickable links work without any HTML markup.
- **Zero HTML attack surface.** Because the transport is `text/plain`, there is no HTML execution context in the message body. HTML tags and scripts are treated as literal characters by the receiving client, eliminating any injection risk through message content.

## License

This project is intended for personal, educational, and small-business use. Adjust the configuration and security settings to match your deployment environment.
