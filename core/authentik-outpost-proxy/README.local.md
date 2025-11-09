# Authentik Outpost (VM300)

## Layout
- `appdata/` : réserve pour caches/outils locaux.
- `config/` : déposer d’éventuelles surcharges d’outpost.
- `logs/` : dossier cible pour les journaux si on monte un volume.
- `secrets/outpost_token` : token fourni par Authentik.

## Compose
Le service tourne en lecture seule sans volumes par défaut ; les montages supplémentaires sont à définir dans `docker-compose.yml`.
