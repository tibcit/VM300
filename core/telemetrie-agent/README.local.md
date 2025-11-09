# Grafana Alloy Agent

## Layout
- `config/config.alloy` : pipeline Alloy (monté en lecture seule).
- `appdata/` : cache Alloy (`/var/lib/alloy`).
- `logs/` : dossier pour exporter les journaux d’agent si besoin.
- `secrets/telemetry_*` : identifiants remote-write / Loki / Tempo / Pyroscope.

## Compose
Voir `docker-compose.yml` : le conteneur monte `config/`, `appdata/` et de multiples volumes host (`/var/log`, `/var/lib/docker`, socket Docker, etc.).
