# CrowdSec Agent (VM300)

## Layout
- `config/` : fichiers `config.yaml.template`, `acquis.yaml`, credentials LAPI.
- `appdata/agent` : données agent `/var/lib/crowdsec/data`.
- `appdata/hub` : cache hub.
- `logs/` : journaux locaux de l’agent.
- `secrets/` : clé d’enrôlement + CA TLS.

## Compose
Le service monte `config/` en lecture seule, `appdata/*` en lecture/écriture et les logs Traefik via `${DOCKER_ROOT}` selon `docker-compose.yml`.
