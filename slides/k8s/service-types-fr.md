# Service Types

- Il existe différents types de services :

  `ClusterIP`, `NodePort`, `LoadBalancer`, `ExternalName`

- Il existe également des *headless services*

- Les services peuvent également porter des *external IPs* optionnelles

- Il existe aussi un autre type de ressource nommé *Ingress*

  (spécifiquement dédié aux services HTTP)

- Cela fait beaucoup d'éléments ! Commençons par les bases...

---

## `ClusterIP`

- C'est le type de service par défaut

- Une adresse IP virtuelle est allouée pour le service

  (dans une plage privée interne ; par ex. 10.96.0.0/12)

- Cette adresse IP est accessible uniquement depuis l'intérieur du cluster (nodes et pods)

- Notre code peut se connecter au service en utilisant le numéro de port d'origine

- Idéal pour la communication interne au sein du cluster

---

class: pic
![](images/kubernetes-services/11-CIP-by-addr.png)

---

class: pic
![](images/kubernetes-services/12-CIP-by-name.png)

---

class: pic
![](images/kubernetes-services/13-CIP-both.png)

---

class: pic
![](images/kubernetes-services/14-CIP-headless.png)

---

## `LoadBalancer`

- Un load balancer externe est alloué pour le service

  (généralement un load balancer cloud, par ex. ELB sur AWS, GLB sur GCE...)

- Cette option n'est disponible que lorsque l'infrastructure sous-jacente propose une forme de
  "load balancer as a service"

  (ou dans certains cas spécifiques avec des add-ons comme [MetalLB])

- Chaque service de ce type engendre généralement un coût financier direct

  (par ex. quelques centimes par heure sur AWS ou GCE)

- Idéalement, le trafic transite directement du load balancer vers les pods

- En pratique, il passe souvent d'abord par un `NodePort`

[MetalLB]: https://metallb.io/

---

class: pic
![](images/kubernetes-services/31-LB-no-service.png)

---

class: pic
![](images/kubernetes-services/32-LB-plus-cip.png)

---

class: pic
![](images/kubernetes-services/33-LB-plus-lb.png)

---

class: pic
![](images/kubernetes-services/34-LB-internal-traffic.png)

---

class: pic
![](images/kubernetes-services/35-LB-pending.png)

---

class: pic
![](images/kubernetes-services/36-LB-ccm.png)

---

class: pic
![](images/kubernetes-services/37-LB-externalip.png)

---

class: pic
![](images/kubernetes-services/38-LB-external-traffic.png)

---

class: pic
![](images/kubernetes-services/39-LB-all-traffic.png)

---

class: pic
![](images/kubernetes-services/41-NP-why.png)

---

class: pic
![](images/kubernetes-services/42-NP-how-1.png)

---

class: pic
![](images/kubernetes-services/43-NP-how-2.png)

---

class: pic
![](images/kubernetes-services/44-NP-how-3.png)

---

class: pic
![](images/kubernetes-services/45-NP-how-4.png)

---

class: pic
![](images/kubernetes-services/46-NP-how-5.png)

---

class: pic
![](images/kubernetes-services/47-NP-only.png)

---

## `NodePort`

- Un numéro de port est alloué pour le service

  (par défaut, dans la plage 30000-32767)

- Ce port est ouvert et exposé *sur l'ensemble de nos nodes*, et tout client peut s'y connecter

  (nous pouvons contacter n'importe quel node sur ce port pour joindre le service)

- Notre code applicatif doit être configuré pour cibler ce nouveau numéro de port

- Sous le capot : `kube-proxy` configure un ensemble de règles de port forwarding sur nos nodes

  (via `iptables`, `ipvs`, `nftables`... plusieurs implémentations sont disponibles)

- Option très pratique pour le trafic externe lorsque les Services de type `LoadBalancer` ne sont pas disponibles

  (par ex. certains clusters déployés on-premises et/ou via kubeadm)

---

class: extra-details

## `ExternalName`

- Les services de type `ExternalName` fonctionnent de manière très différente

- Aucun load balancer (qu'il soit interne ou externe) n'est créé

- Seule une entrée DNS est injectée dans le DNS administré par Kubernetes

- Cette entrée DNS correspond simplement à un enregistrement `CNAME` pointant vers la cible spécifiée

Exemple :
```bash
kubectl create service externalname k8s --external-name kubernetes.io
```
*Crée un enregistrement CNAME `k8s` pointant vers `kubernetes.io`*

---

class: extra-details

## External IPs

- Nous pouvons adjoindre une External IP à un service, par exemple :
  ```bash
  kubectl expose deploy my-little-deploy --port=80 --external-ip=1.2.3.4
  ```

- `1.2.3.4` doit correspondre à l'adresse de l'un de nos nodes

  (il peut également s'agir d'une adresse virtuelle, d'une adresse de service ou VIP partagée par plusieurs nodes)

- Les connexions à destination de `1.2.3.4:80` seront routées vers notre service

- Les External IPs s'affichent également sur les services de type `LoadBalancer`

  (elles sont ajoutées automatiquement par le contrôleur provisionnant le load balancer)

---

class: extra-details

## Headless services

- Dans certaines situations, nous voulons accéder directement à nos services déployés à l'échelle :

  - pour économiser une fraction de latence (généralement moins de 1 ms)

  - pour se connecter sur des ports arbitraires (au lieu d'une liste restreinte et statique)

  - pour communiquer via un protocole autre que UDP ou TCP

  - pour gérer nous-mêmes la répartition de charge côté client (client-side load balancing)

  - ...

- Dans ce scénario, nous pouvons recourir à un "headless service"

---

class: extra-details

## Création d'un headless service

- Un headless service s'obtient en définissant le champ `clusterIP` à `None`

  (soit via le flag `--cluster-ip=None`, soit en fournissant un manifest YAML dédié)

- Par conséquent, le service ne dispose d'aucune adresse IP virtuelle

- En l'absence d'adresse IP virtuelle, aucun load balancer n'est mis en place

- CoreDNS résout le nom de service en renvoyant directement les adresses IP des pods sous forme d'enregistrements `A` multiples

- Cela fournit un mécanisme direct pour découvrir l'ensemble des replicas d'un deployment

---

class: extra-details

## Services et endpoints

- Un service est adossé à un ensemble d'"endpoints"

- Chaque endpoint représente un couple host + port sur lequel le service est joignable

- Les endpoints sont surveillés et mis à jour automatiquement par Kubernetes

.lab[

- Examiner les endpoints que Kubernetes a associés à notre service `blue` :
  ```bash
  kubectl describe service blue
  ```

]

Dans la sortie, repérez la ligne débutant par `Endpoints:`.

Cette ligne énumère une liste d'adresses au format `host:port`.

---

class: extra-details

## Visualiser le détail des endpoints

- Lorsque les endpoints sont nombreux, les commandes d'affichage usuelles tronquent la liste :
  ```bash
  kubectl get endpoints
  ```

- Pour afficher l'intégralité de la liste, utilisez l'une des commandes suivantes :
  ```bash
  kubectl describe endpoints blue
  kubectl get endpoints blue -o yaml
  ```

- Ces commandes retournent une liste d'adresses IP

- Ces adresses IP doivent correspondre à celles des pods sous-jacents :
  ```bash
  kubectl get pods -l app=blue -o wide
  ```

---

class: extra-details

## `endpoints` et non `endpoint`

- `endpoints` est la seule ressource dont le type ne s'emploie pas au singulier

```bash
$ kubectl get endpoint
error: the server doesn't have a resource type "endpoint"
```

- Cela s'explique par le fait que le type lui-même est au pluriel dans l'API (contrairement aux autres ressources)

- Il n'existe pas d'objet unitaire `endpoint` : la structure Go est `type Endpoints struct`

- Le type ne représente pas un endpoint isolé, mais directement une collection d'endpoints

---

class: extra-details

## `Ingress`

- L'Ingress constitue un type (kind) de ressource distinct

- Il est conçu spécifiquement pour le trafic applicatif HTTP/HTTPS

  (et non TCP brut ou UDP)

- Il prend en charge la terminaison des certificats TLS, la réécriture d'URL...

- Il nécessite la présence d'un *Ingress Controller* actif pour fonctionner

---

class: pic
![](images/kubernetes-services/61-ING.png)

---

class: pic
![](images/kubernetes-services/62-ING-path.png)

---

class: pic
![](images/kubernetes-services/63-ING-policy.png)

---

class: pic
![](images/kubernetes-services/64-ING-nolocal.png)

---

class: extra-details

## Traffic engineering

- Par défaut, les flux vers une ClusterIP ou un NodePort sont répartis
  sur l'ensemble des backends associés au Service

- Cela peut générer des sauts réseau supplémentaires (network hops), sources de latence additionnelle

- Pour éliminer ce saut intermédiaire, plusieurs mécanismes sont disponibles :

  - `spec.externalTrafficPolicy`

  - `spec.internalTrafficPolicy`

  - L'annotation [Topology aware routing](https://kubernetes.io/docs/concepts/services-networking/topology-aware-routing/) (beta)

  - `spec.trafficDistribution` (alpha en 1.30, beta en 1.31)

---

## `internal / externalTrafficPolicy`

- S'applique respectivement aux flux ciblant `ClusterIP` et `NodePort`

- Accepte les valeurs `Cluster` ou `Local`

- `Cluster` : répartition de charge sur l'intégralité des backends du cluster (par défaut)

- `Local` : routage restreint aux seuls backends locaux (situés sur le même node)

- Avec `Local`, si aucun backend n'est présent localement sur le node, la connexion est rejetée !

  (ce paramètre constitue une règle stricte, et non une simple préférence)

- Exemple : `externalTrafficPolicy: Local` pour les contrôleurs Ingress

  (comme illustré sur les schémas précédents)

---

class: extra-details

## Topology aware routing

- En statut beta depuis Kubernetes 1.23

- Activé via l'annotation `service.kubernetes.io/topology-mode=Auto` 

- S'appuie sur le label de node `topology.kubernetes.io/zone`

- Le proxy de service Kubernetes tentera de confiner le trafic au sein d'une même zone

  (les flux émis par un pod en zone `a` seront acheminés vers des pods situés en zone `a`)

- ...Sauf si aucun pod n'est disponible dans la zone (auquel cas un repli s'opère sur l'ensemble des zones)

- Attention : cela peut perturber le fonctionnement de l'autoscaling !

---

class: extra-details

## `spec.trafficDistribution`

- [KEP4444, Traffic Distribution for Services][kep4444]

- Succède au Topology aware routing

- Plusieurs valeurs sont supportées

- `PreferClose` (alpha depuis K8S 1.30, beta depuis K8S 1.31, stable depuis K8S 1.33)

  "tente d'acheminer le trafic vers les endpoints situés dans la même zone que le client"

- `PreferSameZone` (beta depuis K8S 1.34)

  "identique à `PreferClose`, mais avec une sémantique plus explicite sur l'intention"

- `PreferSameNode` (beta depuis K8S 1.34)

  "tente d'acheminer le trafic vers les endpoints situés sur le même node que le client"

[kep4444]: https://github.com/kubernetes/enhancements/issues/4444

???

:FR:- Différents types de services : ClusterIP, NodePort, LoadBalancer