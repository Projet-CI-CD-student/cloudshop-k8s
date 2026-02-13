CloudShop – Déploiement Kubernetes Sécurisé


1. Présentation du projet

Ce projet consiste à déployer une application e-commerce simplifiée nommée CloudShop sur un cluster Kubernetes multi-nodes, en respectant des exigences strictes en matière d’architecture, de sécurité, de haute disponibilité et de gestion des accès.

L’application est composée de :

Un frontend web

Une API backend

Une base de données PostgreSQL

Un Ingress avec TLS


2. Architecture
Architecture logique
Internet
   ↓
Ingress (TLS)
   ↓
Frontend (Deployment - replicas ≥ 2)
   ↓
Backend (Deployment - replicas ≥ 2)
   ↓
Database (StatefulSet + PVC)

Principes appliqués

Namespace dédié

Services internes en ClusterIP

Exposition uniquement via Ingress

Isolation réseau via NetworkPolicies

RBAC avec ServiceAccounts dédiés

Node dédié à la base via taint

Stockage persistant via PVC


3. Structure du dépôt
cloudshop-k8s/
│
├── namespace/
├── deployments/
├── services/
├── ingress/
├── rbac/
├── networkpolicy/
├── storage/
└── README.md


4. Namespace

Toutes les ressources sont déployées dans un namespace dédié :

kubectl get ns
kubectl -n cloudshop get pods


Aucune ressource n’est présente dans le namespace default.


5. Deployments
Frontend

Type : Deployment

Replicas : 2 minimum

Stratégie : RollingUpdate

Backend

Type : Deployment

Replicas : 2 minimum

Stratégie : RollingUpdate

Anti-affinity configurée

Base de données

Type : StatefulSet

Stockage persistant via PersistentVolumeClaim

Déployée sur un node dédié via taint


6. Rolling Update (Zero Downtime)

Mise à jour du backend sans interruption de service :

kubectl -n cloudshop set image deploy/backend backend=hashicorp/http-echo:0.2.4
kubectl -n cloudshop rollout status deploy/backend


Comportement :

Création d’un nouveau ReplicaSet

Remplacement progressif des pods

Service toujours disponible

Aucun downtime observé


7. Ingress et TLS

L’application est exposée via Ingress avec routage basé sur le host et les paths.

https://cloudshop.local → frontend

https://cloudshop.local/api → backend

Test :

curl -k https://cloudshop.local
curl -k https://cloudshop.local/api


TLS est activé avec un certificat auto-signé.


8. NetworkPolicies

Les flux réseau sont strictement contrôlés.

Flux	Autorisé
Frontend → Backend	Oui
Backend → Database	Oui
Frontend → Database	Non
Namespace externe → Backend	Non

Vérification :

kubectl -n cloudshop get networkpolicy


9. RBAC
ServiceAccounts

Un ServiceAccount pour le frontend (aucun accès API)

Un ServiceAccount pour le backend

Rôle du backend

Le backend peut :

Lire les ConfigMaps

Lire les Secrets

ClusterRole lecture seule

Un ClusterRole readonly-pods autorise :

get

list

watch

Test utilisateur :

kubectl --kubeconfig=rbac/test-user.kubeconfig get pods -n cloudshop
kubectl --kubeconfig=rbac/test-user.kubeconfig delete pod -n cloudshop --all


L’utilisateur peut lire les pods mais ne peut pas les modifier.


10. Stockage persistant

La base de données utilise :

PersistentVolumeClaim

StorageClass standard

StatefulSet

Test :

kubectl -n cloudshop delete pod db-0


Après recréation du pod, les données restent disponibles.


11. Taints et Affinity

Un node est dédié à la base de données via un taint NoSchedule.

La base possède :

Une toleration correspondante

Une nodeAffinity

Le backend utilise une anti-affinity pour assurer la répartition sur différents nodes.

Vérification :

kubectl describe node minikube-m03
kubectl -n cloudshop get pod db-0 -o wide


12. Conclusion

Ce projet démontre :

Isolation via namespace

Haute disponibilité avec replicas et RollingUpdate

Sécurité réseau via NetworkPolicies

Contrôle d’accès via RBAC

Persistance des données via PVC

Isolation physique de la base via taints et affinity

Toutes les exigences techniques demandées ont été implémentées et validées.
