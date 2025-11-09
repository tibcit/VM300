# VM300 · Plane Services (10.0.30.30)

Plan d’exécution applicatif (media stack, dashboards, automatisation). Consomme SSO/CrowdSec/observabilité fournis par VM100.

## Vue express

Users ─HTTPS─▶ Traefik-proxy (dns01) ─┬─▶ Apps médias (Jellyfin/Radarr/…)
├─▶ Dashboards (Homepage/Homarr/Deployrr)
├─▶ Automatisation (n8n, Arcane agent)
└─▶ Socket-proxy (Docker API)

CrowdSec agent ─TLS─▶ VM100 LAPI
Traefik bouncer ◀──── décisions
Alloy agent ─OTLP/HTTP─▶ VM100 Grafana Alloy ─► Prom/Loki/Tempo/Pyroscope
Authentik outpost ─forward-auth─▶ `auth.seite.me`

## Sous-systèmes

- **Traefik-proxy** (`core/traefik-proxy`) : ACME DNS-01 Cloudflare, chaînes `crowdsec→ratelimit→buffer→forwardauth→headers→compression`, dashboard protégé, routers pour chaque app.
- **Authentik outpost** (`core/authentik-outpost-proxy`) : token secret, fournit forward-auth local pour Traefik, jamais protégé par lui-même.
- **CrowdSec agent + bouncer** (`core/crowdsec-agent`, `core/crowdsec-bouncer`) : collecte journaux Traefik / host, envoie au LAPI VM100 via CA, plugin Traefik applique les décisions.
- **Grafana Alloy agent** (`core/telemetrie-agent`) : image 1.11.3, scrape Traefik (`metrics@internal`, 9100 exporter), collecte journaux Docker, exporte remote-write/Loki/Tempo/Pyroscope vers VM100.
- **Socket-proxy** (`core/socket-proxy`) : expose API Docker en lecture via BasicAuth (secrets), uniquement pour dashboards.
- **Stack médias** (`applications/{radarr,sonarr,jellyfin,...}`) : monte `../media` (data/downloads/torrents/usenet), secrets Transmission sous `applications/transmission/secrets/*`.
- **Dashboards & ops** (`applications/homepage`, `homarr`, `deployrr-dashboard`, `dashboard-sync`) : configs versionnées dans `appdata/config`.
- **Automatisation** (`applications/{n8n,notifiarr,cleanuparr,huntarr,arcane-agent}`) : dépendances vers Traefik, socket-proxy, services médias.

## Compose & profils

| Profil           | Services                                                                                                                      | Notes                               |
| ---------------- | ----------------------------------------------------------------------------------------------------------------------------- | ----------------------------------- |
| `core`           | Traefik, socket-proxy, crowdsec-agent, traefik-bouncer, authentik-outpost, arcane-agent                                       | Démarrage initial + dépendances TLS |
| `dashboard`      | dashboard-sync, deployrr-dashboard, homepage, homarr                                                                          | Discovery via socket-proxy          |
| `medias`         | bazarr, jellyfin, jellyseerr, radarr, sonarr, sabnzbd, transmission, prowlarr, kavita, mylar3, cleanuparr, huntarr, notifiarr | Montent `../media`                  |
| `apps`           | theme-park, freshrss, vscode                                                                                                  | Divers services non médias          |
| `automatisation` | n8n                                                                                                                           | Workflows                           |
| `monitoring`     | telemetrie-agent                                                                                                              | Envoi OTLP vers VM100               |

`compose.yaml` pilote chaque service via `extends` et injecte ses `.env` dédiés pointés par `ENV_*`.

## Réseaux, volumes & flux

[t3_proxy] ─ Traefik, socket-proxy, Alloy agent, toutes les apps exposées
[socket_proxy] ─ Traefik, dashboards, socket-proxy (API Docker)
Media host path ../media/ {data,downloads,torrents,usenet}

- Réseaux Docker externes requis : `t3_proxy`, `socket_proxy`. Créer une fois sur l’hôte.
- Flux sortants majeurs : CrowdSec agent → `https://10.0.10.30`, Alloy → Prom/Loki/Tempo/Pyroscope URLs, Traefik → Cloudflare DNS API.
- `applications/homepage|homarr|deployrr` lisent `appdata/config/*.yml` suivis en Git.

## Secrets & env

- `core/traefik-proxy/secrets/cf_dns_api_token`, `core/crowdsec-agent/secrets/{crowdsec_enroll_key,crowdsec_ca_cert}`, `core/crowdsec-bouncer/secrets/crowdsec_bouncer_api_key`, `core/authentik-outpost-proxy/secrets/outpost_token`.
- `core/socket-proxy` BasicAuth (`docker_api_basicauth`).
- `applications/transmission/secrets/{transmission_user,transmission_pass}`, `applications/vscode/secrets/password`, etc.
- Copier chaque `.env.example` → `.env` (racine + dossiers) pour définir domaines (`*.seite.me`), IPs, chemins (`MEDIA_ROOT=../media`), credentials non secrets.

## Runbook

1. **Bootstrap** : créer réseaux externes, copier `.env`, déposer secrets, `docker compose --profile core up -d`.
2. **Dashboards** : `docker compose --profile dashboard up -d` (vérifier accès SSO).
3. **Médias** : `docker compose --profile medias up -d`, contrôler montages `../media`.
4. **Apps & automatisation** : `docker compose --profile apps up -d`, `docker compose --profile automatisation up -d`.
5. **Monitoring** : `docker compose --profile monitoring up -d` pour Alloy.
6. **Upgrade** : `docker compose pull && docker compose up -d <profil>` ; Traefik/core en premier.

## Ops rapides

- Config Traefik : `docker compose exec traefik traefik version` puis `docker logs -f traefik`.
- CrowdSec agent : `docker compose logs -f crowdsec-agent`, `docker compose exec crowdsec-agent cscli metrics`.
- Alloy agent : `docker compose up -d telemetrie-agent` (rebuild) et `docker logs -f telemetrie-agent`.
- Validation : `docker compose config` + `docker network inspect t3_proxy`.

## Sécurité

- Services exposés : `read_only`, `cap_drop:ALL`, `no-new-privileges`, volumes `tmpfs` si prévus.
- Traefik & socket-proxy protégés par BasicAuth + Authentik forward-auth + CrowdSec bouncer.
- Secrets uniquement via `/run/secrets`; ne jamais committer `.env` ou contenus `secrets/`.

## Arborescence utile


core/
traefik-proxy/ (config/static.yml, config/dynamic/*.yml, appdata/acme)
crowdsec-agent/ (config/acquis, secrets)
crowdsec-bouncer/
authentik-outpost-proxy/
socket-proxy/
telemetrie-agent/ (config.alloy, secrets telemetry\__)
applications/
dashboard-sync · deployrr-dashboard · homepage · homarr
médias : bazarr, jellyfin, jellyseerr, radarr, sonarr, sabnzbd, transmission, prowlarr, kavita, mylar3
automation : n8n, notifiarr, cleanuparr, huntarr, arcane-agent
divers : freshrss, theme-park, vscode
compose.yaml · .env.example · memo.md · ../media/{data,downloads,torrents,usenet}
