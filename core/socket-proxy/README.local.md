# Socket Proxy (Services Plane)

## Layout
- `appdata/` : réservé pour des métadonnées éventuelles.
- `config/` : personnalisations additionnelles (vides par défaut).
- `logs/` : dossier destiné aux journaux si l’on ajoute un bind.
- `secrets/` : BasicAuth ou autres secrets si activés.

## Compose
Même principe que côté VM100 : seule la socket Docker est montée ; ajuster `docker-compose.yml` pour exposer davantage d’API si nécessaire.
