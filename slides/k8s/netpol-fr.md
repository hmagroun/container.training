# Network policies

- Les Namespaces nous aident à *organiser* les ressources

- Les Namespaces ne fournissent pas d'isolation

- Par défaut, chaque pod peut contacter n'importe quel autre pod

- Par défaut, chaque service accepte du trafic provenant de n'importe qui

- Si nous souhaitons un comportement différent, nous avons besoin de *network policies*

---

## Qu'est-ce qu'une network policy ?

Une network policy est définie par les éléments suivants :

- Un *pod selector* indiquant à quels pods elle s'applique

  par ex. : "tous les pods dans le namespace `blue` avec le label `zone=internal`"

- Une liste de *règles d'ingress* indiquant quel trafic entrant est autorisé

  par ex. : "les connexions TCP vers les ports 8000 et 8080 en provenance des pods ayant le label `zone=dmz`,
  et depuis le subnet externe 4.42.6.0/24, sauf 4.42.6.5"

- Une liste de *règles d'egress* indiquant quel trafic sortant est autorisé

Une network policy peut comporter des règles d'ingress, d'egress, ou les deux.

---

## Comment s'appliquent les network policies ?

- Un pod peut être "sélectionné" par un nombre quelconque de network policies

- Si un pod n'est sélectionné par aucune network policy, son trafic n'est soumis à aucune restriction

  (En d'autres termes : en l'absence de network policies, tout le trafic est autorisé)

- Si un pod est sélectionné par au moins une network policy, alors tout le trafic est bloqué...

  ... à moins qu'il ne soit explicitement autorisé par l'une de ces network policies

---

class: extra-details

## Le filtrage de trafic est orienté flux

- Les network policies gèrent des *connexions*, et non des paquets individuels

- Exemple : pour autoriser les connexions HTTP (80/tcp) vers le pod A, seule une règle d'ingress est requise

  (Il n'est pas nécessaire d'ajouter une règle d'egress correspondante pour autoriser le passage du trafic de réponse)

- Cela s'applique également au trafic UDP

  (Autoriser le trafic DNS peut s'effectuer au moyen d'une unique règle)

- Les implémentations de network policies utilisent un suivi de connexion avec état (stateful connection tracking)

---

## Trafic pod-à-pod

- Les connexions du pod A vers le pod B doivent être autorisées par les deux pods :

  - le pod A ne doit avoir aucune restriction, ou autoriser la connexion via une règle d'*egress*

  - le pod B ne doit avoir aucune restriction, ou autoriser la connexion via une règle d'*ingress*

- En conséquence : si une network policy restreint le trafic entrant ou sortant d'un pod,
  <br/>
  cette restriction ne peut pas être contournée par une network policy sélectionnant un autre pod

- Cela empêche une entité administrant les network policies dans le namespace A
  (mais sans permission pour le faire dans le namespace B)
  d'ajouter des network policies lui accordant un accès au namespace B

---

## La raison d'être des network policies

- En sécurité réseau, il est généralement considéré préférable de "tout interdire, puis autoriser sélectivement"

  (L'autre approche, "tout autoriser, puis bloquer sélectivement", expose trop facilement au risque d'oublier des failles)

- Dès qu'une network policy sélectionne un pod, le pod bascule dans cette logique de "deny all"

- D'autres network policies peuvent ouvrir des accès additionnels

- Les bonnes network policies doivent être ciblées le plus précisément possible

- En particulier : assurez-vous que le selector ne soit pas trop large

  (Sinon, vous risquez d'impacter des pods qui étaient par ailleurs bien sécurisés)

---

## Notre première network policy

Voici notre plan d'action :

- exécuter un serveur web dans un pod

- créer une network policy pour bloquer tout accès au serveur web

- créer une autre network policy pour n'autoriser l'accès que depuis des pods spécifiques

---

## Exécuter notre serveur web de test

.lab[

- Utilisons l'image `nginx` :
  ```bash
  kubectl create deployment testweb --image=nginx
  ```

<!--
```bash
kubectl wait deployment testweb --for condition=available
```
-->

- Identifiez l'adresse IP du pod à l'aide de l'une de ces deux commandes :
  ```bash
  kubectl get pods -o wide -l app=testweb
  IP=$(kubectl get pods -l app=testweb -o json | jq -r .items[0].status.podIP)
  ```

- Vérifiez que nous pouvons nous connecter au serveur :
  ```bash
  curl $IP
  ```
]

La commande `curl` devrait afficher la page "Welcome to nginx!".

---

## Ajouter une network policy très restrictive

- La policy sélectionnera les pods dotés du label `app=testweb`

- Elle spécifiera une liste vide de règles d'ingress (ne correspondant à aucun flux)

.lab[

- Appliquez la policy présente dans ce fichier YAML :
  ```bash
  kubectl apply -f ~/container.training/k8s/netpol-deny-all-for-testweb.yaml
  ```

- Vérifiez si nous pouvons toujours accéder au serveur :
  ```bash
  curl $IP
  ```

<!--
```wait curl```
```key ^C```
-->

]

La commande `curl` devrait désormais expirer (timeout).

---

## Examiner la network policy

Voici le fichier que nous venons d'appliquer :

```yaml
@@INCLUDE[k8s/netpol-deny-all-for-testweb.yaml]
```

---

## Autoriser les connexions uniquement depuis des pods spécifiques

- Nous souhaitons autoriser le trafic depuis les pods portant le label `run=testcurl`

- Rappel : ce label est automatiquement appliqué lorsque nous exécutons `kubectl run testcurl ...`

.lab[

- Appliquez une autre policy :
  ```bash
  kubectl apply -f ~/container.training/k8s/netpol-allow-testcurl-for-testweb.yaml
  ```

]

---

## Examiner la network policy

Voici le second fichier que nous avons appliqué :

```yaml
@@INCLUDE[k8s/netpol-allow-testcurl-for-testweb.yaml]
```

---

## Tester la network policy

- Créons des pods disposant, et ne disposant pas, du label requis

.lab[

- Tentez de vous connecter à testweb depuis un pod avec le label `run=testcurl` :
  ```bash
  kubectl run testcurl --rm -i --image=centos -- curl -m3 $IP
  ```

- Tentez de vous connecter à testweb avec un label différent :
  ```bash
  kubectl run testkurl --rm -i --image=centos -- curl -m3 $IP
  ```

]

La première commande fonctionnera (et affichera la page "Welcome to nginx!").

La deuxième commande échouera et se terminera en timeout au bout de 3 secondes.

(Le timeout est configuré à l'aide de l'option `-m3`.)

---

## Un avertissement important

- Certains plugins réseau ne proposent qu'un support partiel des network policies

- Par exemple, Weave a ajouté le support des règles d'egress [dans la version 2.4](https://github.com/weaveworks/weave/pull/3313) (sortie en juillet 2018)

- Et a ajouté le support d'ipBlock [dans la version 2.5](https://github.com/weaveworks/weave/pull/3367) (sortie en nov. 2018)

- Avant 2023, AWS EKS n'installait par défaut aucun contrôleur de network policy

- Les fonctionnalités non prises en charge peuvent être ignorées silencieusement

  (Vous laissant croire que votre environnement est sécurisé alors qu'il ne l'est pas)

---

## Network policies, pods et services

- Les network policies s'appliquent aux *pods*

- Un *service* peut sélectionner plusieurs pods

  (Et assurer le load balancing du trafic entre eux)

- Il est possible de pouvoir se connecter à certains pods, mais pas à d'autres

  (En fonction de la façon dont les network policies ont été définies pour ces pods)

- Dans ce cas, les connexions au service réussiront ou échoueront de manière aléatoire

  (Selon que la requête a été acheminée vers un pod auquel nous avons accès ou non)

---

## Network policies et namespaces

- Une stratégie efficace consiste à isoler un namespace, de sorte que :

  - tous les pods du namespace puissent communiquer entre eux

  - les autres namespaces ne puissent pas accéder aux pods

  - les accès externes doivent être activés explicitement

- Voyons à quoi cela ressemble pour l'application DockerCoins !

---

## Network policies pour DockerCoins

- Nous allons appliquer deux policies

- La première policy empêchera le trafic provenant d'autres namespaces

- La deuxième policy autorisera le trafic vers les pods `webui`

- C'est tout ce dont nous avons besoin pour cette application !

---

## Bloquer le trafic provenant d'autres namespaces

Cette policy sélectionne tous les pods du namespace courant.

Elle autorise le trafic uniquement depuis les pods du namespace courant.

(Un `podSelector` vide signifie "tous les pods".)

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: deny-from-other-namespaces
spec:
  podSelector: {}
  ingress:
  - from:
    - podSelector: {}
```

---

## Autoriser le trafic vers les pods `webui`

Cette policy sélectionne tous les pods portant le label `app=webui`.

Elle autorise le trafic provenant de n'importe quelle source.

(Un champ `from` vide signifie "toutes les sources".)

```yaml
kind: NetworkPolicy
apiVersion: networking.k8s.io/v1
metadata:
  name: allow-webui
spec:
  podSelector:
    matchLabels:
      app: webui
  ingress:
  - from: []
```

---

## Appliquer les deux network policies

- Les deux network policies sont déclarées dans le fichier @@LINK[k8s/netpol-dockercoins.yaml]

.lab[

- Appliquez les network policies :
  ```bash
  kubectl apply -f ~/container.training/k8s/netpol-dockercoins.yaml
  ```

- Vérifiez que nous pouvons toujours accéder à la web UI depuis l'extérieur
  <br/>
  (et que l'application fonctionne toujours correctement !)

- Vérifiez que nous ne pouvons plus nous connecter à `rng` ou `hasher` via leur ClusterIP

]

Note : l'utilisation de `kubectl proxy` ou `kubectl port-forward` nous permet de nous connecter
indépendamment des network policies en place. Cela permet de debugger et
diagnostiquer facilement, sans avoir à ouvrir de brèches dans notre pare-feu.

---

## Nettoyer nos network policies

- Les network policies que nous avons installées bloquent tout trafic vers le namespace par défaut

- Nous devons les supprimer, sans quoi les prochaines démonstrations et exercices échoueront !

.lab[

- Supprimez toutes les network policies :
  ```bash
  kubectl delete networkpolicies --all
  ```

]

---

## Protéger le control plane

- Devrions-nous ajouter des network policies pour bloquer les accès non autorisés au control plane ?

  (etcd, API server, etc.)

--

- Au premier abord, cela semble être une bonne idée...

--

- Mais cela ne *devrait pas* être nécessaire :

  - tous les plugins réseau ne prennent pas en charge les network policies

  - le control plane est sécurisé par d'autres mécanismes (principalement le mutual TLS)

  - le code s'exécutant dans nos pods peut légitimement avoir besoin de contacter l'API
    <br/>
    (et il peut le faire en toute sécurité grâce au modèle de permissions de l'API)

- Si nous bloquons l'accès au control plane, nous risquons de perturber du code légitime

- ... Sans nécessairement améliorer la sécurité

---

## Outils et ressources

- [Cilium Network Policy Editor](https://editor.cilium.io/)

- [Tufin Network Policy Viewer](https://orca.tufin.io/netpol/)

- [`kubectl np-viewer`](https://github.com/runoncloud/kubectl-np-viewer) (plugin kubectl)

- Deux ressources par [Ahmet Alp Balkan](https://ahmet.im/) :

  - une [excellente présentation sur les network policies](https://www.youtube.com/watch?list=PLj6h78yzYM2P-3-xqvmWaZbbI1sW-ulZb&v=3gGpMmYeEO8) à la KubeCon North America 2017

  - un dépôt de [recettes prêtes à l'emploi](https://github.com/ahmetb/kubernetes-network-policy-recipes) pour les network policies

---

## Documentation

- Comme toujours, la [documentation Kubernetes](https://kubernetes.io/docs/concepts/services-networking/network-policies/) constitue un excellent point de départ

- La documentation de l'API détaille la structure des différents objets : <!-- ##VERSION## -->

  - [NetworkPolicy](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.28/#networkpolicy-v1-networking-k8s-io)

  - [NetworkPolicySpec](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.28/#networkpolicyspec-v1-networking-k8s-io)

  - [NetworkPolicyIngressRule](https://kubernetes.io/docs/reference/generated/kubernetes-api/v1.28/#networkpolicyingressrule-v1-networking-k8s-io)

  - etc.

???

:EN:- Isolating workloads with Network Policies
:FR:- Isolation réseau avec les *network policies*