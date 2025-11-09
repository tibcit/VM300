# Mémo – Plateforme Admin / Services

## Vue globale commune

          +----------------------+                        +----------------------+
          |      VM300 (LAN30)   |  Agents / Apps / UI    |      VM100 (LAN10)   |
          |  - Traefik proxy     |  ─────────────TLS────> |  - Traefik admin     |
          |  - Authentik outpost |                        |  - Authentik core    |
          |  - CrowdSec agent    |  Telemetry OTLP/HTTP   |  - CrowdSec LAPI     |
          |  - Alloy agent       |  ───────┬────────────> |  - Telemetry stack   |
          +----------------------+         |              +----------------------+
                                           |  Grafana Alloy scrapes Traefik &
                                           |  node exporters on both VLANs
                                           ▼
                                   Observabilité centralisée

- **Traefik**: chaque VM possède un traefik dédié (admin vs services) avec mêmes chaînes de middlewares (`crowdsec → rate-limit → buffering → forwardauth → headers → compression`). ACME DNS-01 Cloudflare, stockages locaux.
- **Authentik**: VM100 héberge Authentik (Postgres + Redis + worker); chaque VM possède un outpost-proxy (forward-auth) exposé publiquement (`auth-admin.seite.me`, `auth.seite.me`). Les outposts ne reçoivent pas de forward-auth sur leur propre routeur.
- **CrowdSec**: VM100 fournit LAPI/DB + bouncer Traefik; VM300 n’exécute qu’un agent + bouncer liés au LAPI via TLS. API keys stockées en secrets Docker.
- **Télémétrie**: VM100 exécute Prometheus/Loki/Tempo/Pyroscope + Alloy; VM300 héberge Alloy agent. OTLP endpoints `alloy:4317/4318` en interne, Traefik expose remote-write/Pyroscope/Tempo/Loki via BasicAuth secrets.
- **Conventions**:
  - `.env` global accessible via `./.env`; chaque stack ajoute ses propres `.env` relatifs (`ENV_*` dans le global).
  - IPs/ VLANs unifiés: `VM100_IP`, `VM300_IP`, `VLAN10_CIDR`, `VLAN30_CIDR`.
  - Secrets sensibles toujours via Docker secrets (`/run/secrets/...`).
  - Images `latest` sauf dossiers `core/*` qui pin une version datée.
  - Sécurité par défaut: `read_only`, `tmpfs /tmp`, `cap_drop: [ALL]`, `no-new-privileges: true` pour frontaux/exporters.

## Spécifique VM300 (services plane)

### Rôle

- Plan exécution applicatif (media stack, automations, dashboards) derrière Traefik-proxy (`traefik.seite.me`).
- Fournit flux de télémétrie (Alloy agent → VM100 remote-write, Loki push, Tempo OTLP, Pyroscope) et relaie events CrowdSec vers VM100.

### Réseaux & dépendances

- Réseaux externes: `t3_proxy`, `socket_proxy` (pour auto-discovery Homepage/Deployrr).
- Traefik dépend du service `authentik-outpost-proxy` (wait-for start) pour forward-auth.
- CrowdSec agent (`core/crowdsec-agent`) enrol via `crowdsec_enroll_key` secret, s’attache au LAPI TLS (CA cert monté).
- Alloy agent scrape Traefik metrics via entrypoint `metrics@internal` + Traefik 9100 exporter, envoie OTLP vers VM100.

### Services cœur

- `core/traefik-proxy`: Traefik 3, DNS-01 Cloudflare, DNS records `TRAEFIK_DASHBOARD_DOMAIN`, full dynamic config (routers/services/middlewares) alignée sur VM100.
- `core/authentik-outpost-proxy`: forward-auth vers Authentik (`auth.seite.me`), token stocké dans `core/authentik-outpost-proxy/secrets/outpost_token`.
- `core/crowdsec-agent`: agent + acquisition (journaux système/nginx/docker), healthcheck `crowdsec -t`, BasicAuth to LAPI.
- `core/crowdsec-bouncer`: Traefik bouncer plugin key via secret, same middleware naming.
- `core/socket-proxy`: BasicAuth-protected Docker API pour Homepage & Deployrr discovery.
- `core/telemetrie-agent`: Grafana Alloy 1.11.3 pipeline (Prom scrape + Loki/Tempo/Pyroscope exporters) avec secrets `telemetry_*` (user/pass, API keys).

### Applications notables

- Media stack: Jellyfin, Radarr, Sonarr, Bazarr, Sabnzbd, Transmission (secrets for users).
- Ops dashboards: Homepage, Homarr, Deployrr; auto-discovery via socket-proxy, config `applications/homepage/*` avec blocs `docker:`.
- Automatisation: n8n, Notifiarr, Arcane agent.

### Points d’attention

- Remplir `core/authentik-outpost-proxy/secrets/outpost_token`, `core/traefik-proxy/secrets/cf_dns_api_token`, `core/crowdsec-agent/secrets/crowdsec_enroll_key`.
- Socket-proxy exposé uniquement aux dashboards (auth BasicAuth secret). Ne pas publier derrière `chain-internet-public`.
- Transmission user/pass secrets sont générés localement; partager via password manager si besoin d’accès UI.
- Garder `LAN_EXTRA_SOURCE_RANGES` aligné avec VLANs pour allowlists Traefik et bouncers.
