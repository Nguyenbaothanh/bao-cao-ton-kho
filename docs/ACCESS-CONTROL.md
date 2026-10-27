# Access control – Báo cáo tồn kho

Handover guide: how the site is locked to invited `@nbc.com.vn` users, and how to add or remove people.
Everything below can be copy-pasted. No secrets or invitation links are stored in this document.

## 1. Overview

The site is a single static page (`index.html`) hosted on Azure Static Web Apps (SWA). Access is enforced
by SWA itself, **before** the file is sent, so someone who is not allowed never receives any content.

```
Browser ──▶ SWA edge ──▶ has role "reader"? ──yes──▶ index.html
                │                  │
                │                  no, not signed in (401)
                │                  ▼
                │            redirect to Microsoft login (/.auth/login/aad)
                │                  │
                │                  ▼
                │            signed in, but not invited ──▶ 403 Forbidden
                ▼
        rules live in staticwebapp.config.json
```

Key points:
- Rule: every path requires the custom role `reader` (only `/robots.txt` is public).
- A person gets `reader` only if an admin **invited their email** (see section 6).
- Sign-in uses Microsoft Entra ID (`aad`). GitHub and Twitter logins are disabled (404).

### Free-plan limitation (important)
The Free plan cannot use our own Entra app registration, so sign-in **cannot be restricted to the
nbc.com.vn tenant**. The built-in `aad` provider accepts work/school accounts from any tenant. The
restriction therefore comes from the invitation list: signing in successfully is not enough, the account must
also have the `reader` role. Because `@nbc.com.vn` addresses can only be issued by our verified domain, the
residual risk is low. For a hard tenant lock, upgrade to Standard (section 10).

## 2. Resource reference

| Item | Value |
|---|---|
| Static Web App name | `nbc-bao-cao-ton-kho` |
| Resource group | `nbc-dev-bi-rg-sea` |
| Subscription ID | `ce66f59f-1aa3-482c-9f1a-3862fd26a0ff` |
| Default hostname | `blue-field-0fce4c400.6.azurestaticapps.net` |
| Plan | Free |
| Source repo | Azure DevOps `nhabecorp / NBC-App / nbc-bao-cao-ton-kho` |
| Deploy trigger | push to `main` (pipeline file: `azure-static-web-apps-blue-field-0fce4c400.yml`) |
| App location / output | `/` and `/` (the repo root is deployed as-is) |
| Access rules file | `staticwebapp.config.json` (repo root) |

## 3. Prerequisites

- Azure CLI installed (`az --version`).
- Signed in: `az login` (use your nbc.com.vn admin account).
- Permission: Contributor (or higher) on the Static Web App.
- **Gotcha:** your default subscription may belong to a different tenant. Always pass `--subscription`
  explicitly, or set it once for your shell:

  ```bash
  export SUB=ce66f59f-1aa3-482c-9f1a-3862fd26a0ff
  export RG=nbc-dev-bi-rg-sea
  export APP=nbc-bao-cao-ton-kho
  export HOST=blue-field-0fce4c400.6.azurestaticapps.net
  ```

  The commands below use these variables.

## 4. How `staticwebapp.config.json` works

```json
{
  "routes": [
    { "route": "/.auth/login/github",  "statusCode": 404 },   // disable GitHub login
    { "route": "/.auth/login/twitter", "statusCode": 404 },   // disable Twitter login
    { "route": "/robots.txt", "allowedRoles": ["anonymous"] },// crawlers can read it (it says Disallow: /)
    { "route": "/*", "allowedRoles": ["reader"] }             // everything else needs role "reader"
  ],
  "responseOverrides": {
    "401": {                                                  // not signed in
      "statusCode": 302,
      "redirect": "/.auth/login/aad?post_login_redirect_uri=.referrer"
    }
  }
}
```

(The comments above are explanations only; the real file is plain JSON without comments.)

- Routes are evaluated top to bottom; the first match wins, so keep `/*` last.
- 401 = not signed in, so the user is sent to Microsoft login and returned to the page afterwards.
- 403 = signed in but no `reader` role. SWA shows its Forbidden page. The user can switch account at
  `https://<HOST>/.auth/logout`.

## 5. One-time setup log

| Date | What | Who |
|---|---|---|
| 2026-10-27 | Added `staticwebapp.config.json` and this guide, deployed via push to `main` | _fill in_ |
| _date_ | First invitations sent (see table in section 6) | _fill in_ |

Steps that were performed, in order:
1. Created `staticwebapp.config.json` in the repo root.
2. Committed and pushed to `main`. The pipeline redeployed the site (check the run in Azure DevOps).
3. Invited each allowed person with role `reader` (section 6).
4. Verified access (section 9).

## 6. Runbook: add a user

1. Create an invitation (valid for at most 7 days = 168 hours):

   ```bash
   az staticwebapp users invite \
     --name "$APP" --resource-group "$RG" --subscription "$SUB" \
     --authentication-provider AAD \
     --domain "$HOST" \
     --user-details "first.last@nbc.com.vn" \
     --roles reader \
     --invitation-expiration-in-hours 168
   ```

2. The command prints an `invitationUrl`. **It does not email the person**, so send the link to them
   yourself (chat or email). Treat the link as sensitive; do not paste it into tickets or this repo.
3. The person opens the link and signs in with the **same nbc.com.vn account** that was invited.
   From then on they can open `https://<HOST>/` directly.
4. Record the person below (email only, no links).

Portal alternative: Azure portal → Static Web Apps → `nbc-bao-cao-ton-kho` → **Role management** →
**Invite** → provider *Azure Active Directory*, enter email, role `reader`, set expiry → copy the link.

### Current invited users
| Email | Invited on | Invited by |
|---|---|---|
| _fill in_ | | |

## 7. Runbook: remove a user / audit

List everyone who has a role (includes users who have used their invitation):

```bash
az staticwebapp users list --name "$APP" --resource-group "$RG" --subscription "$SUB" -o table
```

Remove access by clearing the role. Use the `userId` (or details) shown by the list command:

```bash
az staticwebapp users update \
  --name "$APP" --resource-group "$RG" --subscription "$SUB" \
  --authentication-provider AAD --user-id "<userId from list>" --roles ""
```

Portal alternative: **Role management** → select the user → **Delete**.
Removal takes effect on the user's next request or sign-in; ask them to sign out if it must be immediate.

## 8. Runbook: re-invite after expiry

If the link expired (7 days) or was lost, simply run the invite command from section 6 again for the same
email and send the new link. Old links stop working once they expire.

## 9. Verify access works

Run after any change to the config or the user list.

1. **Anonymous request is blocked**

   ```bash
   curl -sI "https://$HOST/"            | head -n 5   # expect 302 to /.auth/login/aad
   curl -sI "https://$HOST/index.html"  | head -n 5   # expect 302, never 200
   curl -sI "https://$HOST/robots.txt"  | head -n 3   # expect 200 (intentionally public)
   ```

2. **Invited user can open it**: in a private window open `https://<HOST>/`, sign in with the invited
   account, the report page loads. Then open `https://<HOST>/.auth/me`: `userRoles` contains `reader`.
3. **Non-invited account is blocked**: sign in with any other Microsoft account, expect 403 Forbidden and
   no report content. Use `https://<HOST>/.auth/logout` between tests.

## 10. Limits and upgrade path

- Free plan: custom roles by invitation are supported, with a cap on role-assigned users (about 25; check
  the Role management page for the current number).
- No tenant lock on Free (see section 1).
- Upgrade to Standard to restrict sign-in to the nbc.com.vn tenant:
  1. Create an Entra app registration (single tenant) and add its redirect URI
     `https://<HOST>/.auth/login/aad/callback`.
  2. Add the app's client ID and secret as SWA application settings.
  3. Add an `auth.identityProviders.azureActiveDirectory` block (with the tenant-specific `openIdIssuer`) to
     `staticwebapp.config.json`, and change the 401 redirect to `/.auth/login/aad`.
  4. Optionally switch the route rule from role `reader` to `authenticated`, or keep `reader` for a
     named-people list.

## 11. Troubleshooting

| Symptom | Likely cause | Fix |
|---|---|---|
| 403 Forbidden after signing in | Account has no `reader` role, or user signed in with a different account than invited | Check `users list`; re-invite the exact email; sign out at `/.auth/logout` and retry |
| Invitation link says expired/invalid | Older than 7 days, or already used | Re-invite (section 8) |
| Login loop between page and Microsoft | Browser blocks cookies, or an old session | Use a private window; clear cookies for the host |
| Changes to the config have no effect | Pipeline did not run or failed, or file not at repo root | Check the latest Azure DevOps run; confirm `staticwebapp.config.json` is at the root and `app_location` is `/` |
| `az` says resource not found | Wrong subscription (default is another tenant) | Pass `--subscription "$SUB"` |
| Cannot add more users | Free-plan role cap reached | Remove unused users or upgrade to Standard |

## 12. Related exposure checklist

- [ ] GitHub Pages copy (the README mentions one): unpublish it, or make that repo private. SWA
      rules do not protect it.
- [ ] `SHEET_URL` in `index.html` is a Google Apps Script web-app URL. After this change only invited
      users can read it from the page, but anyone who already has the URL can still call it. Make sure
      the script requires a token or is restricted to the domain.
- [ ] The Git repository itself holds the full content; keep the Azure DevOps repo private.
