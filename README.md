# Code postaux (Docker)

## Lancer en local
    docker compose up -d --build     # 1er démarrage : ~1 min (import du dump + téléchargement browscap)
    # http://localhost:8080

## Arrêter / repartir de zéro
    docker compose down              # arrêt (données conservées)
    docker compose down -v           # arrêt + suppression de la base (réimport au prochain up)

## Production
    docker compose build && docker compose push          # (après docker login)
    # sur le serveur : copier docker-compose.prod.yml + .env, puis
    docker compose -f docker-compose.prod.yml up -d
    # reverse proxy -> http://127.0.0.1:8080 avec l'en-tête X-Forwarded-For

## Kubernetes
Voir [k8s/README.md](k8s/README.md) (`kubectl apply -k k8s/`).
