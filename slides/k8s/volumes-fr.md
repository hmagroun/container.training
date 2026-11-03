# Volumes

* Les volumes sont des répertoires spéciaux montés à l'intérieur des conteneurs

* Les volumes peuvent répondre à de multiples cas d'usage :

  * partager des fichiers et des répertoires entre conteneurs s'exécutant sur la même machine

  * partager des fichiers et des répertoires entre les conteneurs et leur hôte

  * centraliser les informations de configuration dans Kubernetes et les exposer aux conteneurs

  * gérer les identifiants et les secrets pour les exposer de manière sécurisée aux conteneurs

  * stocker des données persistantes pour des services avec état (stateful)

  * accéder à des systèmes de stockage (tels que Ceph, EBS, NFS, Portworx et bien d'autres)

---

class: extra-details

## Kubernetes volumes vs. Docker volumes

* Les volumes Kubernetes et Docker sont très similaires

  (la [documentation de Kubernetes](https://kubernetes.io/docs/concepts/storage/volumes/) affirme le contraire...
  <br/>
  mais elle fait référence à Docker 1.7, sorti en 2015 !)

* Les volumes Docker permettent de partager des données entre conteneurs tournant sur le même hôte

* Les volumes Kubernetes permettent de partager des données entre conteneurs au sein d'un même Pod

* Les volumes Docker et Kubernetes permettent tous deux d'accéder à des systèmes de stockage externes

* Les volumes Kubernetes sont également utilisés pour exposer de la configuration et des secrets

* Docker dispose de concepts spécifiques pour la configuration et les secrets
  <br/>
  (mais sous le capot, l'implémentation technique reste semblable)

* Si vous n'êtes pas familier avec les volumes Docker, vous pouvez ignorer cette slide en toute sécurité !

---

## Volumes ≠ Persistent Volumes

* Les Volumes et les Persistent Volumes sont liés, mais fondamentalement différents !

* *Volumes* :

  * apparaissent dans les spécifications des Pods (nous le verrons dans les prochaines slides)

  * n'existent pas en tant que ressources d'API (**impossible** de faire `kubectl get volumes`)

* *Persistent Volumes* :

  * sont de véritables ressources d'API (**on peut** faire `kubectl get persistentvolumes`)

  * correspondent à des volumes concrets (par ex. sur un SAN, EBS, etc.)

  * ne peuvent pas être associés directement à un Pod ; ils nécessitent une Persistent Volume Claim

  * ne seront pas abordés plus en détail dans cette section

---

## Ajouter un volume à un Pod

* Nous allons partir du manifeste de Pod le plus simple possible

* Nous allons ajouter un volume à ce manifeste de Pod

* Nous allons monter ce volume dans un conteneur du Pod

* Par défaut, ce volume sera un `emptyDir`

  (un répertoire vide)

* Il va masquer ("shadow") le contenu du répertoire où il est monté

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

C'est un MVP ! (Minimum Viable Pod😉)

Il exécute un unique conteneur NGINX.

---

## Tester le pod de base

.lab[

- Créer le Pod :
  ```bash
  kubectl create -f ~/container.training/k8s/nginx-1-without-volume.yaml
  ```

<!-- ```bash kubectl wait pod/nginx-without-volume --for condition=ready ``` -->

- Récupérer son adresse IP :
  ```bash
  IPADDR=$(kubectl get pod nginx-without-volume -o jsonpath={.status.podIP})
  ```

- Envoyer une requête avec curl :
  ```bash
  curl $IPADDR
  ```

]

(Nous devrions voir la page "Welcome to NGINX".)

---

## Ajouter un volume

* Nous devons déclarer le volume à deux endroits distincts :

  * au niveau du Pod (pour déclarer le volume)

  * au niveau du conteneur (pour monter le volume)

* Nous allons déclarer un volume nommé `www`

* Aucun type n'est spécifié, il prendra donc la valeur par défaut `emptyDir`

  (comme son nom l'indique, il sera initialisé comme un répertoire vide à la création du pod)

* Dans ce pod, se trouve également un conteneur nommé `nginx`

* Ce conteneur monte le volume `www` sur le chemin `/usr/share/nginx/html/`

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
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: www
      mountPath: /usr/share/nginx/html/
```

---

## Tester le Pod avec un volume

.lab[

- Créer le Pod :
  ```bash
  kubectl create -f ~/container.training/k8s/nginx-2-with-volume.yaml
  ```

<!-- ```bash kubectl wait pod/nginx-with-volume --for condition=ready ``` -->

- Récupérer son adresse IP :
  ```bash
  IPADDR=$(kubectl get pod nginx-with-volume -o jsonpath={.status.podIP})
  ```

- Envoyer une requête avec curl :
  ```bash
  curl $IPADDR
  ```

]

(Nous devrions maintenant obtenir une page d'erreur "403 Forbidden".)

---

## Peupler le volume avec un autre conteneur

* Ajoutons un second conteneur dans le Pod

* Montons le volume dans *les deux* conteneurs

* Ce conteneur va peupler le volume avec des fichiers statiques

* NGINX servira ensuite ces fichiers statiques

* Pour alimenter le volume, nous allons cloner le dépôt Spoon-Knife

  * ce dépôt est accessible à https://github.com/octocat/Spoon-Knife

  * il est très populaire (plus de 100K étoiles !)

---

## Partager un volume entre deux conteneurs

.small[
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-git
spec:
  volumes:
  - name: www
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: www
      mountPath: /usr/share/nginx/html/
  - name: git
    image: alpine
    command: [ "sh", "-c", "apk add git && git clone https://github.com/octocat/Spoon-Knife /www" ]
    volumeMounts:
    - name: www
      mountPath: /www/
  restartPolicy: OnFailure
```
]

---

## Le partage de volume expliqué

* Nous avons ajouté un autre conteneur au pod

* Ce conteneur monte le volume `www` sur un chemin différent (`/www`)

* Il utilise l'image `alpine`

* À son démarrage, il installe `git` et clone le dépôt `octocat/Spoon-Knife`

  (ce dépôt contient un mini site web en HTML)

* En conséquence, NGINX sert désormais ce site web

---

## Tester le volume partagé

* Cette manipulation est sensible au temps !

* Nous devons récupérer l'adresse IP du Pod *dès sa création*

* Puis lui envoyer une requête *le plus rapidement possible*

.lab[

- Surveiller les pods (afin de pouvoir intercepter l'adresse IP du Pod) :
  ```bash
  kubectl get pods -o wide --watch
  ```

<!--
```wait NAME```
```tmux split-pane -v```
-->

]

---

## Le volume partagé en action

.lab[

- Créer le pod :
  ```bash
  kubectl create -f ~/container.training/k8s/nginx-3-with-git.yaml
  ```

<!--
```bash kubectl wait pod/nginx-with-git --for condition=initialized```
```bash IP=$(kubectl get pod nginx-with-git -o jsonpath={.status.podIP})```
-->

- Dès que son adresse IP apparaît, accédez-y :
  ```bash
  curl `$IP`
  ```

<!-- ```bash /bin/sleep 5``` -->

- Quelques secondes plus tard, l'état du pod va évoluer ; accédez-y à nouveau :
  ```bash
  curl `$IP`
  ```

]

La première fois, nous devrions observer une erreur "403 Forbidden".

La deuxième fois, nous devrions voir le fichier HTML provenant du dépôt Spoon-Knife.

---

## Explications

* Les deux conteneurs sont démarrés simultanément

* NGINX démarre extrêmement vite

  (il est capable de traiter des requêtes immédiatement)

* Mais à cet instant, le volume est encore vide

  (NGINX renvoie "403 Forbidden")

* L'autre conteneur installe git et clone le dépôt

  (cela prend un peu plus de temps)

* Une fois ce conteneur terminé, le volume contient le dépôt

  (NGINX sert alors le fichier HTML)

---

## Le diable est dans les détails

* La valeur par défaut de `restartPolicy` est `Always`

* Cela forcerait notre conteneur `git` à s'exécuter encore... et encore... et encore

  (avec un délai de temporisation exponentiel [exponential back-off], comme expliqué [dans la documentation](https://kubernetes.io/docs/concepts/workloads/pods/pod-lifecycle/#restart-policy))

* C'est pourquoi nous avons précisé `restartPolicy: OnFailure`

---

## Incohérences

* Il existe un court laps de temps durant lequel le site web n'est pas disponible

  (le conteneur `git` n'ayant pas encore terminé son travail)

* Avec un site web plus volumineux, nous pourrions obtenir des résultats incohérents

  (où seule une partie du contenu serait prête)

* Dans des applications en production, cela pourrait engendrer des erreurs applicatives

* Comment pouvons-nous éviter cela ?

---

## Init Containers

* Nous pouvons définir des conteneurs qui doivent s'exécuter *avant* les conteneurs principaux

* Ils seront exécutés de manière séquentielle

  (au lieu d'être exécutés en parallèle)

* Ils doivent tous se terminer avec succès avant que les conteneurs principaux ne démarrent

* C'est *exactement* ce dont nous avons besoin ici !

* Voyons cela en pratique

.footnote[Consultez la documentation des [Init Containers](https://kubernetes.io/docs/concepts/workloads/pods/init-containers/) pour tous les détails.]

---

## Définir des Init Containers

.small[
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-with-init
spec:
  volumes:
  - name: www
  containers:
  - name: nginx
    image: nginx
    volumeMounts:
    - name: www
      mountPath: /usr/share/nginx/html/
  initContainers:
  - name: git
    image: alpine
    command: [ "sh", "-c", "apk add git && git clone https://github.com/octocat/Spoon-Knife /www" ]
    volumeMounts:
    - name: www
      mountPath: /www/
```
]

---

## Tester l'Init Container

.lab[

- Créer le pod :
  ```bash
  kubectl create -f ~/container.training/k8s/nginx-4-with-init.yaml
  ```

- Tenter d'envoyer des requêtes HTTP dès que le pod démarre

<!--
```key ^D```
```key ^C```
-->

]

* Cette fois, au lieu d'un "403 Forbidden", nous obtenons un "connection refused"

* NGINX ne démarre pas tant que le conteneur git n'a pas achevé sa tâche

* Nous n'obtenons jamais d'état incohérent

  (un conteneur "à moitié prêt")

---

## Cycle de vie des volumes

* Le cycle de vie d'un volume est étroitement lié au cycle de vie du pod

* Cela signifie qu'un volume est créé lors de la création du pod

* Cela concerne principalement les volumes `emptyDir`

  (les autres volumes, comme le stockage distant, ne sont pas "créés" mais plutôt "attachés")

* Un volume survit aux redémarrages de conteneurs

* Un volume est détruit (ou, pour le stockage distant, détaché) lors de la suppression du pod

---

## Autres usages des Init Containers

* Charger du contenu, des jeux de données...

* Générer de la configuration (ou des certificats)

* Exécuter des migrations de bases de données

* Attendre que d'autres services soient opérationnels

  (pour éviter une vague d'erreurs de connexion dans le conteneur principal)

* etc.

---

## Init containers vs sidecars

* Les Init Containers s'exécutent *avant* le ou les conteneur(s) principal(aux)

* Les sidecars s'exécutent *en parallèle* du ou des conteneur(s) principal(aux)

* Quelle est la différence entre un sidecar et un "conteneur principal" ?

--

  * le sidecar peut devoir démarrer *avant* le ou les conteneur(s) principal(aux)
    <br/>(par ex. s'il fournit un service de connectivité de type "ambassadeur")

  * le sidecar peut devoir s'arrêter *après* le ou les conteneur(s) principal(aux)
    <br/>(idem)

  * le sidecar peut devoir être arrêté automatiquement lorsque le ou les conteneur(s) principal(aux) se terminent
    <br/>(par ex. pour des batch jobs)

* Kubernetes intègre une prise en charge dédiée pour les sidecars !

---

## Sidecars

* Introduit comme fonctionnalité alpha dans K8S 1.28 ; passé en statut GA dans K8S 1.33

* Un sidecar est un `initContainer` doté d'une `restartPolicy: Always`

* Les sidecars démarrent dans l'ordre défini par la liste des `initContainers`

* Ils peuvent disposer de healthchecks

* Kubernetes n'attend pas qu'ils se terminent

  (ils s'exécutent de façon asynchrone)

* Lorsque tous les conteneurs principaux sont terminés, les sidecars sont arrêtés automatiquement

  (ils n'empêchent pas les jobs de se terminer !)

???

:EN:- Sharing data between containers with volumes
:EN:- When and how to use Init Containers

:FR:- Partager des données grâce aux volumes
:FR:- Quand et comment utiliser un *Init Container*