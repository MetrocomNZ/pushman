# Update portal

An Auth0-gated admin console for updating upstream API endpoints. React front end,
FastAPI back end, one API key per endpoint held on the server.

## How access works

```
Browser ──login──▶ Auth0
Browser ──Bearer access token──▶ FastAPI
                                  │ verifies RS256 signature against Auth0 JWKS
                                  │ checks audience + issuer + expiry
                                  │ checks the endpoint's required permission
                                  └──X-API-Key / Bearer / ?api_key──▶ upstream endpoint
```

The browser never receives an API key. Auth0 decides *who* gets in and *what* they
may change; the API keys are read from server environment variables at call time
and stripped from everything returned to the client.

## Auth0 setup

**1. Create an API** (Applications → APIs → Create API)

| Setting | Value |
| --- | --- |
| Name | Update portal API |
| Identifier | `https://update-portal-api` (this is your `audience` — it needn't resolve) |
| Signing algorithm | RS256 |

On the API's **Settings** tab enable *Enable RBAC* and *Add Permissions in the Access Token*.

**2. Add permissions** (the API's Permissions tab). One per endpoint, plus the audit one:

```
update:3cx           Change extensions on the 3CX PBX
update:yeastar       Change extensions on the Yeastar PBX
update:grandstream   Change extensions on the Grandstream UCM
update:3cx-lab       Change the lab 3CX only (per-system override)
manage:systems       Add and remove phone systems through the portal
read:audit           See everyone's activity, not just your own
```

**3. Create roles** (User Management → Roles), e.g. `3CX operator` with
`update:3cx`, `Telephony admin` with everything. Assign roles to users.

**4. Create the SPA** (Applications → Create Application → Single Page Web App).
On its Settings tab:

| Field | Value |
| --- | --- |
| Allowed Callback URLs | `http://localhost:5173` |
| Allowed Logout URLs | `http://localhost:5173` |
| Allowed Web Origins | `http://localhost:5173` |

Under Advanced Settings → Grant Types, make sure *Refresh Token* is ticked, and
under the API's settings enable *Allow Offline Access*.

**5. Optional — expose role names.** Permissions are what the API enforces; roles
are only for display. If you want them, add an Auth0 Action (Login flow):

```js
exports.onExecutePostLogin = async (event, api) => {
  const ns = "https://portal.example.com/roles";
  api.accessToken.setCustomClaim(ns, event.authorization?.roles ?? []);
};
```

## Running it

**Back end**

```bash
cd backend
python -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
cp .env.example .env        # fill in AUTH0_DOMAIN, AUTH0_AUDIENCE, and the KEY_* vars
uvicorn app.main:app --reload --port 8000
```

**Front end**

```bash
cd frontend
npm install
cp .env.example .env        # fill in domain, client id, audience
npm run dev
```

Open http://localhost:5173.

## The three phone systems

None of them take a static API key. Each gets a credential provider that turns
long-lived secrets into short-lived sessions, caches them, and re-authenticates
on its own when they lapse. `credentials.py` holds all four strategies.

| PBX | Where to enable it | Credential | Session |
| --- | --- | --- | --- |
| 3CX v20 | Admin console → Advanced → API | Client ID + Secret (`THREECX_*`) | `POST /connect/token`, `grant_type=client_credentials`, token good for 1 hour |
| Yeastar P-Series | Web portal → Integrations → API | Client ID + Secret (`YEASTAR_*`) | `POST /openapi/v1.0/get_token`, access token 30 min, refresh token 24 h |
| Grandstream UCM63xx | Integrations → API Configuration → HTTPS API Settings (New) | Username + Password (`GRANDSTREAM_*`) | `challenge` → `MD5(challenge + password)` → `login`, cookie dies after 5 min idle |

Vendor quirks the portal handles for you:

- **Grandstream stores config changes without activating them.** Any operation
  marked `apply_changes: true` gets a follow-up `applyChanges` call once the
  first one succeeds, and the result reports both. Without it the change sits
  inert on the UCM.
- **Grandstream isn't REST.** Every call is `POST /api` with the action in a JSON
  body, and success is `status: 0` rather than an HTTP code. That's the
  `json_rpc` transport; the UI adapts, showing the action name instead of a verb.
- **Yeastar requires a `User-Agent` header** on every request or the PBX ignores
  it, and it uses `GET` for deletes. The `delete-extension` operation is marked
  `readonly: false` so the audit log still treats it as a write.
- **Yeastar's `id` is not the extension number.** Run List extensions first.
- **3CX ids are OData.** The path is `/Users({id})` and `id` is the numeric user
  Id, not the dialled extension.
- **Sessions expire mid-form.** On a `401`/`403` — or a UCM `-6`/`-8` — the
  provider is invalidated and the call retried once, so an admin doesn't lose a
  submission to a cookie that timed out while they were typing.

Point the three `base_url`s at your own PBXs before first run. If a UCM or
Yeastar box uses a self-signed certificate, `httpx.AsyncClient` in `proxy.py`
needs `verify=` pointed at your CA bundle — don't reach for `verify=False`.

## Manufacturers and their PBXs

The registry has two levels. A **manufacturer** defines how to talk to a
platform — transport, auth shape, operations, where the firmware version lives.
An **instance** is one PBX: a host, the names of its credential variables, and
optionally its own permission and firmware target.

```json
{
  "id": "grandstream",
  "name": "Grandstream",
  "transport": "json_rpc",
  "api_path": "/api",
  "required_permission": "update:grandstream",
  "auth": { "kind": "challenge_login", "login_path": "/api", "session_ttl": 240 },
  "operations": [ "... shared by every Grandstream ..." ],
  "instances": [
    {
      "id": "depot",
      "name": "Depot UCM",
      "site": "Hamilton",
      "model": "UCM6304",
      "host": "https://ucm-ham.example.com:8089",
      "credentials": {
        "user": "GRANDSTREAM_DEPOT_USER",
        "password": "GRANDSTREAM_DEPOT_PASSWORD"
      }
    }
  ]
}
```

Adding the ninth Grandstream at a new site is that instance block plus two
secret files. Operations, auth and firmware probing are inherited.

Each instance gets its own credential provider, its own cached session and its
own circuit breaker, keyed `manufacturer/instance`. A wrong password on the
depot UCM trips the breaker for that box alone — head office keeps working.

**Per-PBX permissions.** An instance may override `required_permission`. The lab
3CX in the sample registry uses `update:3cx-lab`, so you can let someone
practise on staging without granting them head office. Rate limits are keyed per
PBX too, so a busy system does not throttle the branches.

## Adding a PBX from the portal

Each manufacturer group has an **Add PBX** button, gated on `manage:systems`.
The form is generated from the manufacturer's credential roles, so 3CX asks for
a client id and secret while Grandstream asks for an API username and password,
with no per-vendor code in the frontend.

This is the one place the security model had to bend. Everything else in the
portal is read-only with respect to secrets: credentials arrive as files an
admin placed and the app only ever reads them. A button that adds a PBX means a
credential travels from a browser into the server and has to be kept somewhere
the app can write. So:

- **Credentials are encrypted at rest** with `PORTAL_SECRET_KEY`, a Fernet key
  the app never stores. Generate one with:

  ```bash
  python -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
  ```

  Lose it and every stored credential is unrecoverable. That is the correct
  failure mode — a stolen `portal.db` on its own is not enough. Under Docker the
  key is read from `/run/secrets/PORTAL_SECRET_KEY` like any other secret.

- **Without the key the feature is simply off.** The store stays closed, the Add
  button does not render, and everything defined in `systems.json` works exactly
  as before. Nothing degrades quietly.

- **Credentials are tested before the system is saved.** The portal
  authenticates against the real PBX and rolls the row back if it fails, so a
  typo surfaces in the form rather than as a locked-out UCM three days later.
  If the box is not reachable yet — a firewall rule that has not landed — *Save
  anyway, untested* records that choice in the audit log.

- **Adding and removing are audited** like any other change, with the host and
  which credential roles were set. The values themselves are never written to
  the log.

**File-defined systems stay read-only.** A PBX from `systems.json` cannot be
removed through the UI, because the file is its source of truth and the deletion
would silently undo itself on the next restart. On an id clash the file wins.
Config you can review in a pull request outranks config someone typed into a
form at 2am, so keep the estate you manage deliberately in the file and let the
store hold the rest.

Removing a system deletes its stored credentials and rebuilds every provider,
so a cached session cannot outlive the system it belonged to.

## Firmware checking

Each manufacturer group in the rail has a **Check firmware** button. It fans out
across every PBX under that manufacturer that you have permission for, reads the
installed version, and reports which ones are behind. The button sits on the
manufacturer rather than each PBX because the real question is "which of my
Grandstreams are behind", not "what version is Hamilton on".

Version sources, all verified against vendor documentation:

| Manufacturer | Call | Version field |
| --- | --- | --- |
| 3CX | `GET /xapi/v1/SystemStatus` | `Version` |
| Yeastar | `GET /openapi/v1.0/system/information` | `version` |
| Grandstream | `getSystemGeneralStatus` | `prog-version` (also `product-model`) |

**Where the target version comes from matters.** None of the three vendors
publishes a machine-readable feed of current releases — Grandstream lists them on
per-model web pages, Yeastar has each PBX check for itself, 3CX checks from its
admin console. Scraping those pages would break the first time a marketing team
restyles them. So the target comes from `backend/firmware-baseline.json`, which
you maintain, with a per-instance `firmware_target` override for hardware pinned
to an older branch. The portal's job is to tell you in one place which systems
disagree with a target you have chosen — which is the question you actually have
when an advisory lands.

Two implementation details worth knowing:

- **Version fields are found by searching the response recursively**, not at a
  fixed path. Each vendor wraps its payload differently and changes the envelope
  between releases; matching the leaf key survives that.
- **Comparison is numeric and component-wise.** A string compare would rank
  `1.0.20.9` above `1.0.20.13`, which is exactly the case that matters when the
  patch release is the one carrying the fix.

The check is read-only and deliberately not written to the audit log — that
trail is for changes, and routine checks would bury them.

## Audit trail

Every write is recorded in SQLite (`backend/audit.db`) with the actor, **which
PBX it landed on**, the inputs, and the result. Field names matching
`key|secret|token|password|authorization|cookie|vmsecret` are redacted before
storage — SIP secrets and voicemail PINs pass through these forms. Users see
their own entries; `read:audit` shows everyone's.

## Running with Docker

See **[DEPLOY.md](DEPLOY.md)** for the full build, run and Auth0 walkthrough.
The short version:

```bash
cp .env.example .env          # Auth0 domain, client id, audience
bash scripts/init-secrets.sh     # creates one file per credential + the encryption key
docker compose up --build -d
```

Then open http://localhost:8080.

## Testing

```bash
cd backend
pip install -r requirements-dev.txt
pytest
```

The suite runs offline against a simulated estate of three PBXs built on
`httpx.MockTransport`, so it exercises the real credential and transport code
rather than mocking it away. What it covers:

- **Auth** — valid tokens, expiry, wrong audience and issuer, a malformed
  permissions claim, an oversized token, and the classic JWT forgery: an HS256
  token signed with the RSA public key.
- **Proxy** — OAuth2 token caching, Yeastar's query-parameter token and
  mandatory `User-Agent`, the Grandstream challenge handshake, the automatic
  `applyChanges`, silent recovery from an expired session, response masking,
  the field allowlist, and path-value escaping.
- **Firmware** — numeric version comparison, recursive field discovery across
  different response envelopes, model-substring baseline matching, per-instance
  target overrides, and per-PBX failure isolation.
- **Store and audit** — credential encryption (asserting the plaintext is not
  in the ciphertext), rollback when credentials fail verification, permission
  gating, protection of file-defined systems, and hash-chain tamper detection.
- **Security primitives** — rate limiter keying, breaker scoping and reset,
  header-injection rejection, recursive masking.

One gotcha if you extend the fixtures: modules capture `settings = get_settings()`
at import, so database paths cannot vary per test. `conftest.py` uses a
session-scoped directory and deletes the files between tests instead.

## Security

This portal is a privileged proxy. It holds credentials that are effectively PBX
administrator on all three systems, and a compromised phone system converts
straight into money via toll fraud. The controls below assume that threat, not a
generic internal CRUD app.

**Identity and authorisation.** Access tokens are verified RS256-only against
Auth0's JWKS, with audience, issuer, expiry and subject all required rather than
optional. Pinning the algorithm is what prevents an attacker swapping RS256 for
HS256 and self-signing with the public key. Authorisation reads the `permissions`
claim alone; the roles claim is display sugar and never decides access. Failed
validation returns a flat "Invalid token" so the endpoint cannot be used to probe
why a token was rejected.

**Credentials.** PBX secrets never leave the server process. They are attached in
`proxy.py` and nowhere else, stripped from every response, and masked out of
response bodies via `mask_response_fields` — Grandstream's `getSIPAccount`
returns SIP and voicemail passwords in clear text, and this portal exists to
change settings, not to hand out extension credentials.

**Rate limits.** Writes get 10/minute per user per endpoint, reads 60, with a
120/minute global bucket. Limits apply after authorisation and are keyed per
endpoint, so an unauthorised caller cannot map the registry by watching which ids
throttle. These are per process: run more than one worker and put a shared
limiter in front.

**Credential circuit breaker.** Three consecutive login failures against a system
open a five-minute cooldown. Without this, one stale password in the environment
becomes a login loop, and on Grandstream a login loop gets this server onto the
UCM's IP blacklist — a config typo turning into an outage.

**Input handling.** Path values are percent-encoded, so a form field cannot
rewrite the URL. Header values are rejected if they contain control characters.
Any field not declared in the registry is dropped before the upstream call, so
an authenticated user cannot smuggle extra parameters. Payloads are capped at
20k characters and responses at 200k.

**Information disclosure.** Users without an endpoint's permission see that it
exists but not its `base_url` or credential variable names — someone with no
rights should not learn your PBX hostnames and ports from the catalogue. Upstream
exceptions carry hosts, ports and TLS detail, so they are logged and replaced
with a summary in the response. `/healthz` reveals nothing about what is
configured, unhandled errors return a bare 500, and the OpenAPI schema is off by
default.

**Audit integrity.** Every action reaches the PBX as one shared API account, so
the phone system's own log cannot attribute anything to a person. This log is the
only attribution that exists, so rows are SHA-256 hash-chained, the database file
is chmod 600, and each entry is mirrored to the process log. `GET
/api/activity/integrity` re-walks the chain and names the first broken row. Ship
the process log off the box: an attacker with server access can still delete the
database, and only the copy that already left survives.

**Transport.** Plaintext `base_url`s are refused at startup. Redirects are not
followed, so an upstream redirect cannot replay a credential at an attacker's
host. CORS rejects a wildcard origin outright, and responses carry `no-store`,
`nosniff`, `frame-ancestors 'none'` and HSTS.

**Browser tokens.** Tokens are memory-only by default, so an XSS bug does not
also yield a working admin session. The cost is re-authenticating after a page
reload; `VITE_TOKEN_CACHE=localstorage` reverses that trade if you decide it is
wrong for you.

### Residual risks

- **One shared PBX identity.** Every change lands as the same API account. The
  portal's audit log is the only per-person record; treat it as evidence.
- **Token revocation lags.** Removing an Auth0 role does not invalidate a token
  already in a browser. Set a short lifetime on the API — 15 minutes or so is
  your worst-case window after an offboarding.
- **Rate limits are per process.** They multiply by worker count.
- **SQLite is single-writer.** Fine for one worker, not for several.
- **Secrets sit in environment variables**, visible in `/proc` and crash dumps.

## Before production

- **Reach the PBXs over a private network or VPN.** A UCM with its HTTPS API on
  the public internet gets its login blacklist tripped by scanners, locking out
  this portal along with the attacker.
- **Swap SQLite for Postgres** if more than one API worker will run, and add a
  shared rate limiter (Redis, or your ingress) at the same time.
- **Move the PBX secrets to a secret manager** (AWS Secrets Manager, Vault,
  Doppler). Providers read `os.environ` at call time, so the change is confined
  to `_require_env()` in `credentials.py`.
- **Keep TLS verification on.** If a UCM or Yeastar box uses a self-signed
  certificate, point `httpx.AsyncClient(verify=...)` in `proxy.py` at your CA
  bundle. Never `verify=False`.
- **Set a short access-token lifetime** on the Auth0 API.
- **Ship the audit log to an append-only sink**, and check
  `/api/activity/integrity` on a schedule.
