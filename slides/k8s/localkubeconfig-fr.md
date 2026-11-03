# Contrôler un cluster Kubernetes à distance

* `kubectl` peut être utilisé soit directement sur les instances du cluster, soit depuis l'extérieur

* Ici, nous allons utiliser `kubectl` depuis notre machine locale

---

## Prérequis

.warning[Les commandes de ce chapitre doivent être exécutées *sur votre machine locale*.]

* `kubectl` est officiellement disponible sous Linux, macOS, Windows

  (et officieusement partout où l'on peut compiler et exécuter des binaires Go)

* Vous pouvez ignorer ces commandes si vous suivez la formation depuis :

  * une tablette ou un smartphone

  * un terminal dans un navigateur web

  * un environnement où vous ne pouvez pas installer ni exécuter de nouveaux binaires

---

## Installer `kubectl`

* Si vous avez déjà `kubectl` installé sur votre machine locale, vous pouvez passer cette étape

.lab[

<!-- ##VERSION## -->

* Téléchargez le binaire `kubectl` depuis l'un des liens suivants :

  [Linux](https://storage.googleapis.com/kubernetes-release/release/v1.19.2/bin/linux/amd64/kubectl)
  |
  [macOS](https://storage.googleapis.com/kubernetes-release/release/v1.19.2/bin/darwin/amd64/kubectl)
  |
  [Windows](https://storage.googleapis.com/kubernetes-release/release/v1.19.2/bin/windows/amd64/kubectl.exe)

* Sous Linux et macOS, rendez le binaire exécutable avec `chmod +x kubectl`

  (Et pensez à le lancer avec `./kubectl` ou à le déplacer dans votre `$PATH`)

]

Note : si vous suivez ce cours avec une plateforme différente (par ex. Linux sur une architecture distincte d'amd64, ou depuis un smartphone ou une tablette), l'installation de `kubectl` peut être plus complexe (voire impossible), n'hésitez donc pas à ignorer cette section.

---

## Tester `kubectl`

* Vérifiez que `kubectl` fonctionne correctement

  (avant même de tenter de vous connecter à un cluster distant !)

.lab[

* Demander à `kubectl` d'afficher son numéro de version :
  ```bash
  kubectl version --client
  ```

]

La sortie devrait ressembler à ceci :
```
Client Version: version.Info{Major:"1", Minor:"15", GitVersion:"v1.15.0",
GitCommit:"e8462b5b5dc2584fdcd18e6bcfe9f1e4d970a529", GitTreeState:"clean",
BuildDate:"2019-06-19T16:40:16Z", GoVersion:"go1.12.5", Compiler:"gc",
Platform:"darwin/amd64"}
```

---

## Sauvegarder le fichier `~/.kube/config` existant

* Si vous possédez déjà un fichier `~/.kube/config`, renommez-le

  (nous allons l'écraser dans les diapositives suivantes !)

* Si vous n'avez jamais utilisé `kubectl` sur votre machine : vous n'avez rien à faire !

.lab[

* Faites une copie de `~/.kube/config` ; sous macOS ou Linux, vous pouvez exécuter :
  ```bash
  cp ~/.kube/config ~/.kube/config.before.training
  ```

* Si vous utilisez Windows, adaptez cette commande en conséquence

]

---

## Copier le fichier de configuration depuis `node1`

* Le fichier `~/.kube/config` présent sur `node1` contient toutes les informations d'authentification (credentials) nécessaires

* Copions-le en local !

.lab[

* Copier le fichier depuis `node1` ; sous macOS ou Linux, vous pouvez faire :
  ```
  scp `USER`@`X.X.X.X`:.kube/config ~/.kube/config
  # Veillez à remplacer X.X.X.X par l'adresse IP de node1,
  # et USER par le nom d'utilisateur servant à vous connecter à node1 !
  ```

* Si vous êtes sous Windows, adaptez ces instructions selon votre client SSH

]

---

## Mettre à jour l'adresse du serveur

* Il y a de fortes chances que nous devions mettre à jour l'adresse du serveur

* Pour vérifier si cela est nécessaire, exécutez `kubectl config view`

* Recherchez l'adresse indiquée après `server:` :

  * si elle correspond à l'adresse IP publique de `node1`, vous êtes prêt !

  * s'il s'agit d'autre chose (en particulier d'une adresse IP privée), mettez-la à jour !

* Pour mettre à jour l'adresse du serveur, exécutez :
  ```bash
  kubectl config set-cluster kubernetes --server=https://`X.X.X.X`:6443
  # Pensez à remplacer X.X.X.X par l'adresse IP de node1 !
  ```

---

class: extra-details

## Que faire en cas d'erreur de certificat ?

* Généralement, l'API Kubernetes utilise un certificat qui n'est valide que pour :

  * `kubernetes`
  * `kubernetes.default`
  * `kubernetes.default.svc`
  * `kubernetes.default.svc.cluster.local`
  * l'adresse ClusterIP du Service `kubernetes`
  * le nom d'hôte du node hébergeant le control plane (par ex. `node1`)
  * l'adresse IP du node hébergeant le control plane

* Chez la plupart des fournisseurs cloud, l'adresse IP du node est une adresse IP interne

* ... Et nous allons nous connecter via l'adresse IP externe

* ... Et cette adresse IP externe n'a pas été incluse lors de la génération du certificat !

---

class: extra-details

## Contourner l'erreur de certificat

* Nous devons demander à `kubectl` d'ignorer la vérification TLS

  (ne faites cela que sur des clusters de test, jamais en production !)

* La commande suivante permet de l'appliquer :
  ```bash
  kubectl config set-cluster kubernetes --insecure-skip-tls-verify
  ```

---

## Vérifier la connexion au cluster

* Nous pouvons à présent lancer quelques commandes élémentaires pour nous assurer que tout fonctionne

.lab[

* Vérifier les versions du client local et du serveur distant :
  ```bash
  kubectl version
  ```

* Lister les nodes du cluster :
  ```bash
  kubectl get nodes
  ```

]

Nous pouvons désormais utiliser le cluster exactement comme si nous étions connectés directement sur un node, mais à distance.

???

:FR:- Travailler avec des *clusters* distants