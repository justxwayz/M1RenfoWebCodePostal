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
