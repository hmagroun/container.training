

# Lab Kubernetes : StatefulSet vs Deployment (Identité, Ordre et Stockage)

### Objectifs du Lab

1. **Partie 1 :** Démonstration de l'identité et de l'ordre de déploiement/suppression (Pods interchangeables vs Pods ordonnés).
2. **Partie 2 :** Gestion du stockage persistant (Volume unique partagé vs Volumes dédiés par Pod via `volumeClaimTemplates`).

---

## Partie 1 : Ordre de démarrage/arrêt et identité des Pods

Dans un **Deployment**, les Pods sont interchangeables (stateless) et reçoivent des noms aléatoires. Dans un **StatefulSet**, chaque Pod possède un nom fixe et prévisible (`<nom-statefulset>-0`, `<nom-statefulset>-1`, etc.) et le démarrage/arrêt suit un ordre séquentiel strict.

### 1. Test du comportement d'un Deployment

Créez le fichier `deploy-demo.yaml` :

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: deploy-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: deploy-app
  template:
    metadata:
      labels:
        app: deploy-app
    spec:
      containers:
      - name: nginx
        image: nginx:alpine

```

Appliquez et observez le déploiement :

```bash
kubectl apply -f deploy-demo.yaml
kubectl get pods -l app=deploy-app -w

```

* **Observation :** Les 3 Pods sont créés en parallèle avec des suffixes aléatoires (ex: `deploy-app-5d86b-abcde`, `deploy-app-5d86b-xyz12`).

Supprimez le Deployment :

```bash
kubectl delete -f deploy-demo.yaml
kubectl get pods -l app=deploy-app -w

```

* **Observation :** Tous les Pods sont détruits simultanément sans ordre précis.

---

### 2. Test du comportement d'un StatefulSet

Pour fonctionner correctement, un StatefulSet nécessite d'abord la création d'un **Headless Service** (`clusterIP: None`) pour assurer le nommage réseau individuel.

Créez le fichier `statefulset-demo.yaml` :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
  labels:
    app: stateful-app
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: stateful-app
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: stateful-app
spec:
  serviceName: "nginx-service"
  replicas: 3
  selector:
    matchLabels:
      app: stateful-app
  template:
    metadata:
      labels:
        app: stateful-app
    spec:
      containers:
      - name: nginx
        image: nginx:alpine

```

Appliquez et observez attentivement l'ordre de démarrage dans un second terminal :

```bash
kubectl apply -f statefulset-demo.yaml
kubectl get pods -l app=stateful-app -w

```

* **Observation du démarrage :**
1. `stateful-app-0` est créé en premier et passe à l'état `Running`.
2. `stateful-app-1` est créé seulement après que le précédent est prêt.
3. `stateful-app-2` est créé en dernier.



Supprimez le StatefulSet et observez l'ordre d'arrêt :

```bash
kubectl delete -f statefulset-demo.yaml
kubectl get pods -l app=stateful-app -w

```

* **Observation de l'arrêt (Ordre inverse strict) :**
1. `stateful-app-2` est arrêté en premier.
2. Puis `stateful-app-1`.
3. Enfin `stateful-app-0`.



---

## Partie 2 : Gestion du stockage persistant (Volumes partagés vs VolumeClaimTemplates)

### 1. Limite du Deployment avec un volume persistant

Si vous attachez un PVC classique à un Deployment à 3 répliques (avec `ReadWriteOnce`), soit les répliques échouent car elles ne peuvent pas accéder au même disque distant simultanément, soit toutes les répliques pointent exactement vers le même dossier et partagent les mêmes fichiers. Il n'y a **pas de stockage individuel** par Pod.

---

### 2. Le StatefulSet et les `volumeClaimTemplates`

Avec un StatefulSet, on utilise un modèle de réclamation (`volumeClaimTemplates`). Kubernetes va générer automatiquement **un PVC dynamique dédié pour chaque Pod** (`storage-stateful-app-0`, `storage-stateful-app-1`, etc.).

Créez le fichier `stateful-storage.yaml` :

```yaml
apiVersion: v1
kind: Service
metadata:
  name: web-service
spec:
  ports:
  - port: 80
    name: web
  clusterIP: None
  selector:
    app: web-store
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: web-store
spec:
  serviceName: "web-service"
  replicas: 3
  selector:
    matchLabels:
      app: web-store
  template:
    metadata:
      labels:
        app: web-store
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        volumeMounts:
        - name: storage
          mountPath: /usr/share/nginx/html
  volumeClaimTemplates:
  - metadata:
      name: storage
    spec:
      accessModes: [ "ReadWriteOnce" ]
      resources:
        requests:
          storage: 100Mi

```

Appliquez le fichier :

```bash
kubectl apply -f stateful-storage.yaml

```

#### Étape A : Inspection des volumes créés

Une fois les Pods prêts, vérifiez les PVC générés automatiquement :

```bash
kubectl get pvc,pv

```

* **Observation :** 3 PVC distincts ont été instanciés dynamiquement : `storage-web-store-0`, `storage-web-store-1`, et `storage-web-store-2`.

#### Étape B : Écriture de données personnalisées dans chaque Pod

Écrivez une donnée unique dans le volume propre à chaque réplique :

```bash
kubectl exec web-store-0 -- sh -c "echo 'Donnees du Pod 0' > /usr/share/nginx/html/index.html"
kubectl exec web-store-1 -- sh -c "echo 'Donnees du Pod 1' > /usr/share/nginx/html/index.html"
kubectl exec web-store-2 -- sh -c "echo 'Donnees du Pod 2' > /usr/share/nginx/html/index.html"

```

Vérifiez que chaque Pod lit bien son propre fichier :

```bash
kubectl exec web-store-0 -- cat /usr/share/nginx/html/index.html
kubectl exec web-store-1 -- cat /usr/share/nginx/html/index.html
kubectl exec web-store-2 -- cat /usr/share/nginx/html/index.html

```

#### Étape C : Test de Scale-Down puis Scale-Up (Persistance de l'affectation)

Réduisez le nombre de répliques à 1 (Scale-Down) :

```bash
kubectl scale statefulset web-store --replicas=1

```

Vérifiez les Pods et les PVC :

```bash
kubectl get pods -l app=web-store
kubectl get pvc

```

* **Observation importante :** Les Pods `web-store-1` et `web-store-2` ont été supprimés, **mais leurs PVC `storage-web-store-1` et `storage-web-store-2` restent intacts** dans le cluster.

Remontez à 3 répliques (Scale-Up) :

```bash
kubectl scale statefulset web-store --replicas=3

```

Attendez que les Pods redémarrent et contrôlez le contenu des fichiers :

```bash
kubectl exec web-store-1 -- cat /usr/share/nginx/html/index.html
kubectl exec web-store-2 -- cat /usr/share/nginx/html/index.html

```

* **Résultat :** `web-store-1` récupère exactement le PVC `storage-web-store-1` et relit `Donnees du Pod 1`. L'affectation entre l'index du Pod et son stockage dédié a été conservée sans aucune perte.

---

## Partie 3 : Nettoyage de l'environnement

```bash
kubectl delete -f stateful-storage.yaml
kubectl delete pvc -l app=web-store

```

*(Remarque : la suppression d'un StatefulSet ne détruit pas automatiquement les PVC générés par les `volumeClaimTemplates` afin d'éviter les pertes de données accidentelles en production. Il faut les supprimer explicitement).*