# Volumes

* Les *volumes* sont des répertoires spéciaux montés dans les *containers*.

* Les *volumes* peuvent avoir plusieurs objectifs :

  * partager des fichiers et des répertoires entre des *containers* s'exécutant sur la même machine

  * partager des fichiers et des répertoires entre les *containers* et leur *host*

  * centraliser les informations de configuration dans Kubernetes et les exposer aux *containers*

  * gérer les *credentials* et les *secrets* et les exposer en toute sécurité aux *containers*

  * stocker des données persistantes pour des *stateful services*

  * accéder à des systèmes de *storage* (comme Ceph, EBS, NFS, Portworx et bien d'autres)

---

## Volumes ≠ Persistent Volumes

* Les *Volumes* et les *Persistent Volumes* sont liés, mais très différents !

* *Volumes* :

  * apparaissent dans les spécifications des *Pods*

  * n'existent pas en tant que ressources API (**impossible** de faire `kubectl get volumes`)

* *Persistent Volumes* :

  * sont des ressources API (**possible** de faire `kubectl get persistentvolumes`)

  * correspondent à des volumes concrets (par ex. sur un SAN, EBS, etc.)

  * ne peuvent pas être associés directement à un *Pod* ; mais via une *Persistent Volume Claim*

  * ne seront pas abordés plus en détail dans cette section

---

## Ajouter un volume à un Pod

* Nous allons commencer avec le *manifest* de *Pod* le plus simple possible.

* Nous allons ajouter un *volume* à ce *manifest* de *Pod*.

* Nous allons monter ce *volume* dans un *container* au sein du *Pod*.

* Par défaut, ce *volume* sera un `emptyDir` (un répertoire vide).

* Il va masquer (*shadow*) le répertoire dans lequel il est monté.

---

## Notre Pod de base

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-without-volume
spec:
  containers:
  - name: nginx
    image: nginx
```

C'est un MVP ! (Minimum Viable Pod 😉)

Il exécute un seul *container* NGINX.

---

## Ajouter un volume

* Nous devons ajouter le *volume* à deux endroits :

  * au niveau du *Pod* (pour déclarer le *volume*)

  * au niveau du *container* (pour monter le *volume*)

* Nous allons déclarer un *volume* nommé `www`.

* Aucun type n'est spécifié, il sera donc par défaut un `emptyDir` (comme son nom l'indique, il sera initialisé comme un répertoire vide lors de la création du *pod*).

* Dans ce *pod*, il y a aussi un *container* nommé `nginx`.

* Ce *container* monte le *volume* `www` sur le chemin `/usr/share/nginx/html/`.

---

## Le Pod avec un volume

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-volume
spec:
  volumes:
  - name: www
    emptyDir: {}
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: www
      mountPath: /usr/share/nginx/html/
```

---

## Cycle de vie d'un volume

* Le *lifecycle* d'un *volume* est lié au *lifecycle* du *pod*.

* Cela signifie qu'un *volume* est créé lorsque le *pod* est créé.

* C'est principalement pertinent pour les *volumes* `emptyDir` (les autres *volumes*, comme le *remote storage*, ne sont pas « créés » mais plutôt « attachés »).

* Un *volume* survit aux redémarrages de *containers*.

* Un *volume* est détruit (ou, pour le *remote storage*, détaché) lorsque le *pod* est détruit.