# Exposing HTTP services with Ingress resources

- Service = couche 4 (TCP, UDP, SCTP)

  - fonctionne avec tous les protocoles TCP/UDP/SCTP

  - ne "voit" ni n'interprète le protocole HTTP

- Ingress = couche 7 (HTTP)

  - dédié uniquement au HTTP

  - capable de router les requêtes selon l'URI ou l'en-tête host

  - gère le TLS

---

## Why should we use Ingress resources?

Quelques cas d'usage :

- Routage par URI (par ex. pour les single page apps)

  `/api` → service `api:5000`

  tout le reste → service `static:80`

- Optimisation des coûts

  (utiliser des services `LoadBalancer` pour chaque composant deviendrait très onéreux)

- Gestion automatisée des certificats TLS

---

## `LoadBalancer` vs `Ingress`

- Service de `type: LoadBalancer`

  - requiert un contrôleur dédié (par ex. CCM, MetalLB)
  - si TLS est souhaité, il doit être géré par l'application
  - fonctionne pour n'importe quel protocole TCP (pas seulement HTTP)
  - n'interprète pas le protocole HTTP (aucun routage avancé)
  - engendre un coût pour chaque service

- Ingress

  - requiert un Ingress Controller
  - gère le TLS de manière transparente pour l'application
  - supporte uniquement le protocole HTTP
  - permet un routage basé sur le contenu (par ex. par URI)
  - coût réduit par service
    <br/>(la tarification exacte dépend du modèle du fournisseur cloud)

---

## Ingress resources

- Ressource de l'API Kubernetes (`kubectl get ingress`/`ingresses`/`ing`)

- Conçue pour exposer des services HTTP

- Nécessite un *Ingress Controller*

  (sinon, les ressources peuvent être créées, mais aucune action n'est effectuée)

- Certaines distributions Kubernetes installent automatiquement un Ingress Controller

  (ou proposent cette option via un simple switch lors de l'installation)

- Cela reste toutefois relativement rare, car il n'existe pas d'Ingress Controller universel ("one size fits all")

---

## Checking if we have an Ingress Controller

- Un Ingress Controller moderne crée une ressource IngressClass

- Nous pouvons simplement vérifier sa présence via `kubectl get ingressclasses`

- Exemple :
  ```shell
    $ kubectl get ingressclasses
    NAME      CONTROLLER                      PARAMETERS   AGE
    traefik   traefik.io/ingress-controller   <none>       139m  
  ```

- Il est également possible de constater une IngressClass sans Ingress Controller fonctionnel

  (par ex. si le contrôleur est défaillant ou a été partiellement désinstallé...)

---

## A taxonomy of Ingress Controllers

- Certains Ingress Controllers reposent sur des load balancers existants

  (HAProxy, NGINX...)

- Certains sont autonomes, et parfois spécifiquement conçus pour Kubernetes

  (Contour, Traefik...)

- D'autres sont propriétaires d'un constructeur matériel ou d'un fournisseur cloud spécifique

  (GKE Ingress, AWS ALB Ingress)

- Note : il n'existe aucun Ingress Controller "par défaut" ou "officiel" !

---

class: extra-details

## Details about these proprietary controllers

- GKE propose "[GKE Ingress]", un Ingress Controller sur mesure

  (activé par défaut mais [n'utilise pas IngressClass][gke-ingressclass])

- EKS dispose également d'un contrôleur "AWS ALB Ingress Controller"

  (non activé par défaut, nécessite une configuration supplémentaire)

- Ils s'appuient sur les load balancers HTTP managés des clouds respectifs

  (GCP HTTP LB, AWS ALB)

- Ils engendrent généralement un coût facturé *par ressource Ingress*

[GKE Ingress]: https://cloud.google.com/kubernetes-engine/docs/concepts/ingress
[gke-ingressclass]: https://docs.cloud.google.com/kubernetes-engine/docs/concepts/ingress#deprecated_annotation

---

class: extra-details

## Single or multiple LoadBalancer

- La plupart des Ingress Controllers créent un Service de type LoadBalancer

  (et reçoivent l'ensemble du trafic HTTP/HTTPS via celui-ci)

- Nous devons faire pointer nos enregistrements DNS vers l'adresse IP de ce LoadBalancer

- Quelques rares Ingress Controllers allouent un LoadBalancer dédié par ressource Ingress

  (exemple : les contrôleurs GKE Ingress et ALB Ingress mentionnés précédemment)

- Cela entraîne une augmentation des coûts

- Notez qu'il est possible de regrouper plusieurs "règles" au sein d'une même ressource Ingress

  (ceci réduira les coûts mais peut s'avérer moins pratique à administrer)

---

## Ingress standard features

- Load balancing

- Terminaison SSL / TLS

- Virtual hosting basé sur le nom (Name-based virtual hosting)

- Routage par URI

  (par ex. `/api` → `api-service`, `/static` → `assets-service`)

---

## Ingress extended features

(Pas systématiquement supportées ; configurables via des annotations, CRDs, etc.)

- Routage basé sur d'autres en-têtes (headers) ou cookies

- A/B testing

- Canary deployment

- etc.

*Prendre en charge ces fonctionnalités de façon standardisée et indépendante des constructeurs est
l'un des objectifs de la Gateway API. (Davantage de précisions à la fin de cette section !)*

---

## Principle of operation

- Étape 1 : déployer un *Ingress Controller*

  (configuration initiale unique ; réalisée en règle générale par l'administrateur du cluster)

- Étape 2 : créer les *ressources Ingress*

  - associe un nom de domaine et/ou un chemin à un Service Kubernetes

  - le contrôleur surveille les ressources Ingress et configure le load balancer en conséquence

- Étape 3 : configurer le DNS (optionnel)

  - associer les entrées DNS à l'adresse du load balancer

  - cette tâche peut être automatisée à l'aide d'[ExternalDNS]

[ExternalDNS]: https://github.com/kubernetes-sigs/external-dns

---

## Ingress in action

- Nous allons déployer un Ingress Controller

  (à moins que notre cluster n'en dispose déjà d'un prêt à l'emploi)

- Ensuite, nous créerons des ressources Ingress pour différents services HTTP

- Nous illustrerons également l'intégration DNS

- Si vous ne possédez pas de nom de domaine pour cette étape, vous pouvez utiliser [nip.io]

  (`*.1.2.3.4.nip.io` résout vers `1.2.3.4`)

---

## Deploying the Ingress Controller

- De nombreuses variantes existent, selon :

  - l'Ingress Controller retenu

  - la disponibilité ou non de Services de type `LoadBalancer`

  - l'outil de déploiement privilégié (Helm, YAML standard...)

- Si vous suivez une formation en présentiel/direct, nous utiliserons un raccourci

  (avec un manifeste prêt à l'emploi optimisé pour les clusters utilisés en cours)

- Dans le cas contraire, consultez au préalable la section dédiée à la mise en place d'un Ingress Controller

---

## If you're attending a live class...

- Chaque participant dispose d'un cluster préconfiguré

  (parfois plusieurs clusters, afin d'illustrer différents scénarios)

- Nous avons préparé un manifeste YAML qui se charge de déployer Traefik pour vous

.lab[

- Installer Traefik sur votre cluster :
  ```bash
  kubectl apply -f ~/container.training/k8s/traefik.yml
  ```

]

- Note : ce manifeste YAML est exclusivement destiné aux clusters des formations en direct !

---

class: extra-details

## What's about this manifest?

- Il exécute Traefik sous la forme d'un DaemonSet

  (il y aura ainsi une instance de Traefik sur chaque node du cluster)

- Les Pods utilisent `hostPort: 80`

- Cela signifie que nous pourrons nous connecter à n'importe quel node du cluster sur le port 80

- Il intègre également une *toleration* pour s'assurer que Traefik s'exécute sur chaque Node

  (y compris le Node hébergeant le control plane)

---

## Creating Ingress resources

- Nous allons utiliser l'image `jpetazzo/color`

- Cette image embarque un serveur HTTP statique minimaliste écoutant sur le port 80

- Nous allons déployer un Deployment, par ex. `blue`

- Nous exposerons ce Deployment à l'aide d'un Service

- Puis nous créerons un Ingress pour ce Service

---

## Deploying the `blue` app

- Rien de particulier ici ; nous créons simplement un Deployment et un Service

.lab[

- Créer le Deployment :
  ```bash
  kubectl create deployment blue --image=jpetazzo/color
  ```

- L'exposer via un Service :
  ```bash
  kubectl expose deployment blue --port=80
  ```

]

---

## Creating Ingress resources

- Il existe une commande utilitaire pratique : `kubectl create ingress`

  (disponible depuis Kubernetes 1.19 ; auparavant, le recours aux manifestes YAML était obligatoire)

- Une ressource Ingress peut contenir une ou plusieurs "rules" (règles)

.lab[

- Créer un Ingress doté d'une règle unique :
  ```bash
  kubectl create ingress blue --rule=/blue=blue:80
  ```
]

`/blue` = chemin HTTP que l'Ingress doit intercepter

`blue:80` = nom du Service + port vers lequel transférer les requêtes

---

## Testing our new Ingress

- Nous devons identifier l'adresse IP à joindre

- Si vous participez à un cours en direct, ou si vous avez installé votre Ingress Controller via un DaemonSet avec `hostPort` ou `hostNetwork` :

  *utilisez l'adresse IP de n'importe quel node de votre cluster*

- Si vous avez déployé votre Ingress Controller via un Service de type `LoadBalancer` :

  *utilisez l'EXTERNAL-IP du Service*

- Si vous travaillez sur un cluster de développement local :

  *cela dépend ; nous préconisons `kubectl port-forward` puis l'utilisation de `localhost`*

---

## Testing our new Ingress

- Connectez-vous à `http://<IP address>/blue`

- Nous devrions observer une réponse émise par le Deployment `blue`

---

## Using domain names

- Avec Ingress, nous pouvons exploiter ce que l'on nomme couramment le "name-based virtual hosting"

- Cela permet d'héberger plusieurs applications web derrière une adresse IP unique

- Il suffit d'attribuer un nom de domaine distinct pour chaque application web

  (par ex. : `blue.example.com`, `green.example.com`, `red.example.com`...)

- Nous pouvons employer un véritable nom de domaine ou, pour simplifier, [nip.io]

- Pour les étapes à venir, nous supposerons que notre Ingress Controller utilise l'adresse IP `A.B.C.D`

  (pensez à la remplacer par votre adresse réelle !)

---

## Before creating the Ingress

- Nous allons rendre le Deployment `blue` accessible à l'URL http://blue.A.B.C.D.nip.io

.lab[

- Constatons ce qui se produit si nous nous connectons à cette adresse dès maintenant :
  ```bash
  curl http://blue.A.B.C.D.nip.io
  ```

]

- Si nous utilisons Traefik, il nous retournera une sobre erreur `404 not found`

  (c'est le comportement attendu !)

---

## Creating the Ingress

- La démarche sera très proche de l'Ingress créé précédemment

- Toutefois, nous allons spécifier un nom de domaine dans la règle

.lab[

- Créer l'Ingress :
  ```bash
  kubectl create ingress blue-with-domain --rule=blue.A.B.C.D.nip.io/=blue:80
  ```

- Le tester :
  ```bash
  curl http://blue.A.B.C.D.nip.io
  ```

]

- Nous devrions obtenir une réponse provenant du Deployment `blue`

---

## Exact or prefix matches

- Par défaut, les règles Ingress s'appliquent en correspondance exacte (*exact match*)

  (la requête n'est routée que pour l'URL strictement définie)

.lab[

- Vérifier que seul le chemin `/` route vers l'application `blue` :
  ```bash
  curl http://blue.A.B.C.D.nip.io         # fonctionne
  curl http://blue.A.B.C.D.nip.io/hello   # ne fonctionne pas
  ```

]

- Comment modifier ce comportement ?

---

## Specifying a prefix match

- Si une règle se termine par `*`, elle sera interprétée comme une correspondance par préfixe (*prefix match*)

.lab[

- Créer une règle en prefix match pour le service `blue` :
  ```bash
  kubectl create ingress blue-with-prefix --rule=blue.A.B.C.D.nip.io/*=blue:80
  ```

- Vérifier que cela fonctionne :
  ```bash
  curl http://blue.A.B.C.D.nip.io/hello
  ```

]

---

## What do Ingress manifests look like?

- Examinons les manifestes générés par `kubectl create ingress` !

- Nous utiliserons `-o yaml` pour afficher le YAML produit par `kubectl`

- Et `--dry-run=client` pour demander à `kubectl` de ne pas créer la ressource sur le cluster
 
.lab[

- Générer et afficher quelques manifestes :
  ```bash
    kubectl create ingress -o yaml --dry-run=client \
            exact-route --rule=/blue=blue:80

    kubectl create ingress -o yaml --dry-run=client \
            with-a-domain --rule=blue.test/=blue:80

    kubectl create ingress -o yaml --dry-run=client \
            now-with-a-prefix --rule=blue.test/*=blue:80
  ```

]

---

## Multiple rules per Ingress resource

- Il est également envisageable de définir plusieurs règles au sein d'une même ressource Ingress

- Regardons à quoi cela ressemble !

.lab[

- Afficher le manifeste d'une ressource Ingress comportant plusieurs règles :
  ```bash
    kubectl create ingress -o yaml --dry-run=client rgb \
            --rule=/red*=red:80 \
            --rule=/green*=green:80 \
            --rule=/blue*=blue:80
  ```

]

---

class: extra-details

## Using multiple Ingress Controllers

- Plusieurs Ingress Controllers peuvent être actifs simultanément

  (par ex. Traefik et NGINX)

- Vous pouvez même disposer de plusieurs instances du même contrôleur

  (par ex. l'une dédiée au trafic interne, l'autre au trafic externe)

- Pour indiquer quel Ingress Controller doit être pris en charge par une ressource Ingress donnée :

  - avant Kubernetes 1.18, utiliser l'annotation `kubernetes.io/ingress.class`

  - depuis Kubernetes 1.18, utiliser le champ `ingressClassName`
    <br/>
    (qui doit référencer une ressource `IngressClass` existante)

---

## Ingress shortcomings

- De nombreux aspects ont été laissés de côté dans la spécification Ingress v1

  (par ex. : le routage des requêtes par pondération (weight), par cookies, ou inter-namespaces...)

- La plupart des Ingress Controllers proposent des mécanismes propriétaires pour pallier ces limites

- Mais étant spécifiques à chaque implémentation, les migrations s'en trouvent complexifiées

- Exemple : suppression de préfixes d'URL (*path prefix stripping*)

  - NGINX : [nginx.ingress.kubernetes.io/rewrite-target: /](https://github.com/kubernetes/ingress-nginx/blob/main/docs/examples/rewrite/README.md)

  - Traefik v1 : [traefik.ingress.kubernetes.io/rule-type: PathPrefixStrip](https://doc.traefik.io/traefik/migration/v1-to-v2/#strip-and-rewrite-path-prefixes)

  - Traefik v2 : [nécessite une CRD](https://doc.traefik.io/traefik/migration/v1-to-v2/#strip-and-rewrite-path-prefixes)

---

## A word about Ingress NGINX

- Deux Ingress Controllers reposent sur NGINX (tous deux open source)

- [F5 NGINX Ingress Controller][f5-nginx] alias "NGINX Ingress" ([dépôt GitHub][f5-nginx-repo], [documentation][f5-nginx-docs])

  - développé et maintenu par F5 (entreprise ayant acquis NGINX en 2019)

  - prend en charge des CRD propriétaires telles que [VirtualServer et VirtualServerRoute][f5-nginx-crds]

- Ingress NGINX Controller alias "Ingress NGINX" ([dépôt GitHub][k8s-nginx-repo], [documentation][k8s-nginx-docs])

  - l'un des premiers Ingress Controllers apparus sur Kubernetes

  - développé par la communauté

  - **n'est plus développé activement ; la maintenance cessera en mars 2026**
    (voir l'[annonce officielle][k8s-nginx-announcement])

[f5-nginx]: https://docs.nginx.com/nginx-ingress-controller/
[f5-nginx-docs]: https://docs.nginx.com/nginx-ingress-controller
[f5-nginx-repo]: https://github.com/nginx/kubernetes-ingress
[f5-nginx-crds]: https://docs.nginx.com/nginx-ingress-controller/configuration/virtualserver-and-virtualserverroute-resources/
[k8s-nginx-docs]: https://kubernetes.github.io/ingress-nginx/
[k8s-nginx-repo]: https://github.com/kubernetes/ingress-nginx
[k8s-nginx-announcement]: https://kubernetes.io/blog/2025/11/11/ingress-nginx-retirement/

---

## A word about software sustainability

- Extrait de l'annonce d'arrêt d'Ingress NGINX :

  *Despite the project’s popularity among users, Ingress NGINX has always struggled with insufficient or barely-sufficient maintainership. For years, the project has had only one or two people doing development work, on their own time, after work hours and on weekends.*

--

- Si vos workloads critiques de production dépendent de logiciels open source :

  *que se passe-t-il si les mainteneurs jettent l'éponge ?*

--

- Si vos workloads critiques de production dépendent de solutions commerciales :

  *que se passe-t-il si l'entreprise qui les édite fait faillite ?*

  *que se passe-t-il si elle modifie radicalement son modèle économique ou sa [grille][vmware1] [tarifaire][vmware2] ?*

[vmware1]: https://www.theregister.com/2025/05/22/euro_cloud_body_ecco_says_broadcom_licensing_unfair/
[vmware2]: https://www.ciodive.com/news/att-broadcom-vmware-price-hikes-court-battle/728603/

---

## Ingress in the future

- Le [Gateway API SIG](https://gateway-api.sigs.k8s.io/) incarne très probablement l'avenir d'Ingress

- Il introduit de nouvelles ressources :

 GatewayClass, Gateway, HTTPRoute, TCPRoute...

- Il a désormais atteint la parité de fonctionnalités avec Ingress

  (= il peut être utilisé à la place des ressources Ingress, ou en complément)

- Il s'avère toutefois plus complexe à configurer et opérer

  (du moins pour le moment !)

???

[nip.io]: http://nip.io

:EN:- The Ingress resource
:FR:- La ressource *ingress*