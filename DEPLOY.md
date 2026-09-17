# Deploying the update portal

Two images, one compose file. nginx serves the SPA and proxies `/api` to
FastAPI; the backend publishes no port, so nothing outside the compose network
can reach it. That is what makes the browser same-origin — no CORS to configure,
and the CSP's `connect-src` narrows to your Auth0 tenant.

Do Auth0 first. The portal runs fine with zero PBX credentials configured, so
you can prove sign-in works before any phone system is involved.

---

## 1. Auth0

### Create the API

Applications → APIs → **Create API**.

| Field | Value |
| --- | --- |
| Name | Update portal API |
| Identifier | `https://update-portal-api` |
| Signing algorithm | RS256 |

The Identifier is your `audience`. It never has to resolve to anything.

On the API's **Settings** tab, enable:

- **Enable RBAC**
- **Add Permissions in the Access Token**
- **Allow Offline Access**

Set **Token Expiration** to `900`. Removing someone's role does not invalidate a
token already in a browser, so that number is your worst-case window after an
offboarding.

### Add permissions

The API's **Permissions** tab:

```
update:3cx            Change extensions on the 3CX systems
update:3cx-lab        Change the lab 3CX only
update:yeastar        Change extensions on the Yeastar systems
update:grandstream    Change extensions on the Grandstream UCMs
manage:systems        Add and remove phone systems through the portal
read:audit            See everyone's activity, not just your own
```

These must match `required_permission` in `backend/systems.json`. If you rename
a manufacturer or add a per-instance override, add the permission here too.

### Create a role

User Management → Roles → **Create Role**, add the permissions, assign it to your
user. Permissions only reach a token through a role — a user with none gets a
valid token and no access, which is the correct failure mode but looks like a
broken app.

### Create the SPA

Applications → **Create Application** → Single Page Web Application.

| Field | Value |
| --- | --- |
| Allowed Callback URLs | `http://localhost:8080` |
| Allowed Logout URLs | `http://localhost:8080` |
| Allowed Web Origins | `http://localhost:8080` |

Add `http://localhost:5173` as well if you also run the Vite dev server. For a
real deployment use the actual origin, which must be `https://` — Auth0 refuses
plain HTTP callbacks anywhere but localhost.

Copy the **Domain** and **Client ID**.

---

## 2. Configure

```bash
cp .env.example .env
```

```bash
AUTH0_DOMAIN=your-tenant.us.auth0.com
AUTH0_CLIENT_ID=the-spa-client-id
AUTH0_AUDIENCE=https://update-portal-api
PORTAL_PORT=8080
```

Nothing secret goes in `.env`. Credentials are files:

```bash
bash scripts/init-secrets.sh
```

(Invoked with `bash` because the executable bit does not survive every transfer.
`chmod +x scripts/init-secrets.sh` if you prefer running it directly. The two
scripts inside the images are chmod'ed by their Dockerfiles, so they are
unaffected.)

That reads `backend/systems.json`, creates one empty file per credential it
names, and generates `secrets/PORTAL_SECRET_KEY`. **Back that key up.**
Credentials stored through the portal's Add PBX button cannot be recovered
without it — which is the point: a stolen database alone is not enough.

Fill each credential with `printf`, not `echo`:

```bash
printf '%s' 'your-client-secret' > secrets/THREECX_HQ_CLIENT_SECRET
```

A trailing newline is a common cause of an unexplained 401 and, on a Grandstream
UCM, a blacklisted server IP. The backend strips surrounding whitespace and logs
a warning if it has to, but it is better not to write it.

Empty files are fine. That PBX shows "no credentials" in the rail and every
other system works normally.

| PBX | Where to get the credential |
| --- | --- |
| 3CX | Admin console → Advanced → API. Create a key, enable it for XAPI. |
| Yeastar | PBX web portal → Integrations → API. Client ID and Client Secret. |
| Grandstream | UCM web UI → Integrations → API Configuration → HTTPS API Settings. |

**You do not list credentials in `docker-compose.yml`.** The backend image's
entrypoint turns every file in `/run/secrets` into the matching `*_FILE`
variable, so adding a PBX is two files and a `systems.json` entry — no compose
edit. Files keep the values out of `docker inspect`, `/proc` and crash dumps.

Finally, replace the `example.com` hosts in `backend/systems.json` with your own.

---

## 3. Build and run

```bash
docker compose build
docker compose up -d
docker compose ps
```

Both services should report `healthy`. Open **http://localhost:8080** and sign in.

Images are tagged `update-portal-backend:1.2.0` and
`update-portal-frontend:1.2.0`, so `docker save` or a push to your registry
works without further tagging.

### Verify the permissions actually arrived

This is the step people skip, and skipping it produces a portal where everything
is mysteriously read-only.

```bash
curl -H "Authorization: Bearer <token>" http://localhost:8080/api/me
```

Take the token from your browser's devtools network tab. You want a non-empty
`permissions` array. If it is empty, RBAC or "Add Permissions in the Access
Token" is off, or the role is not assigned.

---

## 4. What the images do

**Backend** (`python:3.12-slim`, multi-stage). Dependencies resolve into a
virtualenv that is copied into a clean runtime layer. Runs as uid 10001 with a
read-only root filesystem, all capabilities dropped and `no-new-privileges`.
Healthcheck uses `python -c` rather than curl, so there is no extra package to
patch. The audit database lives on a named volume — it is the only per-person
record of who changed what, so it must outlive the container.

**Frontend** (`nginxinc/nginx-unprivileged`, multi-stage). Vite builds, nginx
serves on 8080 as a non-root user, so no capability is needed to bind the port.

Two runtime details worth knowing:

- **Config is resolved at start-up, not build time.** A Vite build normally bakes
  `import.meta.env` into the bundle, which would mean one image per environment.
  The container writes `/config.js` from its environment instead, so the same
  image promotes from staging to production unchanged. Only public values go in
  it — an Auth0 domain, a SPA client id and an API identifier are all designed to
  be visible in a browser.
- **The CSP is assembled at start-up** because it has to name your Auth0 tenant
  in `connect-src` (token endpoint) and `frame-src` (silent-auth iframe).
  `NGINX_ENVSUBST_FILTER` restricts substitution to `PORTAL_*`, so nginx's own
  `$host` and `$uri` cannot be clobbered by a stray environment variable.

---

## 5. Beyond localhost

**Terminate TLS in front of this.** The compose stack is HTTP on 8080. Put it
behind a reverse proxy or ingress that does TLS; the HSTS header the API already
sets is aspirational until you do, and Auth0 will not accept a non-localhost
callback over plain HTTP.

**Do not scale `backend` past one replica.** Rate limits, the credential circuit
breaker and the SQLite audit store are all per process. A second replica silently
doubles your limits and will corrupt concurrent audit writes. Move to Postgres
and a shared limiter (Redis, or your ingress) first.

**Reach the PBXs over a private network.** A UCM with its HTTPS API on the public
internet gets its login blacklist tripped by scanners, which locks out this
portal along with the attacker.

**Self-signed PBX certificates.** Point `httpx.AsyncClient(verify=...)` in
`proxy.py` at your CA bundle. Never `verify=False`. If you are testing against a
plaintext lab box, startup refuses it until `REQUIRE_HTTPS_UPSTREAMS=false`.

**Before production:** commit `package-lock.json`, generate a hash-pinned
`requirements.txt` with `pip-compile --generate-hashes` and add
`--require-hashes` to the backend Dockerfile, pin base images by digest rather
than tag, and wire Trivy or `docker scout` into CI.

---

## 6. When something is wrong

| Symptom | Cause |
| --- | --- |
| Everything read-only, no error | RBAC or "Add Permissions in the Access Token" off, or no role assigned |
| 401 "Token was not issued for this API" | `AUTH0_AUDIENCE` mismatched between SPA and backend |
| Callback URL mismatch at login | Deployed origin not in Auth0's allow-lists |
| Signed out on every refresh | Expected. Tokens are memory-only; `TOKEN_CACHE=localstorage` changes it at the cost of XSS exposure |
| A PBX shows "no credentials" | Its file in `./secrets` is empty or missing |
| "stopped trying for 300s" | The circuit breaker opened after three failed logins. Fix the credential; this is what stops a typo becoming a UCM lockout |
| Frontend container exits immediately | `AUTH0_DOMAIN` not set — the CSP cannot be built without it |
| Backend exits on start | A plaintext `host` in `systems.json`, or a duplicate manufacturer or instance id. Both fail fast by design |

Useful commands:

```bash
docker compose logs -f backend
docker compose exec backend python -c "from app.registry import load_registry; \
  print({m: [i.id for i in v.instances] for m, v in load_registry().items()})"
```
