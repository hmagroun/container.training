# Concepts Kubernetes

- Kubernetes est un système de gestion de conteneurs

- Il exécute et gère des applications conteneurisées sur un cluster

--

- Qu'est-ce que cela signifie concrètement ?

---

## Que pouvons-nous faire avec Kubernetes ?

- Imaginons une application e-commerce 3-tiers :

  - frontend web

  - backend d'API

  - base de données

- Nous avons packagé nos composants frontend et backend sous forme d'images

  (par ex. avec des Dockerfiles et `docker build`)

- Ils s'exécutent avec succès dans un environnement local

  (par ex. avec Docker Compose)

- Voyons comment déployer notre application sur Kubernetes !

---

## Kubernetes, niveau 1

--

- Laisser notre base de données en dehors de Kubernetes (parce que les bases de données font peur 🥺)

--

- Déployer un cluster Kubernetes managé (cloud ou via des [services professionnels][enix-k8s-expert])

--

- Démarrer 5 conteneurs utilisant l'image `atseashop/api:v1.3`

--

- Positionner un load balancer interne devant ces conteneurs

--

- Démarrer 10 conteneurs utilisant l'image `atseashop/webfront:v1.3`

--

- Positionner un load balancer public devant ces conteneurs

--

- C'est le Black Friday (ou Noël), le trafic explose : faire grossir notre cluster et ajouter des conteneurs

--

- Nouvelle release ! Remplacer les conteneurs avec la nouvelle image `atseashop/webfront:v1.4`

--

- Continuer à traiter les requêtes pendant l'upgrade ; mettre à jour nos conteneurs un par un

[enix-k8s-expert]: https://enix.io/en/kubernetes-expert/

---

## Kubernetes, niveau 2

- Déployer un environnement de pré-production

  (toujours en exploitant notre base de données externe, pour le moment)

- Gestion des ressources et scheduling

  (réserver CPU/RAM pour les conteneurs ; contraintes de placement ; priorités)

- Autoscaling

  (simple sur le CPU ; plus complexe sur d'autres métriques)

- Stratégies de rollout avancées

  (blue/green deployment, canary deployment)

--

.footnote[
À la section suivante : cage à canari avec une bouteille d'oxygène, conçue pour maintenir le canari en vie.
<br/>
(Voir https://post.lurk.org/@zilog/109632335293371919 pour les détails.)
]

---

class: pic

![Canary cage](images/canary-cage.jpg)

---

## Kubernetes, niveau 3

- Exécuter des bases de données de staging sur le cluster

  (sans réplication, sans sauvegardes, sans scaling)

- Déploiement automatique ou semi-automatique des feature branches

  (chacune avec sa propre base de données)

- Contrôle d'accès granulaire

  (définir *qui* peut faire *quoi* sur *quelles* ressources)

- Batch jobs

  (ponctuels ; parallèles ; exécution périodique type cron)

- Packager des applications avec, par exemple, des charts Helm

---

## Kubernetes, niveau 4

- Services stateful avec persistance, réplication et sauvegardes

  (bases de données, message queues, etc.)

- Automatiser des tâches complexes à l'aide d'*operators*

  (par ex. réplication de base de données, failover, etc.)

- Combiner les deux aspects précédents grâce à des operators de bases de données comme [CloudNativePG][cnpg]

  (en savoir plus sur les database operators : [FR][pirates-video-fr], [EN][pirates-video-en])

- Exploiter du stockage avancé avec par exemple des volumes ZFS locaux

  (en savoir plus sur ZFS et les bases de données sur k8s : [FR][zfs-video-fr], [EN][zfs-video-en])

- Déployer et gérer des clusters on-premises

[cnpg]: https://cloudnative-pg.io/
[pirates-video-fr]: https://www.youtube.com/watch?v=d_ka7PlWo1I
[pirates-video-en]: https://www.youtube.com/watch?v=ojUdBjbiKWk&t=5s
[zfs-video-fr]: https://www.youtube.com/watch?v=XN9YL93f8tI
[zfs-video-en]: https://www.youtube.com/watch?v=3sJIYiDnod4

---

## Kubernetes, niveau 5

- Déployer et administrer des clusters à grande échelle

  (des centaines de clusters, des milliers de nodes...)

- Écrire des operators personnalisés

- Déploiements hybrides

---

## Remarque préalable

Les niveaux présentés dans les sections précédentes ne sont pas nécessairement linéaires.

Ils ne sont pas non plus exhaustifs (nous n'avons pas mentionné, par exemple, l'observabilité et l'alerting).

---

## Architecture de Kubernetes

---

class: pic

![haha only kidding](images/k8s-arch1.png)

---

## Architecture de Kubernetes

- Ha ha ha ha

- Très bien, c'était juste pour vous faire peur, c'est bien plus simple que cela ❤️

---

class: pic

![that one is more like the real thing](images/k8s-arch2.png)

---

## Crédits

- Le premier schéma représente un cluster Kubernetes avec du stockage adossé à de l'iSCSI multi-path

  (Avec l'aimable autorisation de [Yongbok Kim](https://www.yongbok.net/blog/))

- Le second est une représentation simplifiée d'un cluster Kubernetes

  (Avec l'aimable autorisation d' [Imesh Gunaratne](https://medium.com/containermind/a-reference-architecture-for-deploying-wso2-middleware-on-kubernetes-d4dee7601e8e))

---

## Architecture de Kubernetes : les nodes

- Les nodes exécutant nos conteneurs hébergent un ensemble de services :

  - un container Engine (généralement Docker)

  - kubelet (le "node agent")

  - kube-proxy (un composant réseau indispensable, mais non suffisant à lui seul)

- Les nodes étaient autrefois appelés "minions"

  (Vous pouvez rencontrer ce terme dans des articles ou documentations plus anciens)

---

## Architecture de Kubernetes : le control plane

- La logique interne de Kubernetes (son "cerveau") est un ensemble de services :

  - l'API server (notre point d'entrée unique vers l'ensemble du système !)

  - les composants cœur comme le scheduler et le controller manager

  - `etcd` (un key/value store hautement disponible ; la "base de données" de Kubernetes)

- Ensemble, ces services constituent le control plane de notre cluster

- Le control plane est également désigné sous le terme de "master"

---

class: pic

![One of the best Kubernetes architecture diagrams available](images/k8s-arch4-thanks-luxas.png)

---

class: extra-details

## Exécuter le control plane sur des nodes dédiés

- Il est courant de réserver un node dédié pour le control plane

  (À l'exception des clusters de développement single-node, comme avec minikube)

- Ce node est alors qualifié de "master"

  (Ce terme est source d'ambiguïté : le "master" désigne-t-il le node physique ou l'ensemble du control plane ?)

- On interdit aux applications standards de s'exécuter sur ce node

  (En utilisant un mécanisme appelé ["taints"](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/))

- Lorsqu'une haute disponibilité est requise, chaque service du control plane doit être résilient

- Le control plane est alors répliqué sur plusieurs nodes

  (On parle parfois d'architecture "multi-master")

---

class: extra-details

## Exécuter le control plane hors conteneurs

- Les services du control plane peuvent s'exécuter dans ou en dehors de conteneurs

- Par exemple : dans la mesure où `etcd` est un service critique, certains administrateurs
  le déploient directement sur un cluster dédié (bare metal, sans conteneurs)

  (C'est ce qu'illustre le premier schéma "ultra complexe")

- Dans certaines offres Kubernetes managées (ex. AKS, GKE, EKS), le control plane est totalement abstrait

  (Nous ne visualisons qu'un endpoint d'API Kubernetes)

- Dans ce contexte, la notion de "master node" n'apparaît pas

*C'est la raison pour laquelle il est plus rigoureux d'employer l'expression "control plane" plutôt que "master".*

---

class: pic
![](images/control-planes/single-node-dev.svg)

---

class: pic
![](images/control-planes/managed-kubernetes.svg)

---

class: pic
![](images/control-planes/single-control-and-workers.svg)

---

class: pic
![](images/control-planes/stacked-control-plane.svg)

---

class: pic
![](images/control-planes/non-dedicated-stacked-nodes.svg)

---

class: pic
![](images/control-planes/advanced-control-plane.svg)

---

class: pic
![](images/control-planes/advanced-control-plane-split-events.svg)

---

class: extra-details

## Combien de nodes un cluster doit-il comporter ?

- Il n'y a pas de contrainte technique particulière

  (pas d'obligation d'avoir un nombre impair de nodes de travail pour un quorum)

- Un cluster peut comporter zéro node

  (mais il ne pourra démarrer aucun pod)

- Pour les phases de test et de développement, un seul node suffit

- Pour la production, veillez à toujours disposer de marge de capacité

  (afin que votre workload continue de tourner en cas de perte d'un node ou d'un groupe de nodes)

- Kubernetes est validé pour fonctionner jusqu'à [5000 nodes](https://kubernetes.io/docs/setup/best-practices/cluster-large/)

  (exploiter un cluster de cette envergure nécessite néanmoins de nombreux ajustements fins)

---

class: extra-details

## Doit-on obligatoirement faire tourner Docker ?

Non !

--

- Le Docker Engine a longtemps constitué l'option par défaut pour exécuter des conteneurs sur Kubernetes

- Le support direct de Docker (via le composant dockershim) a été retiré dans Kubernetes 1.24

- Nous pouvons utiliser d'autres runtimes enfichables via la *Container Runtime Interface* (CRI)

- <del>On pouvait également utiliser `rkt` ("Rocket") de CoreOS</del> (déprécié)

---

class: extra-details

## Runtimes disponibles via la CRI

- [containerd](https://github.com/containerd/containerd/blob/master/README.md)

  - maintenu par Docker, IBM et la communauté
  - utilisé par Docker Engine, microk8s, k3s, GKE ; utilisable aussi en standalone
  - fournit sa propre CLI, `ctr`

- [CRI-O](https://github.com/cri-o/cri-o/blob/master/README.md) :

  - maintenu par Red Hat, SUSE et la communauté
  - utilisé par OpenShift et Kubic
  - conçu spécifiquement comme un runtime minimaliste dédié à Kubernetes

- [Et d'autres](https://kubernetes.io/docs/setup/production-environment/container-runtimes/)

---

class: extra-details

## Doit-on tout de même utiliser Docker ?

Oui !

--

- Dans ce workshop, nous exécutons d'abord notre application sur un node unique

- Nous aurons besoin de builder des images et de les distribuer

- Ces opérations sont faisables sans Docker
  <br/>
  (mais pour certains langages ou frameworks, cela peut s'avérer beaucoup plus complexe)

- Docker reste encore aujourd'hui le container engine le plus stable
  <br/>
  (bien que les alternatives montent très rapidement en maturité)

---

class: extra-details

## Faut-il encore faire tourner Docker ?

- Sur nos clusters Kubernetes :

  *Plus aujourd'hui*

- Sur nos environnements de développement, pipelines de CI... :

  *Oui, très certainement*

---

## Interagir avec Kubernetes

- Nous interagissons avec notre cluster Kubernetes à travers l'API Kubernetes

- L'API Kubernetes est (essentiellement) de type RESTful

- Elle nous permet de créer, lire, mettre à jour et supprimer des *resources*

- Quelques types de ressources fondamentaux :

  - node (une machine — physique ou virtuelle — au sein de notre cluster)

  - pod (groupe de conteneurs s'exécutant ensemble sur un même node)

  - service (endpoint réseau stable permettant de joindre un ou plusieurs conteneurs)

---

class: pic

![Node, pod, container](images/k8s-arch3-thanks-weave.png)

---

## Scaling

- Comment ferions-nous pour scaler le pod illustré précédemment ?

- **À faire :** créer des pods additionnels

  - chaque pod peut être planifié sur un node différent

  - chaque pod disposera de sa propre adresse IP

- **À ne pas faire :** ajouter des conteneurs NGINX supplémentaires au sein du même pod

  - tous les conteneurs NGINX se retrouveraient sur le même node

  - ils partageraient la même adresse IP
    <br/>(provoquant des erreurs de type `Address already in use`)

---

## Regrouper ou séparer

- Doit-on regrouper, par exemple, un serveur d'application web et un cache ?
  <br/>
  (le "cache" étant un composant tel que Memcached ou Redis)

- Les placer **au sein du même pod** implique :

  - qu'ils doivent être scalés ensemble

  - qu'ils peuvent communiquer de manière extrêmement performante via `localhost`

- Les placer **dans des pods distincts** implique :

  - qu'ils peuvent être scalés indépendamment

  - qu'ils doivent communiquer à travers le réseau via des adresses IP distantes
    <br/>(induisant davantage de latence et des performances moindres)

- Les deux approches sont valables selon vos contraintes d'architecture

---

## Crédits

- Le premier schéma est une courtoisie de Lucas Käldström, issu de [cette présentation](https://speakerdeck.com/luxas/kubeadm-cluster-creation-internals-from-self-hosting-to-upgradability-and-ha)

  - il figure parmi les meilleurs diagrammes d'architecture Kubernetes disponibles !

- Le second schéma est proposé avec l'aimable autorisation de Weave Works

  - un *pod* peut regrouper plusieurs conteneurs collaborant étroitement

  - les adresses IP sont associées aux *pods*, et non aux conteneurs individuels

Les deux diagrammes sont utilisés avec autorisation.

???

:FR:- Concepts Kubernetes
:FR:- Kubernetes en théorie