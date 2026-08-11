# Runbook: Passkey-Free Login for Test Automation (JWT Bearer + frontdoor.jsp)

**Audience:** Salesforce admin setting this up, plus the QA/automation team who consume it.
**Goal:** Let Selenium land in an authenticated browser session without ever touching the passkey/WebAuthn login prompt.
**Why this works:** The OAuth 2.0 JWT Bearer flow is server-to-server. It never invokes the interactive login screen, so no MFA, no passkey, no email OTP is triggered. The access token it returns doubles as a session ID, which `frontdoor.jsp` accepts to drop the browser into a logged-in state.

> Do **not** use the Username-Password OAuth flow instead — it's being blocked in more orgs and can still hit MFA. JWT Bearer is the durable choice.

---

## Part A — One-time admin setup

### Step 1 — Create a dedicated automation user

Create a normal user that exists only for automation. Don't reuse a person's account.

- Give it a recognisable username, e.g. `svc-testauto@yourdomain.com.<sandbox>`
- Assign the **minimum** profile/permission sets the tests actually need. Treat it as a service account.
- Keep it active. (It does not need a passkey registered — JWT bypasses that path entirely.)

### Step 2 — Generate a certificate + private key

The Connected App verifies the JWT signature with the public cert; the automation team signs with the private key.

```bash
openssl req -x509 -sha256 -nodes -days 365 -newkey rsa:2048 \
  -keyout jwt_testauto.key \
  -out   jwt_testauto.crt \
  -subj  "/CN=SF Test Automation JWT"
```

- `jwt_testauto.crt` → uploaded to Salesforce in Step 3.
- `jwt_testauto.key` → handed to the automation team as a **secret** (see Part C).

### Step 3 — Create the Connected App (JWT-enabled)

**Setup → App Manager → New Connected App** (classic Connected App is fine; External Client Apps work too and are the newer equivalent).

- **Connected App Name / Contact Email:** anything sensible.
- **Enable OAuth Settings:** ✅
- **Callback URL:** `http://localhost:1717/OauthRedirect` — a value is required but JWT never uses it, so a dummy is fine.
- **Use digital signatures:** ✅ → upload `jwt_testauto.crt`.
- **Selected OAuth Scopes:** add scopes that grant **API access**, **web/full access** (this is what makes the token valid for `frontdoor.jsp` UI sessions), and **refresh/offline access**. Exact label wording drifts between releases — pick the ones equivalent to:
  - *Manage user data via APIs* (`api`)
  - *Access the Salesforce API Platform* / *Full access* (`web` or `full`) ← needed for the UI session
  - *Perform requests at any time* (`refresh_token`, `offline_access`)

Save, then copy the **Consumer Key** (a.k.a. Client ID) — the team needs it as the JWT `iss`.

> Verify against your own org: the exact scope labels and whether your My Domain login policy requires the audience to be your My Domain URL rather than the generic login host (Step 6).

### Step 4 — Pre-authorize the app (no interactive consent)

**App Manager → your app → Manage → Edit Policies:**

- **Permitted Users:** `Admin approved users are pre-authorized`
- **IP Relaxation:** `Relax IP restrictions` (stops the token exchange being blocked by IP rules)

Then grant the automation user access to the app:

1. Create a Permission Set, e.g. **JWT Test Automation Access**.
2. In the permission set → **Assigned Connected Apps** → add your Connected App.
3. Assign the permission set to the automation user from Step 1.

Without this pre-authorization + assignment, the JWT exchange returns `user hasn't approved this consumer`.

### Step 5 — (Optional) loosen session friction for that user

If your org enforces **High Assurance** sessions on certain areas, a `frontdoor.jsp` session may still get prompted in those areas. For general portal/console test flows this usually isn't an issue. Check **Setup → Session Settings** and the user's login-flow/session-level policies if you see unexpected step-up prompts.

### Step 6 — Record the audience (login host) for the team

The JWT `aud` and the token endpoint host must match the environment:

| Environment | Audience / token host |
|---|---|
| Production / Dev org | `https://login.salesforce.com` |
| Sandbox | `https://test.salesforce.com` |
| My Domain login policy enforced | your My Domain URL, e.g. `https://yourco--uat.sandbox.my.salesforce.com` |

If My Domain "Prevent login from `https://login.salesforce.com`" is on, use the My Domain URL as both the audience and the token host.

---

## Part B — What the automation team runs (per test session)

They need three things from the admin: **Consumer Key**, **automation username**, **private key file**, plus the **audience** from Step 6.

### Step 7 — Build, sign, and exchange the JWT

Python example (`PyJWT` + `requests`):

```python
import jwt, time, requests

CONSUMER_KEY = "3MVG9..."                 # from Step 3
USERNAME     = "svc-testauto@yourdomain.com.uat"
AUDIENCE     = "https://test.salesforce.com"   # from Step 6
PRIVATE_KEY  = open("jwt_testauto.key").read()

claims = {
    "iss": CONSUMER_KEY,
    "sub": USERNAME,
    "aud": AUDIENCE,
    "exp": int(time.time()) + 180,        # <= 5 min in the future
}

assertion = jwt.encode(claims, PRIVATE_KEY, algorithm="RS256")

resp = requests.post(
    f"{AUDIENCE}/services/oauth2/token",
    data={
        "grant_type": "urn:ietf:params:oauth:grant-type:jwt-bearer",
        "assertion": assertion,
    },
)
resp.raise_for_status()
data = resp.json()
access_token = data["access_token"]
instance_url = data["instance_url"]       # e.g. https://yourco--uat.sandbox.my.salesforce.com
```

### Step 8 — Inject the session into Selenium

```python
driver.get(f"{instance_url}/secur/frontdoor.jsp?sid={access_token}")
# Driver is now authenticated. Navigate to the app/portal under test.
driver.get(f"{instance_url}/lightning/page/home")
```

That's the whole login step. No passkey, no OTP, no UI credentials. Wrap Steps 7–8 in the test suite's `setUp`/fixture so every run gets a fresh session.

---

## Part C — Security & handoff notes

- **The private key is a credential.** Anyone with it can mint sessions as the automation user. Store it in the CI secret manager (e.g. pipeline secret/vault), never in the repo. Rotate on the cert's expiry (365 days here) or if leaked.
- **Least privilege.** The automation user should only have access to what the tests exercise. It effectively bypasses MFA by design, so a narrow footprint matters.
- **Access tokens are bearer tokens.** Don't log the `sid`/`access_token`; it grants a live UI session for its lifetime.
- **Environment hygiene.** Keep separate certs/apps per environment if you want to revoke one without affecting others. Point `aud` and token host at the right environment (Step 6) — a prod token won't inject into a sandbox and vice versa.
- **Reserve the UI login test separately.** This runbook deliberately skips the login screen. If you also need to assert on the passkey login experience itself, do that in a small dedicated suite using Selenium's WebAuthn virtual authenticator (Chrome/Edge, CDP `WebAuthn` domain) — keep it out of the main functional suite.

---

## Quick failure reference

| Symptom | Likely cause |
|---|---|
| `invalid_grant: user hasn't approved this consumer` | Step 4 pre-authorization / permission-set assignment missing |
| `invalid_grant: invalid assertion` / signature error | Wrong private key, or `aud` doesn't match the token host |
| `invalid_grant: audience` | `aud` should be the My Domain URL (Step 6) |
| Token works for API but `frontdoor.jsp` bounces to login | Connected App missing the web/full scope (Step 3) |
| Unexpected step-up prompt after frontdoor | High Assurance session policy on that area (Step 5) |
