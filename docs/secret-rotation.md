# Secret rotation runbook

> ⚠️ **Why this exists:** the bootstrap secrets below were generated and **shown in plaintext in
> a chat session (2026-06-10)**. Treat every value as **compromised** and rotate all of them —
> including the Infisical `ENCRYPTION_KEY` (the master key for everything Infisical stores).
> Do this **before** putting any real data into Infisical, while rotation is still trivial.
>
> This file lists secret **names/keys only — never commit actual secret values to it.** The repo
> is public; keep filled-in values in a password manager, not here. (Consider `git rm --cached`
> + `.gitignore` if you'd rather not track this runbook at all.)

## Inventory — secrets that must be rotated

| Namespace | Secret | Keys | Consumed by |
|-----------|--------|------|-------------|
| `monitoring` | `grafana-admin` | `admin-user`, `admin-password` | Grafana admin login |
| `infisical` | `infisical-postgresql` | `password` | Postgres StatefulSet **and** the `DB_CONNECTION_URI` below (same value) |
| `infisical` | `infisical-secrets` | `ENCRYPTION_KEY`, `AUTH_SECRET`, `SITE_URL`, `DB_CONNECTION_URI`, `REDIS_URL` | Infisical app (loaded via `envFrom`) |
| `external-secrets` | `infisical-universal-auth` | `clientId`, `clientSecret` | ESO `ClusterSecretStore` (created later — see §4) |

> `SITE_URL` and `REDIS_URL` aren't secret (no credentials) — leave them. The Postgres password
> appears in **two** places (`infisical-postgresql.password` **and** inside
> `infisical-secrets.DB_CONNECTION_URI`); they must stay identical.
>
> Not in scope (you created these yourself earlier, not shown in this chat): `n8n-encryption-key`,
> `n8n-postgresql`. Rotate them too if you're unsure they're clean — same pattern as §2.

---

## 0. Initial setup — creating these secrets from scratch

If a secret doesn't exist yet (fresh cluster, or you deleted one to rotate it), create it as below.
These must exist **before** the corresponding app syncs (the manual-sync apps won't sync until you
trigger them, which gives you time). All values are random — record them in your password manager.

```bash
# --- monitoring: Grafana admin ---
kubectl create namespace monitoring --dry-run=client -o yaml | kubectl apply -f -
kubectl -n monitoring create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password="$(openssl rand -hex 24)"

# --- infisical: Postgres password (the SAME value is reused in DB_CONNECTION_URI below) ---
kubectl create namespace infisical --dry-run=client -o yaml | kubectl apply -f -
PGPW="$(openssl rand -hex 24)"
kubectl -n infisical create secret generic infisical-postgresql \
  --from-literal=password="$PGPW"

# --- infisical: app secrets (loaded by the app via envFrom) ---
#   ENCRYPTION_KEY : master key for everything Infisical stores — back it up, never rotate after use
#   AUTH_SECRET    : signs session JWTs
#   SITE_URL       : must match the HTTPRoute host (http until TLS is added)
#   DB_CONNECTION_URI / REDIS_URL : point at the self-managed Postgres + Valkey in-cluster
kubectl -n infisical create secret generic infisical-secrets \
  --from-literal=ENCRYPTION_KEY="$(openssl rand -hex 16)" \
  --from-literal=AUTH_SECRET="$(openssl rand -base64 32)" \
  --from-literal=SITE_URL="http://infisical.attat.org" \
  --from-literal=DB_CONNECTION_URI="postgresql://infisical:${PGPW}@infisical-postgresql:5432/infisicalDB?sslmode=disable" \
  --from-literal=REDIS_URL="redis://infisical-redis:6379"
```

Then sync the `infisical` Application in ArgoCD. The ESO machine-identity secret
(`infisical-universal-auth`) is created later, once Infisical itself is running — see §4.

Verify what exists:
```bash
kubectl -n monitoring get secret grafana-admin
kubectl -n infisical  get secret infisical-postgresql infisical-secrets
```

---

## 1. Grafana admin (`monitoring/grafana-admin`)

The Grafana chart applies `admin-password` from the secret at pod startup, so rotating the secret
and restarting picks it up:

```bash
kubectl -n monitoring create secret generic grafana-admin \
  --from-literal=admin-user=admin \
  --from-literal=admin-password="$(openssl rand -hex 24)" \
  --dry-run=client -o yaml | kubectl apply -f -
kubectl -n monitoring rollout restart deploy/grafana   # (statefulset/grafana if it's a STS)
```

If the login doesn't take after the restart (Grafana keeps users in its own DB), force it from
inside the pod:

```bash
kubectl -n monitoring exec deploy/grafana -c grafana -- \
  grafana cli admin reset-admin-password "$(openssl rand -hex 24)"
```

…then update the `grafana-admin` secret to match so a future reinstall stays consistent. Record
the new password in your password manager.

---

## 2. Infisical Postgres password (`infisical-postgresql` + `DB_CONNECTION_URI`)

`POSTGRES_PASSWORD` only sets the password on **first** DB init, so for an already-running DB you
must change it inside Postgres (or wipe the PVC). Since this is fresh, the wipe is simplest:

**Option A — wipe (fresh DB, no data yet):**
```bash
NEWPW="$(openssl rand -hex 24)"
# delete the StatefulSet's data so POSTGRES_PASSWORD re-inits with the new value
kubectl -n infisical delete statefulset infisical-postgresql
kubectl -n infisical delete pvc data-infisical-postgresql-0
# update both secrets (password + the URI that embeds it)
kubectl -n infisical create secret generic infisical-postgresql \
  --from-literal=password="$NEWPW" --dry-run=client -o yaml | kubectl apply -f -
kubectl -n infisical get secret infisical-secrets -o json \
  | jq --arg uri "postgresql://infisical:${NEWPW}@infisical-postgresql:5432/infisicalDB?sslmode=disable" \
       '.data.DB_CONNECTION_URI = ($uri | @base64)' \
  | kubectl apply -f -
# re-sync the infisical Application in ArgoCD (recreates the StatefulSet), then restart the app
kubectl -n infisical rollout restart deploy/infisical
```

**Option B — keep data:** `kubectl -n infisical exec -it sts/infisical-postgresql -- \
psql -U infisical -d infisicalDB -c "ALTER USER infisical PASSWORD '<newpw>';"` then update both
secrets (as above) and `rollout restart deploy/infisical`.

---

## 3. Infisical master keys (`infisical-secrets`)

- **`ENCRYPTION_KEY`** — master key for all data Infisical encrypts. On a **fresh** instance with
  nothing stored it's safe to replace outright. **Once you've stored secrets, changing it makes
  them unreadable** (Infisical won't re-encrypt automatically). → rotate it **now**, before use.
  Generate with `openssl rand -hex 16`.
- **`AUTH_SECRET`** — signs session JWTs. Rotating it logs everyone out (they re-login). Safe
  anytime. Generate with `openssl rand -base64 32`.

```bash
ENC="$(openssl rand -hex 16)"
AUTH="$(openssl rand -base64 32)"
kubectl -n infisical get secret infisical-secrets -o json \
  | jq --arg e "$ENC" --arg a "$AUTH" \
       '.data.ENCRYPTION_KEY=($e|@base64) | .data.AUTH_SECRET=($a|@base64)' \
  | kubectl apply -f -
kubectl -n infisical rollout restart deploy/infisical
```

Back up `ENCRYPTION_KEY` securely — if it's lost, Infisical's stored secrets are unrecoverable.

---

## 4. ESO machine identity (`external-secrets/infisical-universal-auth`)

Not created yet — it's the credential ESO uses to read from Infisical. Create it when wiring up
the `ClusterSecretStore` (see README → "External Secrets / Infisical"):

1. In Infisical: **Admin → Machine Identities → create**, method **Universal Auth** → copy the
   **Client ID** + **Client Secret**.
2. Add that identity to your Infisical **project** with at least **read** access.
3. Store the creds (these are sensitive — keep them out of Git):
   ```bash
   kubectl -n external-secrets create secret generic infisical-universal-auth \
     --from-literal=clientId='<client-id>' \
     --from-literal=clientSecret='<client-secret>'
   ```
4. In `deployments/external-secrets/values.yaml` set `infisical.enabled: true` and fill
   `projectSlug` / `environmentSlug`, then sync the `external-secrets` Application.

To rotate later: regenerate the client secret on the machine identity in Infisical, then
re-apply the k8s secret (step 3) and let ESO pick it up.

---

## After rotating

- Confirm pods restart cleanly: `kubectl -n monitoring get po`, `kubectl -n infisical get po`.
- Log into Grafana / Infisical with the new credentials.
- From here on, manage **non-bootstrap** app secrets in Infisical and sync them via ESO
  `ExternalSecret`s — only the four bootstrap secrets above must remain plain k8s Secrets
  (Infisical/ESO need them to exist before they can run).
