# 🛒 Projet KubeShop (m1-k8s-lkc)

Sujet B - Kubeshop
Groupe : Loris - Kyllian - Corentin

Ce projet déploie une architecture micro-services complète sur Kubernetes via Helm. L'objectif est de simuler une boutique en ligne avec un frontend statique, une API de backend et une base de données persistante, le tout sécurisé par des politiques RBAC et exposé via un Ingress.

🏗️ Architecture du projet

L'application est isolée dans le namespace kubeshop-lkc et se compose de :

Frontend (shop-web) : Serveur Nginx servant du contenu statique. Il est configuré via une ConfigMap pour le contenu HTML et une autre pour la configuration Nginx (port 8080). Il tourne avec 2 replicas pour la haute disponibilité.
Backend (shop-api) : Une API (serveur HTTP Python dans le chart Helm) qui gère la logique métier. Elle récupère ses configurations (messages, flags) via une ConfigMap montée en volume dans /app.
Base de données (shop-db) : Instance PostgreSQL 16. La persistance est assurée par un PVC de 1Gi monté sur /var/lib/postgresql/data. Les identifiants sont stockés de manière sécurisée dans un Secret.
Ingress : Un point d'entrée unique via l'hôte kubeshop.local.
🌐 Focus sur l'Ingress (Routage)

L'Ingress agit comme un Reverse Proxy à l'entrée du cluster. Il utilise des expressions régulières pour diriger le trafic vers le bon service :

Route /api : Redirige vers kubeshop-api-svc. L'annotation rewrite-target: /\$2 permet de supprimer le préfixe /api avant d'envoyer la requête au backend, évitant ainsi des erreurs 404 si l'API ne s'attend pas à ce préfixe.
Route / : Redirige tout le reste du trafic vers le frontend kubeshop-web-svc.

Test de l'Ingress :

# Vérifier la configuration de l'Ingress
kubectl -n kubeshop-lkc describe ingress kubeshop-ingress

# Tester l'accès (si kubeshop.local est configuré dans votre /etc/hosts)
curl -i http://kubeshop.local/      # Accès Web
curl -i http://kubeshop.local/api/  # Accès API

⚙️ Focus sur l'API (Configuration)

L'API (kubeshop-api) illustre la séparation entre le code et la configuration. Au lieu d'écrire les messages en dur, elle utilise une ConfigMap (kubeshop-api-config).

Montage en Volume : Dans ce projet, la ConfigMap n'est pas injectée en variables d'environnement mais montée comme un système de fichiers dans /app. Chaque clé de la ConfigMap devient un fichier (ex: /app/KB_MESSAGE).
Avantage : Cela permet de mettre à jour la configuration sans forcément redémarrer le pod (selon l'application).

Test de l'API :

# Vérifier le contenu de la configuration vue par le pod API
kubectl -n kubeshop-lkc exec -it deploy/kubeshop-api -- ls -la /app
kubectl -n kubeshop-lkc exec -it deploy/kubeshop-api -- cat /app/KB_MESSAGE

🔐 Focus sur le RBAC (Sécurité)

Le projet implémente le principe du moindre privilège via un ServiceAccount dédié.

ServiceAccount (shop-ops) : L'identité du processus.
Role (shop-ops-role) : Définit les permissions de lecture seule (get, list, watch) sur les pods, services et events.
RoleBinding : Lie l'identité aux permissions.

Test de sécurité :

# Vérifier qu'on peut lister les pods (Autorisé)
kubectl -n kubeshop-lkc auth can-i list pods --as=system:serviceaccount:kubeshop-lkc:shop-ops

# Vérifier qu'on NE PEUT PAS supprimer un pod (Interdit)
kubectl -n kubeshop-lkc auth can-i delete pods --as=system:serviceaccount:kubeshop-lkc:shop-ops

💾 Persistance et Nettoyage
Vérification du stockage :
kubectl -n kubeshop-lkc get pvc kubeshop-db-pvc

Désinstallation :
helm uninstall kubeshop
kubectl delete ns kubeshop-lkc
