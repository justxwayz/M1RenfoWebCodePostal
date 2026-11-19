# Déploiement Kubernetes – Code postaux

Images : `ahu/php-cp` et `ahu/mysql-cp` (MySQL sur le port 3306). 1 réplica, stockage 1Gi.

## Déployer
    docker compose build && docker compose push      # (docker login ahu)
    kubectl apply -k k8s/
    kubectl config set-context --current --namespace=cp   # contexte par défaut
    kubectl get all,hpa,quota,limitrange

Accès : ajouter `cp.local` dans /etc/hosts (IP de l'ingress) ou `kubectl -n cp port-forward svc/php 8080:80`.

## Fichiers
| Fichier | Rôle |
|---|---|
| 00 | Namespace `cp` |
| 01 / 02 | ResourceQuota / LimitRange (requests & limits par défaut) |
| 03 | RBAC : ServiceAccounts `cp-app`, `cp-viewer`, `cp-deployer` + Roles/RoleBindings |
| 10 / 11 | ConfigMap / Secret |
| 20-22 | MySQL : PVC 1Gi, Deployment (Recreate), Service 3306 |
| 30-33 | PHP : Deployment (RollingUpdate), Service, Ingress, HPA |

## Ressources (faible performance)
| Pod | requests | limits |
|---|---|---|
| php | 50m / 64Mi | 200m / 128Mi |
| mysql | 100m / 256Mi | 500m / 512Mi |

Quota namespace : 6 pods, requests 500m / 768Mi, limits 2 CPU / 1280Mi, 1 PVC, 1Gi.

## Rolling update (maxSurge 1, maxUnavailable 0)
    kubectl -n cp set image deploy/php php=ahu/php-cp:v2
    kubectl -n cp rollout status deploy/php
    kubectl -n cp rollout undo deploy/php          # retour arrière

## HPA
    kubectl -n cp get hpa php -w
    # test de charge : kubectl -n cp run load --rm -it --image=busybox -- sh -c "while true; do wget -q -O- http://php; done"

## Maintenance d'un nœud (cordon / drain)
    kubectl cordon <noeud>        # plus aucun nouveau pod
    kubectl drain <noeud> --ignore-daemonsets --delete-emptydir-data
    # ... maintenance ...
    kubectl uncordon <noeud>
Avec 1 seul réplica, le drain provoque une courte coupure (le pod est recréé ailleurs). MySQL (volume RWO) doit pouvoir être remonté sur le nouveau nœud.

## RBAC
    kubectl -n cp auth can-i get pods --as=system:serviceaccount:cp:cp-viewer      # yes
    kubectl -n cp auth can-i delete pods --as=system:serviceaccount:cp:cp-viewer   # no
    kubectl -n cp auth can-i patch deploy --as=system:serviceaccount:cp:cp-deployer # yes

## Où trouver quoi

| Intitulé | Description | Utilité | Chemin | Lancer / vérifier |
|---|---|---|---|---|
| Namespace | Espace isolé `cp` pour tout le projet | Isoler et appliquer quotas/RBAC | `k8s/00-namespace.yaml` | `kubectl apply -f k8s/00-namespace.yaml` |
| ResourceQuota | Plafond global du namespace (pods, CPU, mémoire, 1Gi) | Empêcher de dépasser les ressources | `k8s/01-resourcequota.yaml` | `kubectl -n cp describe quota` |
| LimitRange | Requests/limits par défaut et bornes par conteneur | Aucun pod sans limites | `k8s/02-limitrange.yaml` | `kubectl -n cp describe limitrange` |
| RBAC | ServiceAccounts, Roles et RoleBindings | Gérer les accès (lecture / déploiement) | `k8s/03-rbac.yaml` | `kubectl -n cp auth can-i get pods --as=system:serviceaccount:cp:cp-viewer` |
| ConfigMap | Variables DB (hôte, nom, user, port 3306) | Configurer l'appli sans rebuild | `k8s/10-configmap.yaml` | `kubectl -n cp get cm cp-config -o yaml` |
| Secret | Mot de passe MySQL | Séparer les données sensibles | `k8s/11-secret.yaml` | `kubectl -n cp get secret cp-secret` |
| MySQL PVC | Volume persistant de 1Gi | Conserver la base | `k8s/20-mysql-pvc.yaml` | `kubectl -n cp get pvc` |
| MySQL Deployment | 1 pod `ahu/mysql-cp`, stratégie Recreate | Base de données | `k8s/21-mysql-deployment.yaml` | `kubectl -n cp logs deploy/mysql` |
| MySQL Service | Service interne `mysql:3306` | Accès à la base depuis PHP | `k8s/22-mysql-service.yaml` | `kubectl -n cp get svc mysql` |
| PHP Deployment | 1 pod `ahu/php-cp`, RollingUpdate (maxSurge 1, maxUnavailable 0) | Application web | `k8s/30-php-deployment.yaml` | `kubectl -n cp set image deploy/php php=ahu/php-cp:v2` puis `kubectl -n cp rollout status deploy/php` |
| PHP Service | Service interne `php:80` | Exposer l'appli dans le cluster | `k8s/31-php-service.yaml` | `kubectl -n cp port-forward svc/php 8080:80` |
| Ingress | Hôte `cp.local` vers le service PHP | Accès depuis l'extérieur | `k8s/32-php-ingress.yaml` | `kubectl -n cp get ingress` |
| HPA | Autoscaling 1 à 3 pods à 70 % CPU | Absorber la charge | `k8s/33-php-hpa.yaml` | `kubectl -n cp get hpa php -w` |
| Kustomization | Liste ordonnée de tous les manifestes | Tout déployer en une commande | `k8s/kustomization.yaml` | `kubectl apply -k k8s/` |
| Maintenance | cordon / drain / uncordon, contextes, tests | Mise à jour ou arrêt d'un nœud | `k8s/README.md` | `kubectl cordon <noeud>` puis `kubectl drain <noeud> --ignore-daemonsets --delete-emptydir-data` puis `kubectl uncordon <noeud>` |
| Docker Compose (dev) | Lancement local PHP + MySQL | Tester sans Kubernetes | `docker-compose.yml`, `.env.example` | `cp .env.example .env && docker compose up -d --build` |
| Docker Compose (prod) | Images Docker Hub sans build | Déploiement simple sur serveur | `docker-compose.prod.yml` | `docker compose -f docker-compose.prod.yml up -d` |
| Images | Dockerfiles PHP et MySQL (dump `cp.sql`) | Construire `ahu/php-cp` et `ahu/mysql-cp` | `php/Dockerfile`, `mysql/Dockerfile` | `docker compose build && docker compose push` |
| Code appli | Pages PHP et connexion PDO | Recherche de codes postaux + historique | `php/www/` | http://localhost:8080 |
