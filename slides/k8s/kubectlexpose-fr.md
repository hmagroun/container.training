# Exposer des conteneurs

- Nous pouvons nous connecter à nos pods en utilisant leur adresse IP

- Mais nous devons alors résoudre plusieurs problématiques :

  - comment identifier l'adresse IP du ou des pod(s) ?

  - comment s'y connecter depuis l'extérieur du cluster ?

  - comment répartir la charge du trafic (load balancing) ?

  - que se passe-t-il si un pod tombe en panne ?

- Kubernetes propose un type de ressource nommé *Service*

- Les Services répondent à l'ensemble de ces questions !

---

## ⚠️ Attention !

- Nous allons nous connecter directement aux pods et services via leurs adresses internes

- Cela ne fonctionnera que :

  - si vous participez à un cours en direct avec notre environnement de lab dédié

  - ou si vous utilisez nos dev containers au sein de codespaces

- Si vous utilisez un cluster Kubernetes standard (y compris minikube, KinD, etc.) :

  *vous ne pourrez pas accéder directement à ces adresses internes !*

- Dans ce cas, nous vous conseillons d'exécuter un conteneur interactif, par exemple :
  ```bash
  kubectl run --rm -ti --image=archlinux myshell
  ```

- ... Et à chaque fois que vous rencontrez une commande `curl` ou `ping`, exécutez-la directement dans ce conteneur

---

class: extra-details

## Pourquoi cela ?

- Les adresses internes ne sont accessibles que depuis l'intérieur du cluster

  (= depuis un pod, ou directement connecté en session sur un node)

- Nos environnements de lab spécifiques et nos dev containers nous permettent de le faire directement

  (ce qui s'avère bien plus confortable pour apprendre Kubernetes)

- Mais cela ne fonctionne pas sur des clusters Kubernetes standards

- À la place, nous pouvons utiliser [`kubectl port-forward`][kubectl-port-forward] sur ces clusters

[kubectl-port-forward]: https://kubernetes.io/docs/reference/kubectl/generated/kubectl_port-forward/

---

## Exécuter des conteneurs avec des ports ouverts

- Lançons un petit serveur web dans un conteneur

- Nous allons utiliser `jpetazzo/color`, un serveur HTTP minimaliste écrit en Go

- `jpetazzo/color` écoute sur le port 80

- Il sert une page affichant le nom du pod

  (ce qui sera très pratique pour vérifier le comportement du load balancing)

- Nous aurions également pu utiliser l'image officielle `nginx` à la place

  (mais il ne nous serait pas possible de distinguer les backends les uns des autres)

---

## Exécuter notre serveur HTTP

- Nous allons créer un deployment à l'aide de `kubectl create deployment`

- Cela va instancier un Pod exécutant notre serveur HTTP

.lab[

- Créer un deployment nommé `blue` :
  ```bash
  kubectl create deployment blue --image=jpetazzo/color
  ```

]

---

## Se connecter au serveur HTTP

- Connectons-nous directement au serveur HTTP

  (simplement pour vérifier que tout fonctionne ; nous ajouterons le Service ensuite)

.lab[

- Obtenir l'adresse IP du Pod :
  ```bash
  kubectl get pods -o wide
  ```

- Envoyer une requête HTTP au Pod :
  ```bash
  curl http://`IP-ADDRESS`
  ```

]

Vous devriez recevoir une réponse émise par le Pod.

---

## Le Pod ne possède pas d'"identité stable"

- L'adresse IP utilisée ci-dessus n'est pas "stable"

  (si le Pod est supprimé, le Pod de remplacement se verra attribuer une adresse différente)

.lab[

- Vérifier les adresses IP des Pods actifs :
  ```bash
  watch kubectl get pods -o wide
  ```

- Supprimer le Pod :
  ```bash
  kubectl delete pod `blue-xxxxxxxx-yyyyy`
  ```

- Constater que le Pod de remplacement dispose d'une adresse IP différente

]

---

## Les Services en résumé

- Les Services nous fournissent un *endpoint stable* pour joindre un pod ou un ensemble de pods

- Une façon simple de créer un service consiste à utiliser `kubectl expose`

- Si nous disposons d'un deployment nommé `my-little-deploy`, nous pouvons exécuter :

  `kubectl expose deployment my-little-deploy --port=80`

  ... et cela créera un service portant le même nom (`my-little-deploy`)

- Les Services sont automatiquement enregistrés dans une zone DNS interne

  (dans l'exemple ci-dessus, notre application peut désormais se connecter à http://my-little-deploy/)

---

## Exposer notre deployment

- Créons un Service pour notre Deployment

.lab[

- Exposer le port HTTP de notre serveur :
  ```bash
  kubectl expose deployment blue --port=80
  ```

- Consulter l'adresse IP allouée :
  ```bash
  kubectl get service
  ```

]

- Par défaut, cette commande crée un service de type `ClusterIP`

  (nous détaillerons les différents types de services plus loin)

---

class: extra-details

## Les Services opèrent en couche 4 (Layer 4)

- Les Services peuvent avoir des adresses IP, mais ils restent des composants de *couche 4*

  (un service n'est pas simplement une adresse IP ; c'est une adresse IP + un protocole + un port)

- En conséquence : vous *devez* obligatoirement renseigner le numéro de port de votre service

  (à quelques exceptions près, comme pour les services `ExternalName` ou les headless services abordés plus loin)

---

## Tester notre service

- Nous allons maintenant envoyer plusieurs requêtes HTTP vers notre Pod

.lab[

- Récupérer l'adresse IP allouée à notre service, *de manière programmatique :*
  ```bash
  CLUSTER_IP=$(kubectl get svc blue -o go-template='{{ .spec.clusterIP }}')
  ```

<!--
```hide kubectl wait deploy blue --for condition=available```
```key ^D```
```key ^C```
-->

- Envoyer une série de requêtes :
  ```bash
  for i in $(seq 10); do curl http://$CLUSTER_IP; done
  ```

]

---

## Un endpoint *stable*

- Observons le comportement lorsqu'un Pod rencontre une défaillance

.lab[

- Envoyer des requêtes en continu vers l'adresse du Service :
  ```bash
  while sleep 0.3; do curl -m1 http://$CLUSTER_IP; done
  ```

- Dans le même temps, supprimer le Pod :
  ```bash
  kubectl delete pod `blue-xxxxxxxx-yyyyy`
  ```

]

- Une brève interruption peut survenir au moment de la suppression du pod...

- ... Mais le flux de requêtes reprend immédiatement (sans nécessiter la moindre intervention manuelle)

- L'option `-m1` est employée ici pour définir un timeout de 1 seconde

---

## Load balancing

- Le Service agit également comme un load balancer

  (dès lors qu'il y a plusieurs Pods au sein du Deployment)

.lab[

- Mettre à l'échelle (scale up) le Deployment :
  ```bash
  kubectl scale deployment blue --replicas=3
  ```

- Envoyer une série de requêtes vers le Service :
  ```bash
  for i in $(seq 20); do curl http://$CLUSTER_IP; done
  ```

]

- Nos requêtes font l'objet d'un load balancing équilibré entre les Pods !

---

## Intégration DNS

- Kubernetes met à disposition un résolveur DNS interne

- Ce résolveur associe les noms de service à leurs adresses internes

- Par défaut, ce mécanisme opère uniquement *à l'intérieur des Pods* (pas directement depuis les nodes eux-mêmes)

.lab[

- Ouvrir un shell interactif dans un Pod :
  ```bash
  kubectl run --rm -it --image=archlinux test-dns-integration
  ```

- Résoudre le Service `blue` depuis ce Pod :
  ```bash
  curl blue
  ```

]

---

class: extra-details

## Sous le capot...

- Inspectons le contenu du fichier `/etc/resolv.conf` à l'intérieur d'un Pod

- Il devrait ressembler approximativement à ceci :
  ```
  search default.svc.cluster.local svc.cluster.local cluster.local ...
  nameserver 10.96.0.10
  options ndots:5
  ```

- Détaillons la signification de ces lignes...

---

class: extra-details

## `nameserver 10.96.0.10`

- Il s'agit de l'adresse du serveur DNS interrogé par les applications exécutées dans le Pod

- L'adresse exacte peut varier

  (celle-ci correspond à la valeur par défaut lors du déploiement d'un cluster avec `kubeadm`)

- Cette adresse pointe vers un Service de notre cluster

- Vérifier les services présents dans le namespace `kube-system` :
  ```bash
  kubectl get services --namespace=kube-system
  ```

- Vous y trouverez typiquement un service nommé `kube-dns` portant cette adresse précise

  (il s'agit du service DNS interne de Kubernetes !)

---

class: extra-details

## `search default.svc.cluster.local ...`

- Il s'agit du domaine de recherche ("search list")

- Lorsqu'un programme cherche à résoudre `foo`, le résolveur tente successivement :

  `foo.default.svc.cluster.local` (si le Pod réside dans le Namespace `default`)

  `foo.svc.cluster.local`

  `foo.cluster.local`

  ... (les autres entrées figurant dans la liste de recherche) ...

  `foo`

- Ainsi, s'il existe un Service nommé `foo` dans le Namespace du Pod, son adresse IP est immédiatement retournée !

---

class: extra-details

## Pour aller plus loin

- Si vous souhaitez approfondir la résolution DNS sous Kubernetes et Linux...

  consultez [cet article de blog][dnsblog] !

[dnsblog]: https://jpetazzo.github.io/2024/05/12/understanding-kubernetes-dns-hostnetwork-dnspolicy-dnsconfigforming/

---

## Avantages des Services

- Il n'est plus nécessaire de rechercher manuellement l'adresse IP du ou des pod(s)

  (nous résolvons l'adresse IP du service par DNS)

- Il existe plusieurs types de services, certains autorisant le trafic externe

  (par ex. `LoadBalancer` et `NodePort`)

- Les Services assurent le load balancing

  (aussi bien pour le trafic interne qu'externe)

- Les adresses des Services sont complètement découplées des adresses des pods

  (lorsqu'un pod crash, le service réachemine le trafic en toute transparence vers son remplaçant)

???

:FR:- Exposer un service
:FR:- Le DNS interne de Kubernetes et la *service discovery*