

### Commandes essentielles

```bash
# État des pods
kubectl -n kubeshop-lkc get pods -o wide
kubectl -n kubeshop-lkc get pods --show-labels

# État des services et endpoints
kubectl -n kubeshop-lkc get svc -o wide
kubectl -n kubeshop-lkc get endpoints

# État des déploiements et replicas
kubectl -n kubeshop-lkc get deploy -o wide
kubectl -n kubeshop-lkc describe deploy

# État de l'Ingress
kubectl get ingress -A
kubectl -n kubeshop-lkc describe ingress kubeshop-ingress

# État des PersistentVolumeClaims et Volumes
kubectl -n kubeshop-lkc get pvc
kubectl get pv
```

## 2. Erreur 404/503 via Ingress — Diagnostic et résolution

L'Ingress retourne une erreur HTTP (404, 503) quand la requête ne trouve pas de backend valide.

### Cas d'étude : Erreur 503 pour `/api/`

```bash
$ curl -i -H "Host: kubeshop.local" http://192.168.49.2/api/
HTTP/1.1 503 Service Temporarily Unavailable
```

**Logs du controller ingress-nginx :**
```
W0109 13:20:35 controller.go:1232] Service "kubeshop-lkc/kubeshop-api-svc" does not have any active Endpoint.
```

### Étape 1 : Vérifier l'Ingress

```bash
$ kubectl -n kubeshop-lkc describe ingress kubeshop-ingress
Name:             kubeshop-ingress
Namespace:        kubeshop-lkc
Address:          192.168.49.2
Rules:
  Host             Path  Backends
  ----             ----  --------
  kubeshop.local   /     kubeshop-web-svc:80
                   /api/ kubeshop-api-svc:80
```


### Étape 2 : Vérifier les Endpoints

```bash
$ kubectl -n kubeshop-lkc get endpoints

NAME               ENDPOINTS                           AGE
kubeshop-api-svc   <none>                              15m  # ❌ VIDE !
kubeshop-db-svc    10.244.0.84:5432                    15m
kubeshop-web-svc   10.244.0.78:8080,10.244.0.79:8080   15m
```
Service selector ne match aucun pod

### Étape 3 : Vérifier le Service selector

```bash
$ kubectl -n kubeshop-lkc get svc kubeshop-api-svc -o yaml | grep -A3 selector:
selector:
  app: kubeshop-api
  component: api 
```

### Étape 4 : Vérifier les labels des pods

```bash
$ kubectl -n kubeshop-lkc get pods --show-labels

NAME                            READY   STATUS    LABELS
kubeshop-api-f69f75b7d-9g96b    1/1     Running   app=kubeshop-api  
kubeshop-api-f69f75b7d-k2q5t    1/1     Running   app=kubeshop-api  
```

le Service attend `component: api` mais les pods ne l'ont pas.

### Résolution

**Éditer le manifest :**

Fichier : `kubeshop/templates/34-api-deploy.yaml`

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: kubeshop-api
  namespace: kubeshop-lkc
spec:
  template:
    metadata:
      labels:
        app: kubeshop-api
        component: api  # ajout api
```

**Redéployer :**

```bash
cd /home/corentin/WORKSPACE/KUBERNETES/m1-k8s-lkc
helm upgrade --install kubeshop-lkc ./kubeshop
