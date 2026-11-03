# Deploying with YAML

- Jusqu'à présent, nous avons créé des ressources à l'aide des commandes suivantes :

  - `kubectl run`

  - `kubectl create deployment`

  - `kubectl expose`

- Nous pouvons également créer des ressources directement à l'aide de manifests YAML

---

## Pourquoi utiliser le YAML ? (1/3)

- Certaines ressources ne peuvent pas être créées facilement avec `kubectl`

  (par ex. DaemonSets, StatefulSets, configurations de webhooks...)

- Certaines fonctionnalités et champs ne sont pas directement accessibles

  (par ex. limites de ressources, healthchecks, volumes...)

---

## Pourquoi utiliser le YAML ? (2/3)

- Créer une ressource complexe avec une seule commande simple :

  `kubectl create -f stuff.yaml`

- Créer *plusieurs* ressources avec une seule commande simple :

  `kubectl create -f more-stuff.yaml` ou `kubectl create -f directory-with-yaml/`

- Créer des ressources à partir d'un manifest distant :

  `kubectl create -f https://.../.../stuff.yaml`

- Créer et mettre à jour des ressources :

  `kubectl apply -f stuff.yaml`

---

## Pourquoi utiliser le YAML ? (3/3)

- Le YAML nous permet de travailler de façon *déclarative*

- Décrire ce que nous voulons déployer/exécuter sur Kubernetes

  ("desired state")

- Utiliser des outils tels que `kubectl`, Helm, kapp, Flux, ArgoCD... pour concrétiser cet état

  ("reconcile" l'état réel avec l'état désiré)

- Très similaire à des outils comme Terraform

---

class: extra-details

## Surcharges et `kubectl set`

Pour information...

- `kubectl create deployment ... --overrides '{...}'`

  *spécifie un patch qui sera appliqué par-dessus le YAML généré par `kubectl`*

- `kubectl set ...`

  *permet de modifier par exemple les images, les service accounts, les ressources, et bien plus encore*

---

## Diverses manières d'écrire du YAML

- À partir d'exemples issus de la documentation, de tutoriels, d'articles de blog, de LLMs...

  (l'option la plus simple pour démarrer)

- Exporter une ressource existante avec `kubectl get -o yaml ...`

  (inclut de nombreux champs superflus ; il est recommandé de nettoyer le résultat)

- Demander à `kubectl` de générer le YAML

  (avec `kubectl --dry-run=client -o yaml create/run ...`)

- Partir de zéro avec son éditeur favori

  (niveau ceinture noire 😅)

---

## Rédiger un manifest de Pod

- Utilisons `kubectl --dry-run=client -o yaml`

.lab[

- Générer le manifest du Pod :
  ```bash
  kubectl run --dry-run=client -o yaml purple --image=jpetazzo/color
  ```

- L'enregistrer dans un fichier :
  ```bash
  kubectl run --dry-run=client -o yaml purple --image=jpetazzo/color \
  > pod-purple.yaml
  ```

]

---

## Exécuter le Pod

- Créons le Pod avec le manifest que nous venons de générer

.lab[

- Créer l'ensemble des ressources (à ce stade, uniquement notre Pod) décrites dans le manifest :
  ```bash
  kubectl create -f pod-purple.yaml
  ```

- Confirmer que le Pod est bien en cours d'exécution
  ```bash
  kubectl get pods
  ```

]

---

class: extra-details

## Comparaison avec un `kubectl run` direct

- Le Pod doit être identique à un Pod créé directement avec `kubectl run`

.lab[

- Créer un Pod directement avec `kubectl run` :
  ```bash
  kubectl run yellow --image=jpetazzo/color
  ```

- Comparer les manifests et le status des deux Pods :
  ```bash
  kubectl get pod purple -o yaml
  kubectl get pod yellow -o yaml
  ```

]

---

## Générer un manifest de Deployment

- Après un Pod, créons un Deployment !

.lab[

- Générer le YAML d'un Deployment :
  ```bash
  kubectl create deployment purple --image=jpetazzo/color -o yaml --dry-run=client
  ```

- L'enregistrer dans un fichier :
  ```bash
  kubectl create deployment purple --image=jpetazzo/color -o yaml --dry-run=client \
  > deployment-purple.yaml
  ```

- Et créer le Deployment :
  ```bash
  kubectl create -f deployment-purple.yaml
  ```

]

---

## Mettre à jour notre Deployment

- Que faire si nous voulons scaler ce Deployment ?

- Option 1 : `kubectl scale`

- Option 2 : mettre à jour le manifest YAML

- Choisissons l'option 2 !

.lab[

- Éditer le manifest YAML :
  ```bash
  vim deployment-purple.yaml
  ```

- Trouver la ligne contenant `replicas: 1` et modifier le nombre de replicas

]

---

## Appliquer nos modifications

- Problème : `kubectl create` ne met pas à jour ("n'écrase pas") les ressources

.lab[

- Faites le test :
  ```bash
  kubectl create -f deployment-purple.yaml
  # Cela génère une erreur ("AlreadyExists")
  ```

]

- Que pouvons-nous faire alors ?

---

## Mettre à jour des ressources

- Option 1 : supprimer le Deployment et le recréer

  (efficace, mais provoque une interruption de service !)

- Option 2 : utiliser `kubectl scale` ou `kubectl edit` sur le Deployment

  (efficace, mais c'est tricher : nous voulons utiliser le YAML !)

- Option 3 : `kubectl apply`

---

## `kubectl apply` vs `create`

- `kubectl create -f whatever.yaml`

  - crée les ressources si elles n'existent pas

  - si les ressources existent déjà, ne les modifie pas
    <br/>(et affiche un message d'erreur)

- `kubectl apply -f whatever.yaml`

  - crée les ressources si elles n'existent pas

  - si les ressources existent déjà, les met à jour
    <br/>(pour correspondre à la définition fournie par le fichier YAML)

  - stocke le manifest sous forme d'*annotation* dans la ressource

---

## Tester `kubectl apply`

.lab[

- D'abord, supprimer le Deployment :
  ```bash
  kubectl delete deployment purple
  ```

- Le recréer en utilisant `kubectl apply` :
  ```bash
  kubectl apply -f deployment-purple.yaml
  ```

- Éditer le manifest YAML, changer à nouveau le nombre de replicas :
  ```bash
  vim deployment-purple.yaml
  ```

- Appliquer le nouveau manifest :
  ```bash
  kubectl apply -f deployment-purple.yaml
  ```

]

---

## `create` → `apply`

- Quelles sont les différences entre `kubectl create -f` et `kubectl apply -f` ?

  - `kubectl apply` ajoute une annotation
    <br/>
    (`kubectl.kubernetes.io/last-applied-configuration`)

  - `kubectl apply` effectue une requête `GET` supplémentaire
    <br/>
    (pour récupérer l'objet existant, ou au moins vérifier s'il existe)

- En dehors de cela, le résultat final est identique !

- Il est presque toujours préférable d'utiliser `kubectl apply`

  (sauf lorsqu'on souhaite éviter cette annotation supplémentaire, par ex. pour des objets volumineux comme certaines CRD)

- À partir de maintenant, nous utiliserons presque systématiquement `kubectl apply -f` au lieu de `kubectl create -f`

---

## Ajouter un Service

- Générons le YAML d'un Service exposant notre Deployment

.lab[

- Exécuter `kubectl expose`, à nouveau avec `-o yaml --dry-run=client` :
  ```bash
  kubectl expose deployment purple --port 80 -o yaml --dry-run=client
  ```

- L'enregistrer dans un fichier :
  ```bash
  kubectl expose deployment purple --port 80 -o yaml --dry-run=client \
  > service-purple.yaml
  ```

]

- Remarque : si le Deployment n'existe pas, `kubectl expose` ne fonctionnera pas !

---

## Que faire si le Deployment n'existe pas ?

- Nous pouvons aussi utiliser `kubectl create service`

- La syntaxe est légèrement différente

  (`--port` devient `--tcp` pour une raison obscure)

.lab[

- Générer le YAML avec `kubectl create service` :
  ```bash
  kubectl create service clusterip purple --tcp 80 -o yaml --dry-run=client
  ```

]

---

## Combiner les manifests

- Nous pouvons regrouper plusieurs ressources dans un seul fichier YAML

- Il faut les séparer avec le séparateur de document YAML standard

  (c'est-à-dire `---` seul sur sa propre ligne)

.lab[

- Générer un fichier YAML combiné :
  ```bash
    for YAMLFILE in deployment-purple.yaml service-purple.yaml; do
      echo ---
      cat $YAMLFILE
    done > app-purple.yaml
  ```

]

---

class: extra-details

## Ordre des ressources

- *En règle générale,* l'ordre des ressources n'a pas d'importance :

  - dans de nombreux cas, les ressources ne se référencent pas explicitement
    <br/>
    (par ex. un Service peut exister même si le Deployment correspondant n'existe pas)

  - dans certains cas, il peut y avoir une erreur transitoire, mais Kubernetes réessaiera
    <br/>
    (et finira par réussir)

- Une exception : les Namespaces doivent être créés *avant* les ressources qu'ils contiennent !

---

## Utiliser `-f` avec d'autres commandes

- Nous pouvons aussi utiliser `kubectl delete -f`, `kubectl label -f`, et bien d'autres !

.lab[

- Appliquer le fichier YAML résultant :
  ```bash
  kubectl apply -f app-purple.yaml
  ```

- Ajouter un label à la fois au Deployment et au Service :
  ```bash
  kubectl label -f app-purple.yaml release=production
  ```

- Les supprimer :
  ```bash
  kubectl delete -f app-purple.yaml
  ```

]

---

class: extra-details

## Élagueur¹ (pruning) de ressources

- Nous pouvons aussi demander à `kubectl` de supprimer les anciennes ressources

- Cela s'effectue avec `kubectl apply -f ... --prune`

- Il supprimera les ressources qui ne figurent plus dans le ou les fichier(s) YAML

- Mais uniquement si elles ont été créées avec `kubectl apply` à l'origine

  (techniquement, si elles possèdent l'annotation `kubectl.kubernetes.io/last-applied-configuration`)

.footnote[¹Si l'anglais n'est pas votre langue maternelle : *to prune* signifie élaguer, tailler les branches mortes ou superflues d'un arbre pour l'aider à grandir.]

---

## Avantages du YAML

- Utiliser le YAML (au lieu de `kubectl create <kind>`) permet d'être *déclaratif*

- Le YAML décrit l'état désiré de notre cluster et de nos applications

- Le YAML peut être stocké, versionné, archivé (par ex. dans des dépôts git)

- Pour modifier des ressources, on modifie les fichiers YAML

  (au lieu d'utiliser `kubectl edit`/`scale`/`label`/etc.)

- Les modifications peuvent être revues avant d'être appliquées

  (via des revues de code, pull requests...)

- Notre système de gestion de versions conserve désormais un historique complet de ce que nous déployons

---

## GitOps

- Ce workflow est parfois appelé "GitOps"

- Il existe des outils pour le faciliter, par ex. Flux, ArgoCD...

- Comparable à l'« Infrastructure-as-Code », mais appliqué au déploiement d'applications

---

class: extra-details

## Vraiment GitOps ?

Il existe un débat autour de la « véritable » définition de GitOps :

*Mes applications sont définies par des manifests, des templates, des configurations...
stockés dans des dépôts de sources versionnés,
et je ne modifie mes applications qu'en modifiant ces fichiers,
de la même manière que je modifierais du code source.*

vs

*Même chose, mais il ne s'agit de « GitOps » que si le déploiement des manifests est
entièrement automatisé (par opposition à l'exécution manuelle de commandes comme `kubectl apply`
ou de scripts ou outils plus complexes).*

Votre formateur a peut-être un avis sur la question ! 😁

---

## Le YAML en pratique

- Démarrez avec `kubectl create deployment` et `kubectl expose`

  (jusqu'à obtenir quelque chose qui fonctionne)

- Ensuite, réexécutez ces commandes avec `-o yaml --dry-run=client`

  (pour générer et sauvegarder les manifests YAML)

- Essayez d'appliquer ces manifests dans un environnement propre

  (par ex. un nouveau Namespace)

- Vérifiez que tout fonctionne ; ajustez et itérez si nécessaire

- Committez le YAML dans un dépôt 💯🏆️

---

## Le YAML « Jour 2 »

- N'hésitez pas à supprimer les champs inutilisés

  (par ex. `creationTimestamp: null`, la plupart des valeurs `{}`...)

- Analysez votre YAML avec :

  [kube-score](https://github.com/zegl/kube-score) (installable avec krew)

  [kube-linter](https://github.com/stackrox/kube-linter)

- Inspectez les ressources actives avec des outils comme [popeye](https://popeyecli.io/)

- Rappelez-vous que, comme tous les linters, ils doivent être configurés selon vos besoins !

---

class: extra-details

## Spécifier le namespace

- Lors de la création de ressources depuis des manifests YAML, le namespace est optionnel

- Si nous spécifions un namespace :

  - les ressources sont créées dans le namespace spécifié

  - c'est l'usage typique pour les éléments déployés une seule fois par cluster

  - exemple : composants système, add-ons de cluster...

- Si nous ne spécifions pas de namespace :

  - les ressources sont créées dans le namespace courant

  - c'est l'usage typique pour les éléments qui peuvent être déployés plusieurs fois

  - exemple : applications (production, staging, feature branches...)

???

:FR:- Déployer avec des *manifests* YAML
:FR:- Comment écrire des *manifests* YAML