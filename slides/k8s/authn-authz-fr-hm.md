# Authentication and authorization

- Dans cette section, nous allons :

  - définir l'authentification et l'autorisation

  - expliquer comment elles sont implémentées dans Kubernetes

  - aborder les tokens, les certificats, les service accounts, le RBAC...

- Mais tout d'abord : pourquoi avons-nous besoin de tout cela ?

---

## Le besoin d'une sécurité à granularité fine

- L'API Kubernetes ne devrait être accessible qu'aux utilisateurs identifiés

  - nous ne voulons pas d'« accès invité » (sauf cas rarissimes)

  - nous ne voulons pas que des inconnus consomment nos ressources de calcul, suppriment nos applications...

  - nos clés et mots de passe ne doivent pas être exposés au public

- Les utilisateurs auront souvent des droits d'accès différents

  - le cluster admin (semblable au « root » UNIX) peut tout faire

  - un développeur peut avoir accès à des ressources spécifiques, ou à un namespace donné

  - l'équipe de supervision peut avoir un accès en lecture seule à la *plupart* des ressources

---

## Exemple : load balancer HTTP sur mesure

- Imaginons que nous ayons un load balancer HTTP sur mesure pour plusieurs applications

- Chaque application dispose de sa propre ressource *Deployment*

- Par défaut, les applications sont « en veille » et scalées à zéro

- Lorsqu'une requête arrive, l'application correspondante est réveillée

- Après une période d'inactivité, l'application est de nouveau scalée à zéro

- Ce load balancer HTTP nécessite un accès à l'API (pour scaler up/down)

- Que se passe-t-il si *une vulnérabilité survient soudainement* ?

---

## Conséquences d'une vulnérabilité

- Si le load balancer HTTP dispose des mêmes accès à l'API que nous :

  *compromission totale du cluster (fuite de données facile, cryptojacking...)*

- Si le load balancer HTTP possède les permissions `update` sur les Deployments :

  *défaçage (facile), MITM / usurpation d'identité (difficulté moyenne à élevée)*

- Si le load balancer HTTP a uniquement la permission de `scale` les Deployments :

  *déni de service (DoS)*

- Toutes ces issues sont mauvaises, mais certaines sont bien pires que d'autres

---

## Définitions

- Authentification = vérifier l'identité d'une personne

  Sur un système UNIX, nous pouvons nous authentifier par login+mot de passe, clés SSH...

- Autorisation = lister ce qu'elle est autorisée à faire

  Sur un système UNIX, cela inclut les permissions de fichiers, les entrées sudoers...

- Souvent abrégées en « authn » et « authz »

- Dans les architectures bien modulaires, ces deux aspects sont découplés

   (nous pouvons ainsi par ex. changer un mot de passe ou une clé SSH sans devoir réinitialiser les droits d'accès)

---

## Authentification dans Kubernetes

- Lorsque l'API server reçoit une requête, il tente de l'authentifier

  (il examine les en-têtes, les certificats... tout élément disponible)

- De nombreuses méthodes d'authentification sont supportées et peuvent être employées simultanément

  (nous les détaillerons sur la slide suivante)

- Le rôle de la méthode d'authentification consiste à produire :

  - le user name
  - le user ID
  - une liste de groupes

- L'API server n'interprète pas ces données ; ce sera le rôle des *authorizers*

---

## Qui sommes-nous ?

- Nous pouvons interroger l'API Kubernetes pour savoir qui nous sommes

  (ou plutôt : qui elle pense que nous sommes)

.lab[

- Essayez la commande suivante sur votre cluster Kubernetes :
  ```bash
  kubectl auth whoami
  ```

]

---

## Méthodes d'authentification

- Certificats clients TLS

  (c'est la méthode par défaut pour les clusters initialisés avec `kubeadm`)

- Bearer tokens

  (un token secret transmis dans les en-têtes HTTP de la requête)

- [HTTP basic auth](https://en.wikipedia.org/wiki/Basic_access_authentication)

  (transmettant utilisateur et mot de passe dans un en-tête HTTP ; [dépréciée depuis Kubernetes 1.19](https://github.com/kubernetes/kubernetes/pull/89069))

- Proxy d'authentification

  (positionné devant l'API et injectant des en-têtes de confiance)

---

## Requêtes anonymes

- Si une méthode d'authentification *rejette* explicitement une requête, l'accès est refusé

  (code HTTP `401 Unauthorized`)

- Si une requête n'est ni rejetée ni validée par aucun mécanisme, elle est considérée comme anonyme

  - le nom d'utilisateur est `system:anonymous`

  - la liste des groupes est `[system:unauthenticated]`

- Par défaut, l'utilisateur anonyme ne dispose d'aucun privilège

  (c'est le statut obtenu si vous faites un simple `curl` sur l'API Kubernetes)

---

## Authentification par certificats TLS

- Activée sur la quasi-totalité des déploiements Kubernetes

- Le nom d'utilisateur est déterminé par le champ `CN` du certificat client

- Les groupes sont définis par les champs `O` du certificat client

- Du point de vue de l'API Kubernetes, les utilisateurs n'existent pas en tant qu'objets

  (il n'y a pas de ressource avec `kind: User`)

- L'API Kubernetes peut être configurée pour s'appuyer sur votre CA interne afin de valider les certificats clients

---

class: extra-details

## Authentification pour le kubelet

- Sur la plupart des clusters, les kubelets s'authentifient via des certificats

  (`O=system:nodes`, `CN=system:node:name-of-the-node`)

- L'API Kubernetes peut agir en tant qu'autorité de certification (CA)

  (en encapsulant un CSR X509 dans une ressource CertificateSigningRequest)

- Cela permet aux kubelets de renouveler automatiquement leurs propres certificats

- Cela peut aussi servir à émettre des certificats utilisateurs

  (mais manque de souplesse ; par ex. la durée de validité n'est pas personnalisable)

---

## Les certificats utilisateurs en pratique

- L'API server de Kubernetes ne prend pas en charge la révocation de certificats

  (voir l'issue [#18982](https://github.com/kubernetes/kubernetes/issues/18982))

- Par conséquent, il n'existe pas de moyen simple de révoquer l'accès d'une personne

  (si sa clé privée est compromise, ou en cas de départ de l'organisation)

- Générez des certificats à courte durée de vie si vous les utilisez pour authentifier des utilisateurs !

  (courte durée de vie = quelques heures)

- Cela peut être facilité par des outils comme Vault, cert-manager...

---

## Que faire en cas de compromission d'un certificat ?

- Option 1 : attendre l'expiration du certificat

  (ce qui rend les certificats à courte durée de vie très avantageux !)

- Option 2 : révoquer les accès de l'utilisateur et des groupes de ce certificat

  - si l'utilisateur était `bob.smith`, créer un nouvel utilisateur `bob.smith.2`

  - si Bob appartenait au groupe `dev`, créer un nouveau groupe `dev.2`

  - reconnaissons qu'il s'agit d'une solution très peu pratique !

- Option 3 : recréer une nouvelle CA et réémettre l'ensemble des certificats

  - reconnaissons qu'il s'agit d'une solution encore pire !

---

## Authentification par tokens

- Les tokens sont transmis via les en-têtes HTTP :

  `Authorization: Bearer et-voici-le-token`

- Les tokens peuvent être validés par plusieurs méthodes :

  - tokens statiques inscrits en dur dans un fichier sur l'API server

  - [bootstrap tokens](https://kubernetes.io/docs/reference/access-authn-authz/bootstrap-tokens/) (cas d'usage dédié à l'initialisation du cluster ou à l'intégration de nodes)

  - [tokens OpenID Connect](https://kubernetes.io/docs/reference/access-authn-authz/authentication/#openid-connect-tokens) (pour déléguer l'authentification à des fournisseurs compatibles OAuth2)

  - service accounts (ils méritent une attention particulière, voyons cela tout de suite !)

---

## Service accounts

- Un service account est une identité qui existe directement dans l'API Kubernetes

  (visible via des commandes comme `kubectl get serviceaccounts`)

- Les service accounts peuvent donc être créés / mis à jour dynamiquement

  (ils ne requièrent aucune édition manuelle de fichier ni de redémarrage de l'API server)

- Un service account peut être associé à un ensemble de secrets

  (du type de ceux que l'on liste avec `kubectl get secrets`)

- Les service accounts sont généralement employés pour accorder des permissions aux applications, services...

  (par opposition aux utilisateurs humains)

---

## Évolution des tokens de service account

- Dans Kubernetes 1.21 et versions supérieures, les pods utilisent des *bound service account tokens* :

  - ces tokens sont *liés* (bound) à un objet spécifique (par ex. un Pod)

  - ils sont automatiquement invalidés dès que l'objet est supprimé

  - ils expirent rapidement (par ex. 1 heure) et sont renouvelés (rotated) automatiquement

- Dans Kubernetes 1.24 et versions supérieures, les tokens non liés ne sont plus générés automatiquement

  - avant la 1.24, on voyait ces tokens non liés avec `kubectl get secrets`

  - à partir de la 1.24, ces tokens peuvent être créés via `kubectl create token`

  - ... ou à l'aide d'un Secret configuré avec [le type et l'annotation appropriés][create-token]

[create-token]: https://kubernetes.io/docs/reference/access-authn-authz/service-accounts-admin/#create-token

---

class: extra-details

## Vérifier notre méthode d'authentification

- Inspectons notre fichier kubeconfig

- Utilisons-nous un certificat, un token, ou un autre mécanisme ?

---

class: extra-details

## Inspecter un certificat

Si nous utilisons un certificat, exécutons la commande suivante :

```bash
kubectl config view \
        --raw \
        -o json \
        | jq -r .users[0].user[\"client-certificate-data\"] \
        | openssl base64 -d -A \
        | openssl x509 -text \
        | grep Subject:
```

Cette commande affiche les champs `CN` et `O` de notre certificat.

---

class: extra-details

## Analyse détaillée de la commande

- `kubectl config view` affiche la configuration utilisateur Kubernetes
- `--raw` inclut les données des certificats (masquées par REDACTED par défaut)
- `-o json` formate la sortie en JSON
- `| jq ...` extrait le champ contenant le certificat utilisateur (encodé en base64)
- `| openssl base64 -d -A` décode le format base64 (pour obtenir un fichier PEM)
- `| openssl x509 -text` analyse le certificat et l'affiche en texte brut
- `| grep Subject:` filtre sur la ligne qui nous intéresse

→ Nous sommes l'utilisateur `kubernetes-admin`, membre du groupe `system:masters`.

(Nous verrons plus loin comment et pourquoi cela nous confère l'ensemble de nos permissions actuelles.)

---

class: extra-details

## Inspecter un token

Si nous utilisons un token, exécutons la commande suivante :

```bash
kubectl config view \
        --raw \
        -o json \
        | jq -r .users[0].user.token \
        | base64 -d \
        | cut -d. -f2 \
        | base64 -d \
        | jq .
```

Si notre token est un token JWT / OIDC, cette commande affichera son contenu déchiffré.

---

class: extra-details

## Autres méthodes d'authentification

- Autres types de tokens

  - ces tokens sont généralement plus courts que les tokens JWT ou OIDC

  - il est habituellement impossible d'en extraire des informations en clair

- Plugins

  - certains clusters utilisent des plugins externes `exec`

  - ces plugins s'appuient typiquement sur des clés d'API pour générer ou acquérir des tokens

  - exemple : l'authentificateur AWS EKS fonctionne selon ce principe

---

class: extra-details

## L'authentification par token en pratique

- Nous allons lister les service accounts existants

- Puis nous allons extraire le token d'un service account particulier

- Et nous utiliserons ce token pour nous authentifier auprès de l'API

---

class: extra-details

## Lister les service accounts

.lab[

- Le nom de ressource est `serviceaccount`, ou `sa` en version abrégée :
  ```bash
  kubectl get sa
  ```

]

Il ne devrait y avoir qu'un seul service account dans le namespace default : `default`.

---

class: extra-details

## Identifier le secret

.lab[

- Lister les secrets associés au service account `default` :
  ```bash
  kubectl get sa default -o yaml
  SECRET=$(kubectl get sa default -o json | jq -r .secrets[0].name)
  ```

]

Il devrait se nommer `default-token-XXXXX`.

Sous Kubernetes 1.24 et versions supérieures, ce Secret n'existera pas.
<br/>
Dans ce cas, générez un token avec `kubectl create token default`.

---

class: extra-details

## Extraire le token

- Le token est stocké dans le secret, encodé en base64

.lab[

- Afficher le secret :
  ```bash
  kubectl get secret $SECRET -o yaml
  ```

- Extraire le token et le décoder :
  ```bash
  TOKEN=$(kubectl get secret $SECRET -o json \
          | jq -r .data.token | openssl base64 -d -A)
  ```

]

---

class: extra-details

## Utiliser le token

- Envoyons une requête vers l'API, d'abord sans token, puis avec le token

.lab[

- Récupérer l'adresse ClusterIP du service `kubernetes` :
  ```bash
  kubectl get svc kubernetes
  API=$(kubectl get svc kubernetes -o json | jq -r .spec.clusterIP)
  ```

- Se connecter sans le token :
  ```bash
  curl -k https://$API
  ```

- Se connecter avec le token :
  ```bash
  curl -k -H "Authorization: Bearer $TOKEN" https://$API
  ```

]

---

class: extra-details

## Résultats obtenus

- Dans les deux cas, nous obtiendrons une erreur « Forbidden »

- Sans authentification, l'utilisateur est identifié comme `system:anonymous`

- Avec authentification, il apparaît sous l'identité `system:serviceaccount:default:default`

- L'API nous « perçoit » comme un utilisateur différent

- Cependant, aucun de ces utilisateurs n'a de droits accordés, nous ne pouvons donc rien exécuter

- Changeons cela !

---

## Autorisation dans Kubernetes

- Il existe plusieurs manières d'octroyer des permissions dans Kubernetes, appelées [authorizers](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#authorization-modules) :

  - [Node Authorization](https://kubernetes.io/docs/reference/access-authn-authz/node/) (utilisé en interne par le kubelet ; nous pouvons l'ignorer ici)

  - [Attribute-based access control](https://kubernetes.io/docs/reference/access-authn-authz/abac/) (puissant mais complexe et statique ; à ignorer également)

  - [Webhook](https://kubernetes.io/docs/reference/access-authn-authz/webhook/) (chaque requête API est transmise à un service externe pour validation)

  - [Role-based access control](https://kubernetes.io/docs/reference/access-authn-authz/rbac/) (associe dynamiquement des permissions aux utilisateurs)

- C'est ce dernier mécanisme qui nous intéresse, communément abrégé en RBAC

---

## Role-based access control

- Le RBAC permet de définir des permissions extrêmement fines

- Les permissions sont exprimées sous forme de *règles* (rules)

- Une règle est une combinaison de :

  - [verbes](https://kubernetes.io/docs/reference/access-authn-authz/authorization/#determine-the-request-verb) tels que create, get, list, update, delete...

  - ressources (au sens « ressource d'API », comme pods, nodes, services...)

  - noms de ressources (pour cibler par ex. un pod spécifique au lieu de tous les pods)

  - dans certains cas, des [sous-ressources](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#referring-to-resources) (par ex. les logs sont des sous-ressources des pods)

---

class: extra-details

## Lister tous les verbes possibles

- L'API Kubernetes est auto-documentée

- Nous pouvons l'interroger pour savoir quelles ressources, sous-ressources et verbes existent

- Une méthode consiste à utiliser :

  - `kubectl get --raw /api/v1` (pour les ressources core sous `apiVersion: v1`)

  - `kubectl get --raw /apis/<group>/<version>` (pour les autres ressources)

- La réponse JSON peut être formatée par ex. avec `jq` pour en faciliter la lecture

---

class: extra-details

## Exemples

- Lister tous les verbes à travers l'ensemble des ressources `v1`

  ```bash
  kubectl get --raw /api/v1 | jq -r .resources[].verbs[] | sort -u
  ```

- Lister l'ensemble des ressources et sous-ressources sous `apps/v1`

  ```bash
  kubectl get --raw /apis/apps/v1 | jq -r .resources[].name
  ```

- Lister quels verbes sont disponibles sur chaque ressource de `networking.k8s.io`

  ```bash
  kubectl get --raw /apis/networking.k8s.io/v1 | \
          jq -r '.resources[] | .name + ": " + (.verbs | join(", "))'
  ```

---

## Des règles aux rôles et aux rolebindings

- Un *Role* est un objet d'API contenant une liste de *règles*

  Exemple : le rôle « external-load-balancer-configurator » peut :
  - [list, get] sur les ressources [endpoints, services, pods]
  - [update] sur les ressources [services]

- Un *RoleBinding* associe un rôle à un utilisateur

  Exemple : le rolebinding « external-load-balancer-configurator » :
  - associe l'utilisateur « external-load-balancer-configurator »
  - avec le rôle « external-load-balancer-configurator »

- Oui, utilisateurs, rôles et rolebindings peuvent partager un nom strictement identique

- C'est une excellente pratique pour les liaisons 1-1-1 ; moins recommandée pour du 1-N

---

## Permissions à l'échelle du cluster

- Les ressources d'API Role et RoleBinding s'appliquent aux objets au sein d'un namespace donné

- Nous pouvons également définir des ressources ClusterRole et ClusterRoleBinding

- Il s'agit d'un sur-ensemble (superset), nous permettant de :

  - définir des actions sur des objets cluster-wide (comme les nodes)

  - opérer de manière transverse à tous les namespaces

- On peut créer des ressources Role et RoleBinding à l'intérieur d'un namespace

- Les ressources ClusterRole et ClusterRoleBinding ont une portée globale

---

## Pods et service accounts

- Un pod peut être rattaché à un service account

  - par défaut, il est associé au service account `default`

  - comme vu précédemment, ce service account ne dispose d'aucune permission par défaut

- Le token associé est exposé directement dans le système de fichiers du pod

  (sous `/var/run/secrets/kubernetes.io/serviceaccount/token`)

- L'outillage standard de Kubernetes (tel que `kubectl`) le recherche automatiquement à cet emplacement

- Ainsi, les outils Kubernetes exécutés à l'intérieur d'un pod utiliseront automatiquement le service account

---

## En pratique

- Nous allons exécuter un pod

- Ce pod utilisera le service account par défaut de son namespace

- Nous vérifierons nos permissions sur l'API

  (il ne devrait y en avoir aucune)

- Puis nous lierons un rôle à ce service account

- Enfin, nous vérifierons que les permissions correspondantes nous ont bien été accordées

---

## Exécuter un pod

- Nous utiliserons [Nixery](https://nixery.dev/) pour exécuter un pod intégrant `curl` et `kubectl`

- Nixery génère automatiquement des images contenant les paquets demandés

.lab[

- Lancer notre pod :
  ```bash
  kubectl run eyepod --rm -ti --restart=Never \
          --image nixery.dev/shell/curl/kubectl -- bash
  ```

]

---

## Vérifier nos permissions

- À ce stade, nous ne devrions avoir aucune permission sur l'API

.lab[

- Vérifier nos permissions avec `kubectl` :
  ```bash
  kubectl get pods
  ```

]

- Nous devrions recevoir un message indiquant que notre service account
  n'a pas l'autorisation de lister les « pods » dans le namespace courant

- Nous pouvons aussi émettre des requêtes directement vers l'API server

  (utilisez `kubectl -v6` pour visualiser l'URI exacte de la requête !)

---

## Lier un rôle au service account

- Lier un rôle = créer un objet *RoleBinding*

- Nous allons nommer cet objet `can-view`

  (mais nous pourrions tout aussi bien l'appeler `view` ou n'importe quel autre nom)

.lab[

- Créer le nouveau RoleBinding :
  ```bash
  kubectl create rolebinding can-view \
          --clusterrole=view \
          --serviceaccount=default:default
  ```

]

Il est important de noter plusieurs détails cruciaux concernant ces flags...

---

## Roles vs Cluster Roles

- Nous avons spécifié `--clusterrole=view`

- Que se serait-il passé si nous avions utilisé `--role=view` ?

  - nous aurions lié le rôle `view` issu du namespace local
    <br/>(au lieu du ClusterRole global `view`)

  - la commande se serait exécutée avec succès (sans erreur)

  - mais ultérieurement, nos requêtes d'API auraient été rejetées

- Il s'agit d'un choix de conception délibéré

  (nous pouvons référencer des rôles qui n'existent pas encore, puis les créer ou les mettre à jour ultérieurement)

---

## Users vs Service Accounts

- Nous avons spécifié `--serviceaccount=default:default`

- Que se serait-il passé si nous avions utilisé `--user=default:default` ?

  - nous aurions lié le rôle à un utilisateur classique au lieu d'un service account

  - là encore, la commande se serait exécutée sans erreur

  - ... mais nos requêtes d'API auraient été refusées par la suite

- Que signifie le préfixe `default:` ?

  - il désigne le namespace du service account

  - oui, il pourrait être déduit du contexte, mais... `kubectl` l'exige explicitement

---

## Vérifier nos nouvelles permissions

- Nous devrions être en mesure de *consulter* (view) les ressources, mais pas de les *modifier* (edit)

.lab[

- Vérifier nos permissions avec `kubectl` :
  ```bash
  kubectl get pods
  ```

- Tenter de créer une ressource :
  ```bash
  kubectl create deployment can-i-do-this --image=nginx
  ```

- Sortir du conteneur avec `exit` ou `^D`

<!-- ```key ^D``` -->

]

---

class: extra-details

## `kubectl run --serviceaccount`

- `kubectl run` possède également un flag `--serviceaccount`

- ... Mais il est supposé être déprécié prochainement

  (voir [kubernetes/kubernetes#99732](https://github.com/kubernetes/kubernetes/pull/99732) pour plus de détails)

- Il est possible de spécifier le service account au moyen d'un override :
  ```bash
  kubectl run my-pod -ti --image=alpine --restart=Never \
          --overrides='{ "spec": { "serviceAccountName" : "my-service-account" } }'
  ```

---

## `kubectl auth` et autres outils CLI

- La commande `kubectl auth can-i` permet de déterminer :

  - si nous pouvons exécuter une action donnée

  - si quelqu'un d'autre peut exécuter cette action

  - quelles actions nous sommes autorisés à accomplir

- Il existe également d'autres outils très utiles pour manipuler le RBAC

- Faisons un tour d'horizon rapide !

---

## `kubectl auth can-i dothis onthat`

- Ces commandes retournent une réponse de type `yes`/`no` :

  ```bash
  kubectl auth can-i list nodes
  kubectl auth can-i create pods
  kubectl auth can-i get pod/name-of-pod
  kubectl auth can-i get /url-fragment-of-api-request/
  kubectl auth can-i '*' services
  kubectl auth can-i get coffee
  kubectl auth can-i drink coffee
  ```

- Le système RBAC est très flexible

- Nous pouvons vérifier des permissions sur des ressources qui n'existent pas encore (par ex. des CRDs)

- Nous pouvons vérifier des permissions pour des actions totalement arbitraires

---

## `kubectl auth can-i ... --as someoneelse`

- Nous pouvons tester les permissions pour le compte d'autres utilisateurs

  ```bash
  kubectl auth can-i list nodes \
          --as some-user
  kubectl auth can-i list nodes \
          --as system:serviceaccount:<namespace>:<name-of-service-account>
  ```

- Nous pouvons aussi employer `--as-group` pour évaluer les permissions des membres d'un groupe

- `--as` et `--as-group` s'appuient sur l'*impersonation API*

- Ces flags peuvent être utilisés avec de nombreuses autres commandes `kubectl`

  (pas seulement avec `auth can-i`)

---

## `kubectl auth can-i --list`

- Nous pouvons lister l'ensemble des actions qui nous sont accessibles :

  ```bash
  kubectl auth can-i --list
  ```

- ... Ou accessibles à une autre entité (avec `--as SomeOtherUser`)

- C'est extrêmement utile pour auditer les utilisateurs ou service accounts dotés de permissions trop larges

  (ou lorsque l'on recherche des vecteurs d'exploitation d'une faille de sécurité !)

- Pour en savoir plus sur les attaques Kubernetes et les modèles de menaces autour du RBAC :

  📽️ [Hacking into Kubernetes Security for Beginners](https://www.youtube.com/watch?v=mLsCm9GVIQg)
  par [V Körbes](https://twitter.com/veekorbes)
  et [Tabitha Sable](https://twitter.com/TabbySable)

---

class: extra-details

## Autres outils utiles

- À des fins d'audit, il est souvent nécessaire de savoir qui peut exécuter quelles actions

- Plusieurs outils peuvent nous y aider sous la forme de plugins `kubectl` :

  - `kubectl who-can` / [kubectl-who-can](https://github.com/aquasecurity/kubectl-who-can) par Aqua Security

  - `kubectl access-matrix` / [Rakkess (Review Access)](https://github.com/corneliusweig/rakkess) par Cornelius Weig

  - `kubectl rbac-lookup` / [RBAC Lookup](https://github.com/FairwindsOps/rbac-lookup) par FairwindsOps

  - `kubectl rbac-tool` / [RBAC Tool](https://github.com/alcideio/rbac-tool) par insightCloudSec

- Les plugins `kubectl` s'installent et se gèrent facilement avec `krew`

- Ils peuvent également être installés et exécutés en tant que binaires autonomes

---

class: extra-details

## D'où provient ce rôle `view` ?

- Kubernetes intègre nativement un ensemble de ClusterRoles prêts à être assignés aux utilisateurs

- `cluster-admin` peut absolument *tout* faire (équivalent de `root` sous UNIX)

- `admin` peut faire *presque tout* (sauf par ex. modifier les quotas et limites de ressources)

- `edit` est proche de `admin`, mais ne permet ni de consulter ni de modifier les permissions

- `view` dispose d'un accès en lecture seule à la plupart des ressources, hors permissions et secrets

*Dans de nombreuses situations, ces rôles répondront à tous vos besoins.*

*Vous avez également la possibilité de les personnaliser !*

---

class: extra-details

## Personnaliser les rôles par défaut

- Si vous devez *ajouter* des permissions à ces rôles par défaut (ou à d'autres),
  <br/>
  vous pouvez le faire via le mécanisme de [ClusterRole Aggregation](https://kubernetes.io/docs/reference/access-authn-authz/rbac/#aggregated-clusterroles)

- Cela s'effectue en créant un ClusterRole comportant les labels suivants :
  ```yaml
    metadata:
      labels:
        rbac.authorization.k8s.io/aggregate-to-admin: "true"
        rbac.authorization.k8s.io/aggregate-to-edit: "true"
        rbac.authorization.k8s.io/aggregate-to-view: "true"
  ```

- Les permissions de ce ClusterRole seront automatiquement injectées dans `admin`/`edit`/`view` respectivement

---

class: extra-details

## Quand utiliser l'agrégation de rôles ?

- Par défaut, les CRDs ne sont pas incluses dans `view` / `edit` / etc.

  (Kubernetes ne peut pas deviner lesquelles sont sensibles sur le plan de la sécurité et lesquelles ne le sont pas)

- Si nous modifions directement `view` / `edit` / etc., nos modifications entreront en conflit

  (imaginez si deux CRDs différentes fournissent chacune un ClusterRole `view` personnalisé)

- L'utilisation de rôles agrégés permet d'enrichir les rôles par défaut sans modifier leur définition initiale

---

class: extra-details

## Fonctionnement de l'agrégation

- Les rôles correspondants disposeront de règles `aggregationRules` structurées ainsi :

   ```yaml
    aggregationRule:
      clusterRoleSelectors:
      - matchLabels:
          rbac.authorization.k8s.io/aggregate-to-view: "true"
   ```

- Nous pouvons définir nos propres rôles personnalisés avec leurs propres règles d'agrégation

---

class: extra-details

## D'où proviennent nos permissions ?

- Lors de nos interactions avec l'API Kubernetes, nous utilisons un certificat client

- Nous avons vu plus haut que ce certificat client contenait :

  `CN=kubernetes-admin` et `O=system:masters`

- Recherchons ces valeurs dans les ClusterRoleBindings existants :
  ```bash
  kubectl get clusterrolebindings -o yaml |
    grep -e kubernetes-admin -e system:masters
  ```

  (`system:masters` doit ressortir, mais pas `kubernetes-admin`.)

- D'où provient cette correspondance ?

---

class: extra-details

## Le groupe `system:masters`

- En examinant la sortie de `kubectl get clusterrolebindings -o yaml`, nous trouvons l'explication !

- Cette correspondance se trouve dans le binding `cluster-admin` :
  ```bash
  kubectl describe clusterrolebinding cluster-admin
  ```

- Ce binding associe le groupe `system:masters` au cluster role `cluster-admin`

- Et `cluster-admin` correspond concrètement au compte `root` :
  ```bash
  kubectl describe clusterrole cluster-admin
  ```

---

## `list` vs. `get`

⚠️ `list` accorde les permissions de lecture sur les ressources !

- Il est impossible d'accorder la permission de lister des ressources sans autoriser également leur lecture

- Cela a des implications directes par ex. pour les Secrets

  (si un contrôleur doit pouvoir énumérer les Secrets, il aura la possibilité d'en lire le contenu)

???

:FR:- Identification et droits d'accès dans Kubernetes
:FR:- Mécanismes d'identification par jetons et certificats
:FR:- Le modèle RBAC *(Role-Based Access Control)*
:FR:- Restreindre les permissions grâce aux *Service Accounts*
:FR:- Comprendre les *Roles*, *Cluster Roles*, *Role Bindings*, etc.