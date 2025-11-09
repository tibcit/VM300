# Traefik Proxy (VM300)

## Layout
- `config/static.yml` et `config/dynamic/` : configuration Traefik.
- `appdata/acme` : stockage ACME.
- `logs/` : journaux Traefik (`/var/log/traefik`).
- `secrets/` : clés Cloudflare, CrowdSec bouncer, BasicAuth socket proxy.

## Compose
`docker-compose.yml` monte `config/` en lecture seule, `appdata/acme` en lecture/écriture et publie les ports 443/8080/9100 depuis la VM services.
