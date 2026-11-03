# Namespaces

* Les ressources comme les Pods, Deployments, Services... résident dans des *Namespaces*

* Jusqu'à présent, nous avons (probablement) utilisé le Namespace `default`

* Nous pouvons créer d'autres Namespaces pour organiser nos ressources

---

## Cas d'usage

* Exemple : un cluster de "dev" où chaque développeur dispose de son propre Namespace

  (et n'a accès qu'à son propre Namespace, pas à ceux des autres)

* Exemple : un cluster avec un Namespace `production` et un Namespace `staging`

  (avec des applications similaires qui tournent dans chacun, mais avec un dimensionnement différent)

* Exemple : un cluster de "production" avec un Namespace par application

  (ou un Namespace par composant d'une application plus vaste)

* Exemple : un cluster de "production" avec de multiples instances de la même application

  (par ex. une application SaaS avec une instance par client)

---

## Namespaces préexistants

* Sur un cluster fraîchement déployé, nous trouvons généralement les quatre Namespaces suivants :

  * `default` (le Namespace initial pour nos applications ; héberge également le Service `kubernetes`)

  * `kube-system` (pour le control plane)

  * `kube-public` (contient un ConfigMap pour le cluster discovery)

  * `kube-node-lease` (depuis Kubernetes 1.14 ; contient les objets Lease)

* Avec le temps, nous serons presque systématiquement amenés à créer d'autres Namespaces !

---

## Créer un Namespace

* Découvrons deux approches pour créer un Namespace !

.lab[

* Tout d'abord, avec `kubectl create namespace` :
  ```bash
  kubectl create namespace blue
  ```

* Ensuite, avec un extrait YAML :
  ```bash
    kubectl apply -f- <<EOF
    apiVersion: v1
    kind: Namespace
    metadata:
      name: green
    EOF
  ```

]

---

## Utiliser les namespaces

* Nous pouvons passer le flag `-n` ou `--namespace` à la plupart des commandes `kubectl`

.lab[

* Créer un Deployment dans le Namespace `blue` :
  ```bash
  kubectl create deployment purple --image jpetazzo/color --namespace blue
  ```

* Vérifier les Pods qui viennent d'être créés :
  ```bash
  kubectl get pods --all-namespaces
  kubectl get pods --all-namespaces --selector app=purple
  ```

]

---

## Basculer le Namespace actif

* Nous pouvons modifier le Namespace "actif"

* C'est très pratique si nous devons travailler dans un Namespace spécifique pendant un moment :

  * c'est plus simple que d'ajouter systématiquement `--namespace ...`

  * cela permet d'éviter les erreurs
    <br/>
    (par ex. : `kubectl delete -f foo.yaml` oups, mauvais Namespace !)

* Nous allons voir ~~une~~ ~~deux~~ trois méthodes différentes pour basculer de namespace !

---

## Méthode 1 (kubens/kns)

* Pour basculer vers le Namespace `blue`, exécutez :
  ```bash
  kubens blue
  ```

* `kubens` est parfois renommé ou associé à l'alias `kns`

  (encore moins de touches à taper !)

* `kubens -` permet de revenir au Namespace précédent

* Avantages : probablement la solution la plus simple existante

* Inconvénients : `kubens` est un outil supplémentaire qu'il faut installer

---

## Méthode 2 (éditer le fichier kubeconfig)

* Éditer `~/.kube/config`

* Il devrait y avoir un champ `namespace:` quelque part :

  * sauf si vous n'avez encore jamais changé de Namespace !

  * dans ce cas, changez de Namespace au moins une fois via une autre méthode

* Il suffit d'éditer ce fichier, et `kubectl` utilisera ce nouveau Namespace à partir de là

* Avantages : plutôt simple ; ne nécessite aucun outil externe

* Inconvénients : il peut y avoir plusieurs champs `namespace:` dans ce fichier ; difficile à automatiser

---

## Méthode 3 (kubectl config)

* Pour basculer vers le Namespace `blue`, exécutez :
  ```bash
  kubectl config set-context --current --namespace blue
  ```

* Cela modifie automatiquement le fichier kubeconfig

* C'est exactement ce que fait `kubens` en coulisses !

* Avantages : fonctionne toujours (dès lors qu'on dispose de `kubectl`)

* Inconvénients : long et fastidieux à taper

  (mais c'est un bon exercice de dactylographie !)

---

class: extra-details

## Qu'est-ce qu'un contexte ?

* Context = cluster + user + namespace

* Pratique pour basculer rapidement d'un cluster à l'autre

  (par ex. dev, prod, ou différentes applications, différents clients...)

* Également pratique pour basculer rapidement entre différentes identités

  (par ex. un développeur avec des accès standards vs cluster-admin)

* On change de contexte avec `kubectl config set-context` ou avec `kubectx` / `kctx`

* Il est aussi possible de changer totalement de fichier kubeconfig

  (en spécifiant `--kubeconfig` ou en définissant la variable d'environnement `KUBECONFIG`)

---

class: extra-details

## Contenu d'un contexte

* NAME est une chaîne de caractères libre permettant d'identifier le contexte

* CLUSTER est une référence vers un cluster

  (c'est-à-dire l'URL de l'API endpoint, et un certificat optionnel)

* AUTHINFO est une référence vers les données d'authentification à employer

  (c'est-à-dire un certificat client TLS, un token, ou autre méthode)

* NAMESPACE correspond au namespace

  (chaîne vide = `default`)

---

## Namespaces, Services et DNS

* Lorsqu'un Service est créé, un enregistrement est injecté dans le DNS Kubernetes

* Par exemple, pour le service `auth` dans le domaine `staging`, cela ressemble typiquement à :

  `auth.staging.svc.cluster.local`

* Par défaut, les Pods sont configurés pour résoudre les noms dans le domaine de leur propre Namespace

* Par exemple, un Pod situé dans le Namespace `staging` disposera de la search list suivante :

  `search staging.svc.cluster.local svc.cluster.local cluster.local`

---

## Connexion des Pods aux Services

* Imaginons que nous soyons dans le Namespace `staging`

* ... et qu'il existe un Service nommé `auth`

* ... et que nous ayons du code s'exécutant dans un Pod de ce même Namespace

* Notre code peut :

  * se connecter au Service `auth` dans le même Namespace avec `http://auth/`

  * se connecter au Service `auth` dans un autre Namespace (ex. `prod`) avec `http://auth.prod`

  * ... ou `http://auth.prod.svc` ou `http://auth.prod.svc.cluster.local`

---

## Déployer de multiples instances d'une stack

Si tous les conteneurs d'une stack donnée utilisent le DNS pour la service discovery,
cette stack peut être déployée de manière parfaitement identique dans plusieurs Namespaces.

Chaque copie de la stack communiquera avec les services appartenant
au Namespace de cette stack.

Exemple : nous pouvons déployer plusieurs copies de DockerCoins, une par
Namespace, sans modifier une seule ligne de code dans DockerCoins,
et même sans changer une seule ligne dans nos manifests YAML !

Cela équivaut à ce que l'on peut faire par exemple avec Docker Compose
(mais avec Docker Compose, chaque stack est déployée dans un "network" Docker
au lieu d'un Namespace Kubernetes).

---

## Namespaces et isolation

* Les Namespaces *ne garantissent pas* d'isolation réseau

* Par défaut, les Pods des Namespaces `prod` et `staging` peuvent communiquer entre eux

* La véritable isolation est implémentée au moyen des *network policies*

* Les network policies sont des ressources (au même titre que les deployments, services, namespaces...)

* Les network policies spécifient quels flux réseau sont autorisés :

  * entre les pods

  * depuis les pods vers l'extérieur

  * et inversement

---

## `kubens` et `kubectx`

* Ces outils sont disponibles à l'adresse https://github.com/ahmetb/kubectx

* À l'origine simples scripts shell, ce sont désormais des exécutables complets écrits en Go

* Sur nos clusters, ils sont installés sous les alias `kns` et `kctx`

  (pour la concision et pour éviter les conflits d'auto-complétion entre `kubectx` et `kubectl`)

---

## `kube-ps1`

* Il est facile de perdre le fil du cluster / contexte / namespace actif

* `kube-ps1` facilite ce suivi en les intégrant directement dans l'invite de commande (prompt) du shell

* Il est installé sur nos clusters de formation, ainsi que lors de l'utilisation de [shpod](https://github.com/jpetazzo/shpod)

* Il produit un prompt qui ressemble à ceci :
  ```
  [123.45.67.89] `(kubernetes-admin@kubernetes:default)` docker@node1 ~
  ```
  (La partie mise en évidence correspond à `context:namespace`, gérée par `kube-ps1`)

* Vivement recommandé si vous travaillez au quotidien sur plusieurs contextes ou namespaces !

---

## Installer `kube-ps1`

* C'est un simple script shell disponible sur https://github.com/jonmosco/kube-ps1

* Il doit être [installé dans nos fichiers de profil/rc](https://github.com/jonmosco/kube-ps1#installing)

  (les instructions varient en fonction du système d'exploitation, du shell, etc.)

* Une fois installé, il met à disposition les alias `kube_ps1`, `kubeon`, `kubeoff`

  (pour l'activer ou le désactiver à la demande)

* Conseil de pro : installez-le sur votre poste pendant la prochaine pause !

???

:FR:- Organiser les ressources avec des *namespaces*