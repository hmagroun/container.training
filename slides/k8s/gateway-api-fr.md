# The Gateway API

- Au fil du temps, Kubernetes a introduit plusieurs méthodes pour exposer des conteneurs

- Dans les premières versions de Kubernetes, nous utilisions un `Service` de type `LoadBalancer`

- Les services HTTP nécessitent cependant souvent des fonctionnalités complémentaires :

  - routage basé sur le contenu (router les requêtes selon l'URI, les en-têtes HTTP...)

  - terminaison TLS

  - middlewares (par ex. authentification)

  - etc.

- Cela a conduit à l'introduction de la ressource `Ingress`

---

## Histoire d'Ingress

- Kubernetes 1.8 (septembre 2017) a introduit `Ingress` (v1beta1)

- Kubernetes 1.19 (août 2020) a promu `Ingress` au statut GA (v1)

- Ingress prend en charge :

  - le routage basé sur le contenu via l'URI ou l'en-tête HTTP `Host:`

  - la terminaison TLS (avec une intégration fluide avec par ex. cert-manager)

- Ingress ne prend pas en charge :

  - le routage basé sur le contenu via d'autres en-têtes (par ex. les cookies)

  - les middlewares

  - la répartition de trafic (traffic split) pour par ex. des déploiements canary

---

## Le besoin d'une solution plus avancée

- Pratiquement *chaque* Ingress Controller a développé des extensions propriétaires :

  - l'annotation `nginx.ingress.kubernetes.io/configuration-snippet`

  - Traefik propose des CRDs telles que `IngressRoute`, `TraefikService`, `Middleware`...

  - HAProxy propose des CRDs telles que `Backend`, `TCP`...

  - etc.

- Ingress était trop spécifique au trafic de niveau 7 (HTTP)

- Un tout nouvel ensemble d'APIs et de ressources devenait indispensable !

---

## La Gateway API en résumé

- Prise en charge des routes HTTP, GRPC, TCP, TLS, UDP

  (note : en octobre 2025, seules les routes HTTP et GRPC sont en statut GA)

- Modèle de permissions à granularité plus fine

  (par ex. définir quels namespaces peuvent utiliser une "gateway" spécifique ; détails plus loin)

- Standardisation de davantage de fonctionnalités "core" par rapport à Ingress

  (routage par en-têtes, pondération du trafic, réécriture des requêtes et réponses...)

- Perspectives d'extensions futures facilitées grâce à différents ensembles de fonctionnalités

  (`Core` vs `Extended` vs `Implementation-specific`)

- Utilisable également au sein des service meshes

---

## Personas de la Gateway API

- Ingress distinguait informellement deux personas :

  - l'administrateur de cluster (installe et administre l'Ingress Controller)

  - le développeur d'application (crée les ressources Ingress)

- La Gateway API [définit formellement trois personas][gateway-personas] :

  - le fournisseur d'infrastructure (infrastructure provider)
    <br/>
    (~administrateur réseau ; intervient potentiellement chez les fournisseurs managés)

  - l'opérateur de cluster (cluster operator)
    <br/>
    (~administrateur Kubernetes ; gère potentiellement plusieurs clusters)

  - le développeur d'application (application developer)

[gateway-personas]: https://gateway-api.sigs.k8s.io/concepts/roles-and-personas/

---

class: pic

## Ressources de la Gateway API

![Diagram showing GatewayClass, Gateway, HTTPRoute, Service](https://gateway-api.sigs.k8s.io/images/resource-model.png)

---

## Ressources de la Gateway API

- `Service` = notre bon vieux Service Kubernetes

- `HTTPRoute` = décrit quelles requêtes doivent être acheminées vers quel `Service`

  (analogue à la ressource `Ingress`)

- `Gateway` = point d'entrée du trafic dans le système

  (peut correspondre par ex. à un `Service` de type `LoadBalancer`)

- `GatewayClass` = représente différents types de `Gateways`

  (de nombreux contrôleurs de gateway n'en proposent qu'une seule)

---

## Anatomie d'une `HTTPRoute`

- `spec.parentRefs` = provenance des requêtes

  - typiquement une unique `Gateway`

  - peut désigner plusieurs ressources `Gateway`

  - peut également être un `Service` (pour les usages en cluster mesh)

- `spec.hostnames` = hôtes ciblés (en-tête HTTP `Host:`)

- `spec.rules[].matches` = requêtes ciblées (correspondance de chemins, d'en-têtes...)

- `spec.rules[].filters` = transformations optionnelles (modifier les headers, réécrire l'URI...)

- `spec.rules[].backendRefs` = destination vers laquelle acheminer les requêtes

---

## `HTTPRoute` minimale

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: xyz
spec:
  parentRefs:
  - group: gateway.networking.k8s.io
    kind: Gateway
    name: my-gateway
    namespace: my-gateway-namespace
  hostnames: [ xyz.example.com ]
  rules:
  - backendRefs:
    - name: xyz
      port: 80
```

---

## La Gateway API en action

- Déployons Traefik en mode Gateway API !

- Nous utiliserons le [Helm chart officiel de Traefik][traefik-chart]

- Nous devons configurer quelques values

- `providers.kubernetesGateway.enabled=true`

  *active le provisionnement de la Gateway API*

- `gateway.listeners.web.namespacePolicy.from=All`

  *autorise les `HTTPRoutes` de tous les namespaces à référencer la `Gateway` par défaut*

[traefik-chart]: https://artifacthub.io/packages/helm/traefik/traefik

---

## `LoadBalancer` vs `hostPort`

- Si nous utilisons un cluster Kubernetes managé, nous emploierons le mode par défaut :

  - Traefik s'exécute via un `Deployment`

  - le `Service` Traefik est de type `LoadBalancer`

  - nous nous connectons à l'adresse IP publique du `LoadBalancer`

- Sans CCM (ou sans `Service` de type `LoadBalancer`), nous procédons différemment :

  - Traefik s'exécute via un `DaemonSet`

  - le `Service` Traefik est de type `ClusterIP` (pas strictement obligatoire, mais plus propre)

  - nous nous connectons à l'adresse IP publique de n'importe quel node

---

## Installer Traefik (avec `LoadBalancer`)

Installer le Helm chart :
```bash
helm upgrade --install --namespace traefik --create-namespace \
  --repo https://traefik.github.io/charts traefik traefik \
  --version 37.1.2 \
  --set providers.kubernetesGateway.enabled=true \
  --set gateway.listeners.web.namespacePolicy.from=All \
  #
```

Nous nous connecterons via l'adresse IP publique du load balancer :
```bash
kubectl get services --namespace traefik
```

---

## Installer Traefik (avec `hostPort`)

Installer le Helm chart :
```bash
helm upgrade --install --namespace traefik --create-namespace \
  --repo https://traefik.github.io/charts traefik traefik \
  --version 37.1.2 \
  --set deployment.kind=DaemonSet \
  --set ports.web.hostPort=80 \
  --set ports.websecure.hostPort=443 \
  --set service.type=ClusterIP \
  --set providers.kubernetesGateway.enabled=true \
  --set gateway.listeners.web.namespacePolicy.from=All \
  #
```

Nous nous connecterons en utilisant l'adresse IP publique de n'importe quel node du cluster.

---

class: extra-details

## Taints et tolerations

- Par défaut, les Pods Traefik respectent les taints des nodes

- Si certains nodes portent des taints (par ex. les nodes du control plane), des tolerations peuvent être nécessaires

  (si nous souhaitons exécuter Traefik sur l'ensemble des nodes)

- L'ajout des tolerations correspondantes est laissé en exercice au lecteur !

---

class: extra-details

## Rolling updates avec `hostPort`

- Il est impossible d'avoir deux pods utilisant le même `hostPort` sur le même node

- Il est donc essentiel de prêter attention aux paramètres de rolling update du `DaemonSet`

- Si `maxUnavailable` est supérieur à zéro :

  - les anciens pods sont arrêtés en premier

  - les nouveaux pods démarrent sans encombre

  - une courte interruption de service survient

- Si `maxSurge` est supérieur à zéro :

  - de nouveaux pods sont créés mais ne peuvent pas démarrer (le `hostPort` étant déjà occupé)

  - les anciens pods restent actifs et le rolling update n'avance pas

---

## Tester notre Gateway controller

- Envoyez une requête de test à Traefik

  (par ex. avec `curl http://<ipaddress>`)

- Pour l'instant, nous devrions obtenir une erreur `404 not found`

  (puisqu'aucune route n'est encore configurée)

---

## Une route HTTP basique

- Créez un conteneur HTTP de base et exposez-le via un Service ; par ex. :
  ```bash
  kubectl create deployment blue --image jpetazzo/color --port 80
  kubectl expose deployment blue
  ```

---

## Une route HTTP basique

- Créez une `HTTPRoute` avec le manifeste YAML suivant :
  ```yaml
  apiVersion: gateway.networking.k8s.io/v1
  kind: HTTPRoute
  metadata:
    name: blue
  spec:
    parentRefs:
    - group: gateway.networking.k8s.io
      kind: Gateway
      name: traefik-gateway
      namespace: traefik
    rules:
    - backendRefs:
      - name: blue
        port: 80
  ```

- Notre commande `curl` devrait désormais renvoyer une réponse du pod `blue`

---

class: extra-details

## Tableau de bord (dashboard) Traefik

- Par défaut, Traefik expose un dashboard

  (sur un port distinct de celui utilisé pour le trafic applicatif standard)

- Pour y accéder :
  ```bash
  kubectl port-forward --namespace traefik daemonset/traefik 1234:8080
  ```

  (remplacez `daemonset` par `deployment` si nécessaire)

- Connectez-vous ensuite à http://localhost:1234/dashboard/ (attention au `/` final !)

---

## `Core` vs `Extended` vs `Implementation-specific`

- Tout contrôleur Gateway doit impérativement supporter les fonctionnalités `Core`

- Certaines fonctionnalités optionnelles appartiennent à l'ensemble `Extended` :

  - leur implémentation dépend du contrôleur choisi

  - mais leur spécification fait partie intégrante de la définition de l'API

- Les contrôleurs Gateway peuvent également proposer des fonctionnalités `Implementation-specific`

  (= extensions propriétaires)

- Dans les diapositives suivantes, nous identifierons chaque fonctionnalité avec la mention `Core` ou `Extended`

---

## `HTTPRoute.spec.rules[].matches`

Certains champs relèvent du périmètre `Core` ; d'autres du périmètre `Extended`.

```yaml
match:
  path:                   # Core
    value: /hello
    type: PathPrefix      # valeur par défaut ; peut également être "Exact"
  headers:                # Core
  - name: x-custom-header
    value: foo
  queryparams:            # Extended
  - type: Exact           # accepte aussi des valeurs spécifiques à l'implémentation, par ex. Regex
    name: product
    value: pizza
  method: GET             # Extended
```

---

## `HTTPRoute.spec.rules[].filters.*HeaderModifier`

`RequestHeaderModifier` relève du niveau `Core`

`ResponseHeaderModifier` relève du niveau `Extended`

```yaml
type: RequestHeaderModifier       # ou ResponseHeaderModifier
requestHeaderModifier:            # ou responseHeaderModifier
  set:                            # remplace un header existant
  - name: x-my-header
    value: hello
  add:                            # ajoute à un header existant
  - name: x-my-header             # (ajoute une virgule s'il est déjà défini)
    value: hello
  remove:
  - x-my-header
```

---

## `HTTPRoute.spec.rules[].filters.RequestRedirect`

```yaml
type: RequestRedirect
requestRedirect:
  scheme: https                     # http ou https
  hostname: newxyz.example.com
  path: /new
  port: 8080
  statusCode: 302                   # default=302; peut valoir 301 302 303 307 308
```

Tous les champs sont optionnels. Laisser un champ vide signifie "conserver tel quel".

Notez que bien que `RequestRedirect` soit `Core`, certaines options sont `Extended` !

(Consultez la [spécification de l'API pour les détails][http-request-redirect].)

[http-request-redirect]: https://gateway-api.sigs.k8s.io/reference/spec/#httprequestredirectfilter

---

## `HTTPRoute.spec.rules[].filters.URLRewrite`

```yaml
type: URLRewrite
urlRewrite:
  hostname: newxyz.example.com
  path: /new
```

`hostname` réécrit l'en-tête HTTP `Host:`.

Il s'agit d'une fonctionnalité `Extended`.

Elle est incompatible avec `HTTPRequestRedirect`.

---

## `HTTPRoute.spec.rules[].filters.RequestMirror`

Il s'agit d'une fonctionnalité `Extended`. Elle transmet une copie de l'ensemble (ou d'une fraction) des requêtes vers un autre backend. Les réponses du backend miroir sont ignorées.

```yaml
type: RequestMirror
requestMirror:
  percent: 10
  fraction:
    numerator: 1
    denominator: 10
  backendRef:
    group: "" # valeur par défaut
    kind: Service # valeur par défaut
    name: log-some-requests
    namespace: my-observability-namespace # par défaut le même namespace
    port: 80
  hostname: newxyz.example.com
```

Indiquez `percent` ou `fraction`, mais pas les deux. Si aucun n'est spécifié, toutes les requêtes sont dupliquées (mirroring).

---

## Autres routes

- `GRPCRoute` permet d'utiliser les services et méthodes GRPC pour router les requêtes

  *utile si vous utilisez GRPC ; sinon, vous pouvez l'ignorer !*

- `TLSRoute` permet d'utiliser l'en-tête SNI pour router les requêtes (sans déchiffrer le trafic)

  *utile pour héberger plusieurs services TLS sur une adresse unique avec un chiffrement de bout en bout*

- `TCPRoute` permet de router des connexions TCP

  *utile pour mutualiser plusieurs protocoles sur la même adresse, par ex. HTTP+HTTPS+SSH*

- `UDPRoute` permet de router des paquets UDP

  *même principe, par ex. pour DNS/UDP, DNS/TCP, DNS/HTTPS*

---

## `gateway.spec.listeners.allowedRoutes`

- Avec `Ingress`, toute ressource `Ingress` peut intercepter le trafic

- Cela peut poser problème si par ex. un environnement de dev/staging crée par inadvertance (ou par malveillance) un `Ingress` avec un nom d'hôte de production

- La Gateway API introduit des garde-fous

- Une `Gateway` peut définir si elle accepte d'être référencée par des routes :

  - issues de tous les namespaces (comme avec `Ingress`)

  - issues uniquement du même namespace

  - issues uniquement de namespaces spécifiques correspondant à un selector

- C'est la raison pour laquelle nous avons spécifié `gateway.listeners.web.namespacePolicy.from=All` lors du déploiement de Traefik

???

:EN:- The Gateway API
:FR:- La Gateway API