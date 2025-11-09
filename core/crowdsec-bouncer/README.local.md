# CrowdSec Traefik Bouncer (VM300)

## Layout
- `appdata/` : zone de persistance si l’on active des dumps côté bouncer.
- `config/` : overrides additionnels (vide par défaut).
- `logs/` : cible optionnelle pour exporter les journaux.
- `secrets/crowdsec_bouncer_api_key` : clé fournie par la LAPI VM100.

## Compose
Le container ne monte rien par défaut ; ajouter des volumes dans `docker-compose.yml` selon vos besoins de debug.
