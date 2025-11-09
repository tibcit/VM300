# app-template

## Layout
- `appdata/` : données persistées (volume r/w).
- `config/` : fichiers de configuration montés en lecture seule.
- `logs/` : destination locale pour les journaux.
- `secrets/` : secrets montés via Docker.

## Utilisation
1. Copier ce dossier : `cp -R applications/app-template applications/ma-nouvelle-app`.
2. Rechercher/remplacer `app-template` et ajuster `.env.example`, `docker-compose.yml`, `README.local.md`.
3. Ajouter le service aux bons profils dans `compose.yaml`.
