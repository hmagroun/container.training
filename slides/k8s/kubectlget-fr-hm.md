# Premier contact avec `kubectl`

- `kubectl` est (presque) le seul outil dont nous aurons besoin pour interagir avec Kubernetes

- C'est un outil CLI très riche interfaçant l'API Kubernetes

  (Tout ce que vous pouvez réaliser avec `kubectl`, vous pouvez l'exécuter directement via l'API)

- Sur nos machines, un fichier `~/.kube/config` contient :

  - l'adresse de l'API Kubernetes

  - le chemin vers nos certificats TLS servant à l'authentification

- Vous pouvez également utiliser le flag `--kubeconfig` pour transmettre un fichier de configuration

- Ou directement `--server`, `--user`, etc.

- `kubectl` peut se prononcer "Cube C T L", "Cube cuttle", "Cube cuddle"...

---

class: extra-details

## `kubectl` est le nouveau SSH

- On commence souvent par administrer des serveurs avec SSH

  (installation de paquets, troubleshooting...)

- À l'échelle, cela devient fastidieux, répétitif et source d'erreurs

- À la place, on utilise des outils de gestion de configuration, de centralisation de logs, etc.

- Dans bien des cas, SSH reste indispensable :

  - comme méthode d'accès sous-jacente (ex. Ansible)

  - pour débugger des scénarios complexes

  - pour inspecter et tester l'état du système

---

class: extra-details

## Le parallèle avec `kubectl`

- On commence souvent par administrer des clusters Kubernetes avec `kubectl`

  (déploiement d'applications, troubleshooting...)

- À l'échelle (avec de multiples applications ou clusters), cela devient fastidieux, répétitif et source d'erreurs

- À la place, on exploite des pipelines automatisés, des outils d'observabilité, etc.

- Dans bien des cas, `kubectl` reste incontournable :

  - pour débugger des scénarios complexes

  - pour inspecter et tester l'état du cluster

- L'API Kubernetes demeure systématiquement la méthode d'accès sous-jacente

---

## `kubectl get`

- Observons nos ressources `Node` à l'aide de `kubectl get` !

.lab[

- Examiner la composition de notre cluster :
  ```bash
  kubectl get node
  ```

- Ces commandes sont équivalentes :
  ```bash
  kubectl get no
  kubectl get node
  kubectl get nodes
  ```

]

---

## Obtenir une sortie exploitable par machine

- `kubectl get` permet de générer du format JSON, YAML, ou d'appliquer un formatage direct

.lab[

- Obtenir davantage d'informations sur les nodes :
  ```bash
  kubectl get nodes -o wide
  ```

- Afficher la sortie au format YAML :
  ```bash
  kubectl get no -o yaml
  ```
  Notez ce `kind: List` à la fin : il s'agit du type retourné pour notre résultat !

]

---

## Exploiter la puissance de `kubectl` et `jq`

- Il est extrêmement simple de concevoir des rapports personnalisés

.lab[

- Afficher la capacité de l'ensemble de nos nodes sous forme de flux d'objets JSON :
  ```bash
    kubectl get nodes -o json |
            jq ".items[] | {name:.metadata.name} + .status.capacity"
  ```

]

---

class: extra-details

## Explorer les types et les définitions

- Nous pouvons lister tous les types de ressources disponibles via la commande `kubectl api-resources`
  <br/>
  (Avec Kubernetes 1.10 et versions antérieures, cette commande était `kubectl get`)

- Nous pouvons inspecter la définition d'un type de ressource avec :
  ```bash
  kubectl explain type
  ```

- Nous pouvons visualiser la définition d'un champ précis d'une ressource, par exemple :
  ```bash
  kubectl explain node.spec
  ```

- Ou récupérer la définition complète récursive de tous les champs et sous-champs :
  ```bash
  kubectl explain node --recursive
  ```

---

class: extra-details

## Introspection vs documentation

- Nous pouvons accéder aux mêmes informations en consultant la [documentation de l'API](https://kubernetes.io/docs/reference/#api-reference)

- La documentation de l'API est souvent plus confortable à lire, mais :

  - elle ne présente pas les types personnalisés (comme les Custom Resource Definitions)

  - nous devons nous assurer de consulter la version exacte correspondante

- `kubectl api-resources` et `kubectl explain` procèdent par *introspection*

  (elles interrogent l'API server et extraient les définitions de types exactes)

---

## Noms des types

- Les noms de ressources les plus répandus possèdent trois variantes :

  - singulier (ex. `node`, `service`, `deployment`)

  - pluriel (ex. `nodes`, `services`, `deployments`)

  - court (ex. `no`, `svc`, `deploy`)

- Certaines ressources ne possèdent pas de nom court

- `Endpoints` s'emploie uniquement au pluriel

  (car une unique ressource `Endpoints` constitue en réalité une liste d'endpoints)

---

## Visualiser les détails

- Nous pouvons utiliser `kubectl get -o yaml` pour afficher l'ensemble des détails disponibles

- Cependant, la sortie YAML est souvent à la fois trop dense et insuffisante

- Par exemple, `kubectl get node node1 -o yaml` produit :

  - trop d'informations (ex. la liste complète des images présentes sur ce node)

  - pas assez d'informations (ex. n'indique pas les pods s'exécutant sur ce node)

  - un format difficilement lisible pour un opérateur humain

- Pour une vue d'ensemble exhaustive, nous pouvons utiliser `kubectl describe` à la place

---

## `kubectl describe`

- `kubectl describe` prend en paramètre un type de ressource et (optionnellement) le nom d'une ressource

- Il est possible de spécifier un *préfixe* de nom de ressource

  (tous les objets correspondants seront alors listés)

- `kubectl describe` va collecter des informations complémentaires agrégées sur la ressource

.lab[

- Inspecter les informations détaillées pour `node1` à l'aide de l'une de ces commandes :
  ```bash
  kubectl describe node/node1
  kubectl describe node node1
  ```

]

(Vous devriez repérer plusieurs pods du control plane.)

---

## Lister les conteneurs en cours d'exécution

- Les conteneurs sont manipulés à travers des *pods*

- Un pod est un ensemble de conteneurs :

 - exécutés ensemble (sur le même node)

 - partageant des ressources (RAM, CPU ; mais aussi le réseau et les volumes)

.lab[

- Lister les pods présents sur notre cluster :
  ```bash
  kubectl get pods
  ```

]

--

*Où sont passés les pods que nous avons vus il y a un instant ?!?*

---

## Namespaces

- Les namespaces nous permettent de compartimenter et isoler les ressources

.lab[

- Lister les namespaces de notre cluster via l'une de ces commandes :
  ```bash
  kubectl get namespaces
  kubectl get namespace
  kubectl get ns
  ```

]

--

*Attendez un peu... Cet élément `kube-system` a l'air bien suspect.*

*En fait, je suis certain qu'il est apparu plus tôt lorsque nous avons exécuté :*

`kubectl describe node node1`

---

## Accéder aux namespaces

- Par défaut, `kubectl` interroge le namespace `default`

- Nous pouvons inspecter les ressources à travers tous les namespaces grâce à `--all-namespaces`

.lab[

- Lister les pods situés dans l'ensemble des namespaces :
  ```bash
  kubectl get pods --all-namespaces
  ```

- Depuis Kubernetes 1.14, nous pouvons utiliser `-A` comme alias court :
  ```bash
  kubectl get pods -A
  ```

]

*Voici enfin nos pods système !*

---

## Que représentent tous ces pods du control plane ?

- `etcd` est notre serveur etcd

- `kube-apiserver` est l'API server

- `kube-controller-manager` et `kube-scheduler` sont les autres composants clés du control plane

- `coredns` assure la service discovery basée sur DNS ([remplaçant kube-dns depuis la version 1.11](https://kubernetes.io/blog/2018/07/10/coredns-ga-for-kubernetes-cluster-dns/))

- `kube-proxy` est le composant (par node) gérant l'accès réseau aux services internes

- `cilium` est le composant (par node) assurant la gestion de l'overlay network

- la colonne `READY` indique le nombre de conteneurs prêts dans chaque pod

  (ce qui sera souvent égal à 1 pour la plupart des pods !)

---

## Cibler un autre namespace

- Nous pouvons cibler un namespace spécifique (différent de `default`)

.lab[

- Lister uniquement les pods situés dans le namespace `kube-system` :
  ```bash
  kubectl get pods --namespace=kube-system
  kubectl get pods -n kube-system
  ```

]

---

## Les namespaces et les autres commandes `kubectl`

- On peut employer `-n`/`--namespace` avec pratiquement toutes les commandes `kubectl`

- Exemple :

  - `kubectl create --namespace=X` pour créer une ressource dans le namespace X

- On peut utiliser `-A`/`--all-namespaces` avec la plupart des commandes manipulant des collections d'objets

- Exemples :

  - `kubectl delete` permet de supprimer des ressources réparties sur plusieurs namespaces

  - `kubectl label` permet d'ajouter/supprimer/modifier des labels à travers de multiples namespaces

---

class: extra-details

## Qu'en est-il de `kube-public` ?

.lab[

- Lister les pods situés dans le namespace `kube-public` :
  ```bash
  kubectl -n kube-public get pods
  ```

]

Rien !

`kube-public` est généré automatiquement par kubeadm et [utilisé pour l'initialisation de la sécurité (security bootstrapping)](https://kubernetes.io/blog/2017/01/stronger-foundation-for-creating-and-managing-kubernetes-clusters).

---

class: extra-details

## Explorer `kube-public`

- Le seul objet notable dans `kube-public` est un ConfigMap nommé `cluster-info`

.lab[

- Lister les objets ConfigMap :
  ```bash
  kubectl -n kube-public get configmaps
  ```

- Inspecter `cluster-info` :
  ```bash
  kubectl -n kube-public get configmap cluster-info -o yaml
  ```

]

Remarquez l'URI `selfLink` : `/api/v1/namespaces/kube-public/configmaps/cluster-info`

Nous pouvons l'exploiter !

---

class: extra-details

## Accéder à `cluster-info`

- Précédemment, lors d'une tentative d'accès à l'API server, nous avons reçu un message `Forbidden`

- Mais `cluster-info` est accessible en lecture par tout le monde (même sans authentification)

.lab[

- Récupérer `cluster-info` :
  ```bash
  curl -k https://10.96.0.1/api/v1/namespaces/kube-public/configmaps/cluster-info
  ```

]

- Nous avons pu accéder à `cluster-info` (sans authentification)

- Il contient un fichier `kubeconfig`

---

class: extra-details

## Récupérer le `kubeconfig`

- Nous pouvons facilement extraire le fichier `kubeconfig` de ce ConfigMap

.lab[

- Afficher le contenu de `kubeconfig` :
  ```bash
    curl -sk https://10.96.0.1/api/v1/namespaces/kube-public/configmaps/cluster-info \
         | jq -r .data.kubeconfig
  ```

]

- Ce fichier contient l'adresse canonique de l'API server, ainsi que la clé publique de la CA

- Ce fichier *ne contient pas* de clés clients ou de tokens

- Ce ne sont pas des données sensibles, mais elles permettent d'établir la chaîne de confiance

---

class: extra-details

## Qu'en est-il de `kube-node-lease` ?

- Depuis Kubernetes 1.14, il existe un namespace `kube-node-lease`

  (ou dès Kubernetes 1.13 si le feature gate NodeLease est activé)

- Ce namespace contient un objet Lease par node

- Les *node leases* constituent une méthode optimisée pour implémenter les heartbeats des nodes

  (c'est-à-dire l'envoi régulier d'un signal au control plane indiquant "je suis actif !")

- Pour plus de détails, consultez le [KEP Efficient Node Heartbeats](https://github.com/kubernetes/enhancements/blob/master/keps/sig-node/589-efficient-node-heartbeats/README.md) ou la [documentation du node controller](https://kubernetes.io/docs/concepts/architecture/nodes/#node-controller)

---

## Services

- Un *service* est un endpoint stable permettant de se connecter à "quelque chose"

  (Dans la proposition initiale de Kubernetes, ils étaient appelés "portals")

.lab[

- Lister les services de notre cluster via l'une de ces commandes :
  ```bash
  kubectl get services
  kubectl get svc
  ```

]

--

Il existe déjà un service sur notre cluster : l'API Kubernetes elle-même.

---

## Services de type ClusterIP

- Un service `ClusterIP` est interne, joignable uniquement depuis l'intérieur du cluster

- C'est très utile pour réaliser de l'introspection depuis l'intérieur des conteneurs

.lab[

- Tenter de se connecter à l'API :
  ```bash
  curl -k https://`10.96.0.1`
  ```

  - `-k` sert à ignorer la vérification du certificat

  - Pensez à remplacer 10.96.0.1 par la valeur CLUSTER-IP retournée par `kubectl get svc`

]

La commande ci-dessus devrait soit échouer par time out, soit renvoyer une erreur d'authentification. Pourquoi ?

---

## Time out

- Les connexions vers les services ClusterIP ne fonctionnent que *depuis l'intérieur du cluster*

- Si nous sommes à l'extérieur du cluster, la commande `curl` finira probablement en time out

  (Parce que l'adresse IP, ex. 10.96.0.1, n'est pas routée en dehors du cluster)

- C'est le cas sur la grande majorité des clusters Kubernetes en production

- Pour tester la connexion depuis l'intérieur du cluster, nous pouvons utiliser [shpod](https://github.com/jpetazzo/shpod)

---

## Erreur d'authentification

Voici ce que l'on devrait observer en se connectant depuis l'intérieur du cluster :
```json
$ curl -k https://10.96.0.1
{
  "kind": "Status",
  "apiVersion": "v1",
  "metadata": {

  },
  "status": "Failure",
  "message": "forbidden: User \"system:anonymous\" cannot get path \"/\"",
  "reason": "Forbidden",
  "details": {

  },
  "code": 403
}
```

---

## Explications

- On constate la présence des champs `kind`, `apiVersion`, `metadata`

- Ce sont les attributs caractéristiques d'une réponse de l'API Kubernetes

- Car nous communiquons *bien* directement avec l'API Kubernetes

- L'API Kubernetes renvoie "Forbidden"

  (car elle exige une authentification valide)

- L'API Kubernetes est joignable depuis l'intérieur du cluster

  (de nombreuses applications s'interfaçant avec Kubernetes exploitent ce mécanisme)

---

## Intégration DNS

- Chaque service se voit attribuer un enregistrement DNS dédié

- Le résolveur DNS de Kubernetes est accessible *depuis l'intérieur des pods*

  (et parfois depuis les nodes, selon la configuration du cluster)

- Le code applicatif s'exécutant dans les pods peut se connecter aux services en utilisant directement leur nom

  (ex. https://kubernetes/...)

???

:FR:- Se familiariser avec kubectl