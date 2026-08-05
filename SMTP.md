# How to Configure SMTP for UISP / UCRM (Microsoft 365)

This guide walks through setting up outgoing email (system alerts + UCRM billing/invoice emails) for a self-hosted UISP instance using Microsoft 365 / Exchange Online as the SMTP relay.

## Why SMTP?

UISP and UCRM need to send email for things like device/network alerts, client account statements, and invoices. Microsoft 365 doesn't allow anonymous relay, so UISP has to authenticate as a real mailbox before it's allowed to send. There are two ways to authenticate:

- **Basic Auth (username + app password)** — simpler to set up, works today, but Microsoft is phasing this out (SMTP AUTH Basic Auth is scheduled to be disabled by default around the end of December 2026, per Microsoft's published timeline). This is what this guide covers.
- **OAuth2 / XOAUTH2** — more setup work (app registration in Entra ID), but not affected by the Basic Auth deprecation and doesn't depend on Conditional Access/MFA quirks. If your UISP version supports it and you'd rather set this up instead, skip to [Alternative: Using OAuth2](#alternative-using-oauth2-instead-of-app-password) at the bottom.

For most self-hosted setups today, Basic Auth + app password (below) is the quickest path and is what's documented step by step here.

---

## Prerequisites

- A dedicated mailbox to send from (don't use a personal mailbox) — e.g. `alerts@yourdomain.com`
- Global admin or Exchange admin access to your Microsoft 365 tenant
- SSH/CLI access to your UISP server
- Admin access to the UISP and UCRM web UI

---

## Step 1: Confirm SMTP AUTH is enabled (Microsoft 365 side)

Connect to Exchange Online PowerShell and check both the mailbox-level and tenant-level settings:

```powershell
Get-CASMailbox -Identity alerts@yourdomain.com | fl SmtpClientAuthenticationDisabled
Get-TransportConfig | fl SmtpClientAuthenticationDisabled
```

Both should return `False`. If either returns `True`, enable it:

```powershell
Set-CASMailbox -Identity alerts@yourdomain.com -SmtpClientAuthenticationDisabled $false
```

---

## Step 2: Exclude the mailer account from any Conditional Access policy blocking legacy auth

Exchange Online treats SMTP AUTH as "legacy authentication." If your tenant has a Conditional Access (CA) policy blocking legacy auth (common security baseline), you need to exclude this specific mailbox account from that policy:

1. Entra admin center → **Protection → Conditional Access**
2. Open the policy blocking legacy/basic authentication
3. Under **Assignments → Users**, add `alerts@yourdomain.com` to the **Exclude** list
4. Save. Allow up to an hour for this to propagate.

---

## Step 3: Confirm the account's MFA won't block app passwords

App passwords only work if the account's MFA is enforced through **Security Defaults** or **legacy per-user MFA** — not through Conditional Access alone. Check one of these is true for the account:

- Entra admin center → **Identity → Overview → Properties → Manage Security defaults** (is it On?), **or**
- Legacy per-user MFA page: `https://account.activedirectory.windowsazure.com/UserManagement/MultifactorVerification.aspx` — is the account listed as Enabled/Enforced?

If neither is true, app passwords generated for this account won't authenticate over SMTP even though CA is excluded — enable per user MFA for account before continuing.

---

## Step 4: Generate an app password

1. Sign in as the mailer account (or have the user do it) at [https://mysignins.microsoft.com/security-info](https://mysignins.microsoft.com/security-info)
2. Add an **App password**
3. Copy it immediately and save it somewhere secure — it's shown once. Type it out manually where you paste it in later rather than relying on clipboard copy, to avoid invisible characters/whitespace being carried over.

---

## Step 5: Test authentication independently of UISP

Before touching UISP's settings, confirm the credentials actually work using `swaks` (a proper SMTP test tool), from the UISP server itself:

```bash
sudo apt install swaks -y

swaks --to your-test-inbox@example.com \
  --from alerts@yourdomain.com \
  --server smtp.office365.com \
  --port 587 \
  --auth LOGIN \
  --auth-user alerts@yourdomain.com \
  --auth-password 'YOUR_APP_PASSWORD' \
  --tls
```

You're looking for this at the end of the output:
```
<~  235 2.7.0 Authentication successful
...
<~  250 2.0.0 OK ...
```

If you see that, the Microsoft 365 side is fully working and any remaining issue will be on the UISP/UCRM side. Don't move on to Step 6 until this succeeds — it saves a lot of time guessing later.

---

## Step 6: Configure UISP's core Mailer settings

In the UISP web UI, go to **Settings → Mailer** (system-level, not UCRM) and enter:

| Field | Value |
|---|---|
| SMTP Host | `smtp.office365.com` |
| Port | `587` |
| Encryption | STARTTLS / TLS |
| Username | `alerts@yourdomain.com` |
| Password | the app password from Step 4 (type manually, don't paste) |
| Sender / Organization email | `alerts@yourdomain.com` |

Save, then use the built-in **Test SMTP Server** option. You should see a confirmation that settings are working.
<img width="689" height="665" alt="Screenshot 2026-08-04 140309" src="https://github.com/user-attachments/assets/83ed0533-ad64-495b-9fa9-d005c58a8a1d" />

---
<img width="445" height="347" alt="Screenshot 2026-08-04 144616" src="https://github.com/user-attachments/assets/4c791033-e9e8-4530-af0d-d5473df87261" />

## Step 7: Configure UCRM's Organization email to match

This step matters even if Step 6 already succeeded — UCRM's billing emails use a separate "From" identity, and Microsoft 365 requires the authenticated account and the `From` address to match (otherwise you'll get a `530 5.7.57` error, see [Problems We Faced](#problems-we-faced) below).

1. **CRM → System → Organization** → set **Organization email** to `alerts@yourdomain.com` (same account used in Step 6)
2. **CRM → System → Mailer → Support email address** can stay as your normal support inbox (e.g. `help@yourdomain.com`) — this only controls Reply-To and the client contact form, it doesn't affect the sending identity
3. Confirm **Sandbox redirect address** is empty — if it has an address in it, *all* outgoing CRM email gets redirected there instead of the real recipient

---
<img width="1164" height="895" alt="Screenshot 2026-08-04 080713" src="https://github.com/user-attachments/assets/e512bbab-e2b8-493c-a729-91d622cd3c39" />

## Step 8: Send a real test invoice

1. Create a disposable test client in UCRM with an email address you can check
2. Assign it a service plan
3. Generate an invoice for the client and send it
4. Check the test inbox for delivery, and check **CRM → System → Email Log** to confirm it was sent (or see the exact error if not)
5. Delete the test client/invoice once confirmed

---
<img width="1055" height="2284" alt="Image (3)" src="https://github.com/user-attachments/assets/92ae4bd9-2196-4f27-97e1-742d82c2708a" />
<img width="1170" height="1636" alt="Image (4)" src="https://github.com/user-attachments/assets/820aa072-2069-435c-8f79-5ea6fafd49ab" />

## Step 9: Clean up

- Confirm MFA is (or remains) enabled on the mailer account — don't leave it disabled, even temporarily, since a mail-sending account without MFA is a soft target for compromise
- If you cleared UISP's app cache via CLI while troubleshooting, no further action needed — this is safe to do any time settings don't seem to be taking effect

---

## Problems We Faced

These were the specific errors hit while setting this up, in case you run into the same ones:

- **`SMTP server connection is not working properly. SMTP configuration was reverted back.`** — UISP's generic UI error when its internal test fails. Doesn't tell you why. Always follow Step 5 (`swaks`) to get the real error instead of relying on this message.

- **`535 5.7.3 Authentication unsuccessful`** — Seen after CA exclusion + app password were both in place, but before confirming Step 3. Root cause: MFA for the account was enforced purely through Conditional Access, so the app password wasn't actually honored. Fixed by confirming Security Defaults / per-user MFA (Step 3).

- **`535 5.7.139 Authentication unsuccessful, the user credentials were incorrect`** — Turned out to be a copy/paste mistake where a placeholder password string was sent instead of the real app password. Worth ruling this out first before assuming it's a policy issue — decode what's actually being sent if in doubt.

- **`Too Many Requests` from UISP's UI, even after `swaks` confirmed auth worked** — This was UISP's own internal state/backlog from earlier failed attempts, not a Microsoft-side throttle. Confirmed by successfully sending via `swaks` directly. Resolved by checking UISP's Throttler setting and clearing the UISP application cache via CLI using;

      sudo docker ps - This estart the UISP mailer service / UISP app, if there's an option to do so from Settings, or via CLI

- **`530 5.7.57 Client not authenticated to send mail`** — Only happened on UCRM invoice emails, not UISP system alerts. Root cause: the SMTP-authenticated account (`alerts@yourdomain.com`) didn't match UCRM's configured "From" address (was set to a different support address). Fixed by making them match, per Step 7. This is why Step 7 is a separate, required step even after Step 6 succeeds.

If you hit an error not listed here, run the `swaks` command from Step 5 again with your current credentials — reading the raw SMTP response code is almost always faster than guessing from the UISP UI.

---

## Alternative: Using OAuth2 instead of app password

If you'd rather not depend on Conditional Access exclusions, MFA type, and app passwords at all (and your UISP version supports OAuth2/XOAUTH2 for SMTP — check your version's release notes), you can register an app in Entra ID and use OAuth2 client credentials instead. This avoids all of the CA/MFA-related issues in Steps 2–3 entirely, since it doesn't rely on legacy authentication. Broadly this involves:

1. Registering an app in **Entra admin center → App registrations**
2. Granting it the `SMTP.Send` (or equivalent Mail.Send) application permission with admin consent
3. Configuring UISP with the app's client ID, tenant ID, and client secret instead of a username/password, if UISP's mailer settings expose an OAuth2 option

This is worth planning toward given Microsoft's Basic Auth deprecation timeline, but is a larger one-time setup effort than the app-password path above, so it's documented as an alternative rather than the default path in this guide.
