# Standalone ArgoCD Applications

Applications here are synced by the `bootstrap` App-of-Apps (it reads `argocd/`
recursively). Use this directory for Helm charts that the `infra` / `applications`
ApplicationSets can't consume via their directory generator.

## authentik.yaml - Authentik SAML IdP (Lecture 9)

Deploys the Authentik Helm chart (`2026.5.4`) at
`https://authentik.192.168.50.10.nip.io`. Authentik 2026.x needs **no Redis** - only
PostgreSQL.

### Manual prerequisite: create the Authentik database (do this FIRST)

To keep the small worker VMs from running out of memory, Authentik does **not** use a
bundled database - it **reuses the shared lab PostgreSQL** on the `haproxy` VM
(`192.168.50.10:5432`, the same instance Keycloak uses). You must create its database
and role **before** ArgoCD syncs Authentik, or the pods will crash-loop.

On the lab host, as the Postgres superuser (the `haproxy` VM runs Postgres):

```bash
# 1. create the login role
multipass exec haproxy -- sudo -u postgres psql \
  -c "CREATE ROLE authentik WITH LOGIN PASSWORD 'authentik-db-lab-2026';"

# 2. create the database owned by that role
multipass exec haproxy -- sudo -u postgres psql \
  -c "CREATE DATABASE authentik OWNER authentik;"

# 3. verify
multipass exec haproxy -- sudo -u postgres psql -c "\l authentik"
```

The connection details must match `authentik.postgresql` in `authentik.yaml`:

| Setting | Value |
|---------|-------|
| host | `192.168.50.10` (the haproxy VM) |
| port | `5432` |
| database | `authentik` |
| user | `authentik` |
| password | `authentik-db-lab-2026` (lab value; not a real secret) |

> The Postgres superuser password is on the VM at
> `multipass exec haproxy -- sudo cat /root/postgres_credentials.txt`.

### Bootstrap admin

The chart's `global.env` sets `AUTHENTIK_BOOTSTRAP_PASSWORD`,
`AUTHENTIK_BOOTSTRAP_TOKEN`, and `AUTHENTIK_BOOTSTRAP_EMAIL`, so on first boot
Authentik creates the `akadmin` user (password `akadmin-lab-2026`) and an API token
(`authentik-bootstrap-token-lab-2026`) for scripted configuration.
