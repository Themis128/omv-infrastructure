# authentik target architecture — cloudless.gr / SocialAuto / Nextcloud

## Current state (2026-11-04)

- Server: k3s workload on omv-ha cluster, NodePort `192.168.1.128:30910`,
  fronted by `cloudflared-omv` on omv → `https://auth.cloudless.gr`.
- Outpost: `authentik-outpost` on Docker Desktop (cu130-slim) proxying SocialAuto.
- SocialAuto OIDC: issuer `https://auth.cloudless.gr/application/o`, client `social_client`.
- Gaps: no IaC, no admin API token, MFA posture unknown, admin UI likely public.

## Target design


## Target topology

```mermaid
flowchart TB
    subgraph public[Public plane]
        U[Users / browsers] -->|OIDC redirects| CF[Cloudflare edge]
        CF --> TUN[cloudflared-omv]
        TUN --> NP[NodePort 30910]
    end
    subgraph tailnet[Tailnet private plane]
        ADM[Admin / agents] -->|/if/admin| TS[tailscale serve omv-ha]
        OPS[authentik-ops scripts] -->|API + kubectl| TS
    end
    subgraph k3s[omv-ha k3s — ns: authentik]
        NP --> SVC[authentik-server]
        TS --> SVC
        SVC --> PG[(postgres)]
        SVC --> RD[(redis)]
        SVC --> WK[authentik-worker]
    end
    SVC -->|OIDC| SA[SocialAuto]
    SVC -->|OIDC| NC[Nextcloud omv AIO]
    SVC -->|OIDC| PT[cloudless.gr portal]
    SVC -->|OIDC| GR[Grafana]
    OP[authentik-outpost · docker-desktop] -->|enrol via tailnet| SVC
    SVC --> MT[/metrics → Prometheus]
    SVC --> BK[pg_dump → NAS backup]
```

### Placement (keep k3s — correct call)
- Dedicated `authentik` namespace on omv-ha k3s: `authentik-server` (web+api),
  `authentik-worker` (bg tasks), postgres + redis in-cluster (CNPG/Bitnami or
  the bundled chart deps). PVCs on longhorn/local-path — backup via the existing
  NAS job to `omv-infrastructure`-managed backup units.
- Front door stays Cloudflare Tunnel → NodePort for now; upgrade path: point
  cloudflared at Traefik ingress (`auth.cloudless.gr` → ingress svc) for
  per-host TLS + rate limiting once traefik is healthy.

### Access tiers
1. **Public endpoint** — `/application/o/*` (OIDC endpoints) and app login pages
   only. Everything else returns to default-deny.
2. **Admin UI** — `/if/admin` restricted to Tailscale via Cloudflare Access
   policy (device posture: tailnet membership). No public admin surface.
3. **Machine-to-machine** — API tokens (per-agent, scoped, rotating; the
   `authentik-ops` skill handles minting + storage in `.env`).

### Providers + applications (one OIDC provider per relying party — never share clients)

| Relying party | Type | Notes |
|---|---|---|
| SocialAuto (social-api) | OIDC `social_client` | existing; add `groups` claim → role map (admin/staff). |
| cloudless.gr client portal | OIDC | Auth.js/NextAuth or the site API validating id_tokens; enrollment flow for external users. |
| Nextcloud (omv AIO) | OIDC via `user_oidc` app | groups claim → NC groups; `cloudless-admins` → NC admin; `quota`/uid mapping per Nextcloud docs. |
| Grafana (omv k3s) | OIDC | org-role mapping from authentik groups. |
| n8n | Proxy provider via outpost OR SAML | front `/` with outpost + MFA policy. |
| Metabase / EspoCRM / Portainer | OIDC/SAML per app | EspoCRM: SAML2 or OIDC plugin; Portainer: OAuth. |
| Outpost-protected admin UIs | Proxy provider | docker-desktop outpost covers social-api admin routes; add an omv outpost for k3s dashboards (k8s dashboard, longhorn UI). |


### Tailscale fabric (private plane — always preferred per global rule)
- **Admin + ops traffic never touches Cloudflare**: `k3s kubectl`, `ak` CLI,
  authentik admin UI, and API calls go over the tailnet —
  `https://auth.cloudless.gr` stays the browser-facing issuer, but internal
  service calls hit the cluster directly.
- **Admin UI fast path**: `tailscale serve` on omv-ha (or a tailnet Ingress)
  exposes `/if/admin` at `https://authentik.omv-ha.<tailnet>.ts.net` — Tailscale
  HTTPS cert, tailnet-only. CF Access policy on the public path becomes
  defense-in-depth, not the only gate.
- **Outposts dial the server over tailnet** where possible: the docker-desktop
  `authentik-outpost` (office-1 tailnet) and a future omv outpost connect to
  the server via the tailnet/cluster address (`100.x:30910` or the cluster
  Service) — keeps enrolment traffic off the WAN even though browser OIDC
  still uses the public issuer URL.
- **Containers**: social-api OIDC redirects must keep the public issuer
  (browsers need it), but server-side token/userinfo calls can ride tailnet
  DNS — document both endpoints per provider.
- **ACLs**: tag `authentik-server` access to `cloudless-admins` + server nodes
  only in the tailnet ACL (tag:authentik-admin); the NodePort itself stays
  reachable for cloudflared.

### Groups + roles
- `cloudless-admins` — full access everywhere (you + service agents).
- `cloudless-users` — cloudless.gr portal + Nextcloud files.
- `social-users` — SocialAuto only.
- Groups ride the `groups` claim in every OIDC provider (default scope mapping).

### Flows
- **identification → password → TOTP/WebAuthn** mandatory for `cloudless-admins`;
  optional for users initially.
- **enrollment** flow on cloudless.gr sign-up (email verification → cloudless-users).
- **recovery** flow with email verification; no self-admin.

### Security posture
- Redirect URIs allowlisted per provider (no wildcards).
- Access-token TTL ≤10min, refresh 30d sliding.
- Session binding to IP+UA for admin group.
- Brute-force: authentik reputation + CF rate limit on `/application/o/token`.
- Tokens/secrets only in `.env` + secret store — never in provider config exports.

### Infra-as-code
- `omv-infrastructure/omv/authentik/blueprint.yaml` — providers, applications,
  groups, flows as an authentik **blueprint** (declarative; applied via
  `ak apply_blueprint`). Single source of truth, GitOps through this repo.
- `scripts/authentik-backup.sh` — pg_dump + media dir → NAS job, alongside the
  existing nextcloud-backup units.

### Observability
- `authentik-server` exposes `/metrics` → Prometheus (existing omv monitoring).
- Grafana panel: logins/denials/MFA adoption; alert on >N failed logins (telegram-alert-ops P2).
- Sentry DSN on authentik-server for exception tracking.

## Rollout order
1. Namespace/backup baseline + blueprint skeleton (commit to this repo).
2. CF Access policy on `/if/admin` (Tailscale-only) — biggest risk reduction, zero app change.
3. Admin API token (authentik-ops `create_admin_token.sh`) → `.env` → blueprint apply via API.
4. Nextcloud `user_oidc` wiring (manual occ config → captured in blueprint).
5. SocialAuto group-claim role mapping.
6. cloudless.gr portal provider + enrollment flow.
7. Grafana/n8n/Metabase/EspoCRM providers as each is touched.
