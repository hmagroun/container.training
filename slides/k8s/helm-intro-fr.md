# Gérer des stacks avec Helm

- Helm est (en quelque sorte !) un package manager pour Kubernetes

- Nous pouvons l'utiliser pour :

  - trouver des packages existants (appelés "charts") créés par d'autres

  - installer ces packages, en les configurant pour notre environnement spécifique

  - packager nos propres composants (pour distribution publique ou pour usage interne)

  - gérer le cycle de vie de ces installations (rollback vers une version antérieure, etc.)

- C'est un projet "CNCF graduate", attestant d'un niveau élevé de maturité

  (nous y reviendrons plus en détail)

---

## Fonctionnalités de Helm

- Avec Helm, nous créons des "charts"

- Ces charts peuvent être utilisés en interne ou distribués publiquement

- Les charts publics peuvent être indexés via l'[Artifact Hub](https://artifacthub.io/)

- Cela nous permet de rechercher et d'installer les charts créés par d'autres

- Helm fournit également des mécanismes pour gérer le cycle de vie des déploiements :

  - garder une trace de ce qui est installé

  - mettre à jour les versions, modifier les paramètres, faire un rollback, désinstaller

- De plus, même s'il ne s'agit pas de "l'unique" standard, c'est indéniablement "un" standard incontournable !

---

## Statut CNCF Graduation

- Le 30 avril 2020, Helm est devenu le 10e projet à être *graduated* au sein de la CNCF

  (aux côtés de Containerd, Prometheus et Kubernetes lui-même)

- Il s'agit d'une reconnaissance par la CNCF pour les projets qui

  *démontrent une adoption massive, un processus de gouvernance ouvert,
  <br/>
  et un engagement fort en faveur de la communauté, de la pérennité et de l'inclusivité.*

- Voir l'[annonce de la CNCF](https://www.cncf.io/announcement/2020/04/30/cloud-native-computing-foundation-announces-helm-graduation/)
  et l'[annonce de Helm](https://helm.sh/blog/celebrating-helms-cncf-graduation/)

- En d'autres termes : Helm est une technologie pérenne

---

## Concepts de Helm

- `helm` est un outil CLI

- Il est utilisé pour rechercher, installer et mettre à jour des *charts*

- Un chart est une archive contenant des bundles de templates YAML

- Les charts sont versionnés

- Les charts peuvent être stockés sur des repositories privés ou publics

---

## Différences entre charts et packages

- Un package système (deb, rpm...) contient des binaires, des bibliothèques, etc.

- Un chart contient des manifests YAML

  (les binaires, bibliothèques, etc. se trouvent dans les images référencées par le chart)

- Sur la plupart des distributions, un package ne peut être installé qu'une seule fois

  (installer une autre version remplace la version installée)

- Un chart peut être installé plusieurs fois

- Chaque installation est appelée une *release*

- Cela permet de déployer par exemple 10 instances de MongoDB

  (avec potentiellement des versions et des configurations différentes)

---

class: extra-details

## Un instant...

*Pourtant, sur mon système Debian, j'ai Python 2 **et** Python 3.
<br/>
J'ai également plusieurs versions du moteur de base de données Postgres !*

Tout à fait !

Mais ils utilisent des noms de packages distincts :

- `python2.7`, `python3.8`

- `postgresql-10`, `postgresql-11`

Bon à savoir : le package Postgres sous Debian intègre des mécanismes permettant de déployer plusieurs serveurs Postgres sur le même système, mais cela reste une exception (reposant sur un travail conséquent du mainteneur du package, et non sur les outils `dpkg` ou `apt` eux-mêmes).

---

## Helm 2 vs Helm 3

- Helm 3 est sorti le [13 novembre 2019](https://helm.sh/blog/helm-3-released/)

- Les charts restent compatibles entre Helm 2 et Helm 3

- La CLI est très similaire (avec des modifications mineures sur certaines commandes)

- La différence principale réside dans le fait que Helm 2 utilisait `tiller`, un composant côté serveur

- Helm 3 n'utilise plus `tiller` du tout, ce qui simplifie grandement l'architecture

- Si vous trouvez des références à `tiller` dans un tutoriel ou une documentation, cette dernière est obsolète !

---

class: extra-details

## Quel était le problème avec `tiller` ?

- Avec Helm 3 :

  - la CLI `helm` communique directement avec l'API Kubernetes

  - elle crée les ressources (deployments, services...) avec nos propres credentials / droits d'accès

- Avec Helm 2 :

  - la CLI `helm` communiquait avec `tiller`, en indiquant à `tiller` quelles actions exécuter

  - `tiller` communiquait ensuite avec l'API Kubernetes, en utilisant ses propres credentials

- Ce modèle indirect entraînait d'importantes complexités de gestion de permissions (RBAC)

- Il rendait également plus difficile l'intégration de Helm au sein d'autres outils

---

## Installer Helm

- Si la CLI `helm` n'est pas installée dans votre environnement, installez-la

.lab[

- Vérifier si `helm` est installé :
  ```bash
  helm
  ```

- S'il n'est pas installé, exécuter la commande suivante :
  ```bash
  curl https://raw.githubusercontent.com/kubernetes/helm/master/scripts/get-helm-3 \
  | bash
  ```

]

(Pour installer Helm 2, remplacer `get-helm-3` par `get`.)

---

## Charts et repositories

- Un *repository* (ou repo) est une collection de charts

- Il s'agit simplement d'un ensemble de fichiers

  (ils peuvent être hébergés sur un serveur HTTP statique ou dans un répertoire local)

- Nous pouvons déclarer des "repos" dans Helm en leur attribuant un alias

- Cet alias est utilisé pour faire référence aux charts de ce repository

  (par exemple, tenter d'installer `hello/world` signifie le chart `world` provenant du repo `hello` ; ce repo `hello` pointant par exemple vers https://blahblah.hello.io/charts/)

---

## Comment rechercher des charts

- Rendez-vous sur l'[Artifact Hub](https://artifacthub.io/packages/search?kind=0) (https://artifacthub.io)

- Ou utilisez `helm search hub ...` depuis la CLI

- Essayons de trouver un chart Helm pour l'application "OWASP Juice Shop" !

  (il s'agit d'une application de démo reconnue, utilisée pour des challenges de sécurité)

---

## Rechercher des charts depuis la CLI

- Nous pouvons utiliser `helm search hub <keyword>`

.lab[

- Rechercher l'application OWASP Juice Shop :
  ```bash
  helm search hub owasp juice
  ```

- Les URLs étant tronquées par défaut, essayez avec la sortie YAML :
  ```bash
  helm search hub owasp juice -o yaml
  ```

]

⚠️ Aucun résultat trouvé ? Consultez la section suivante !

---

En septembre 2026, l'ancien chart repository pour cette application a été supprimé.

Le chart Helm est désormais accessible via un registry OCI.

Pour une raison particulière, le chart n'est actuellement plus indexé sur l'Artifact Hub.

Tentez d'accéder à l'adresse du chart :

→ https://artifacthub.io/packages/helm/securecodebox/juice-shop

Si le lien ne fonctionne pas, il reste tout de même possible d'installer le chart.

Pour cela, la documentation du chart est nécessaire. Elle est accessible [ici][juice-shop-README].

[juice-shop-README]: https://github.com/secureCodeBox/secureCodeBox/blob/main/demo-targets/juice-shop/README.md

---

## Rechercher des charts sur le web

- Nous pouvons également utiliser le moteur de recherche d'Artifact Hub

.lab[

- Rendez-vous sur https://artifacthub.io/

- Dans la barre de recherche en haut, saisissez "owasp juice"

- Cliquez sur le résultat "juice-shop" (ne sélectionnez pas "multi-juicer" ou "juicy-ctf")

]

---

## Installer le chart

- Cliquez sur le bouton "Install" pour afficher les instructions

- Si la page n'est pas accessible, utilisez directement la commande ci-dessous !

.lab[

- Installer le chart :
  ```bash
  helm install my-juice-shop oci://ghcr.io/securecodebox/helm/juice-shop
  ```

]

---

## Charts et releases

- "Installer un chart" revient à créer une *release*

- Dans l'exemple précédent, la release a été nommée "my-juice-shop"

- Il est également possible d'utiliser `--generate-name` pour demander à Helm d'en générer le nom automatiquement

.lab[

- Lister les releases :
  ```bash
  helm list
  ```

- Vérifier que le Pod `my-juice-shop-...` est actif et en cours d'exécution :
  ```bash
  kubectl get pods
  ```

]

---

## Consulter les ressources d'une release

- Ce chart en particulier applique un label `release` sur l'ensemble de ses ressources

- Nous pouvons utiliser un selector pour filtrer et afficher ces ressources

.lab[

- Lister toutes les ressources créées par cette release :
  ```bash
  kubectl get all --selector=app.kubernetes.io/instance=my-juice-shop
  ```

]

Note : ce label n'a pas été injecté automatiquement par Helm.
<br/>
Il est défini au niveau du chart lui-même. En d'autres termes, tous les charts ne fourniront pas nécessairement ce label.

---

## Configurer une release

- Par défaut, `juice/juice-shop` crée un Service de type `ClusterIP`

- Nous souhaitons modifier cela pour un `NodePort`

- Nous pourrions exécuter `kubectl edit service my-juice-shop`, mais...

  ... nos modifications manuelles seraient écrasées lors de la prochaine mise à jour du chart !

- À la place, nous allons *définir une value* (set a value)

- Les values sont des paramètres exploitables par le chart pour ajuster son comportement

- Les values disposent de valeurs par défaut

- Chaque chart définit librement ses propres values et leurs valeurs par défaut

---

## Examiner les values configurables

- Nous pouvons inspecter un chart à l'aide de `helm show` ou `helm inspect`

.lab[

- Consulter le README de l'application :
  ```bash
  helm show readme juice/juice-shop
  ```

- Examiner les values et leurs valeurs par défaut :
  ```bash
  helm show values juice/juice-shop
  ```

]

Le fichier `values` peut comporter ou non des commentaires explicatifs utiles.

Le `readme` peut fournir ou non des explications détaillées concernant les values.

(En l'absence de documentation détaillée, aucune indication précise ne sera fournie sur l'usage des values !)

---

## Définir des values

- Les values peuvent être appliquées lors de l'installation d'un chart ou lors de son upgrade

- Nous allons mettre à jour la release `my-juice-shop` pour modifier le type du Service

.lab[

- Mettre à jour `my-juice-shop` :
  ```bash
  helm upgrade my-juice-shop oci://ghcr.io/securecodebox/helm/juice-shop \
       --set service.type=NodePort
  ```

]

Notez qu'il est indispensable de repréciser le chart utilisé (`juice/my-juice-shop`), même si l'objectif est simplement d'ajuster quelques variables.

Il est possible de configurer plusieurs values simultanément. Pour transmettre un grand nombre de paramètres, utilisez l'option `-f`/`--values` avec un fichier YAML contenant toutes les valeurs requises.

Toute variable non spécifiée conservera la valeur par défaut définie dans le chart.

---

## Se connecter à Juice Shop

- Vérifions l'application que nous venons d'installer

.lab[

- Identifier le port NodePort alloué au Service :
  ```bash
  kubectl get service my-juice-shop
  PORT=$(kubectl get service my-juice-shop -o jsonpath={..nodePort})
  ```

- Se connecter au service :
  ```bash
  curl localhost:$PORT/
  ```

