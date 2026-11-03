# Managing secrets

* Notre code a parfois besoin d'informations sensibles :

  * mots de passe

  * tokens d'API

  * clés TLS

  * ...

* Les *Secrets* sont prévus à cet effet

* Les Secrets et les ConfigMaps sont très similaires

---

## Similarités entre ConfigMap et Secrets

* Les ConfigMaps et les Secrets sont des tables clé-valeur

  (un Secret peut contenir zéro, une ou plusieurs paires clé-valeur)

* Ils peuvent tous deux être exposés via la downward API ou au moyen de volumes

* Ils peuvent tous deux être créés avec du YAML ou via la CLI

  (`kubectl create configmap` / `kubectl create secret`)

---

## ConfigMap et Secrets sont des ressources distinctes

* Ils peuvent être régis par des permissions RBAC différentes

  (par ex. le rôle par défaut `view` permet de lire les ConfigMaps mais pas les Secrets)

* Ils traduisent une *intention* différente :

  *« Vous devriez utiliser les secrets pour les données réellement secrètes comme les clés d'API, 
  les identifiants, etc., et utiliser les config maps pour les données de configuration non secrètes. »*

  *« À l'avenir, il y aura probablement des éléments différenciateurs pour les secrets, comme la rotation ou le support d'un backend HSM pour l'API des secrets, etc. »*

  (Source : [l'auteur de ces deux fonctionnalités](https://stackoverflow.com/a/36925553/580281))

---

## Les Secrets ont un *type* optionnel

* Le type indique quelles clés doivent obligatoirement être présentes dans les secrets, par exemple :

  `kubernetes.io/tls` requiert `tls.crt` et `tls.key`

  `kubernetes.io/basic-auth` requiert `username` et `password`

  `kubernetes.io/ssh-auth` requiert `ssh-privatekey`

  `kubernetes.io/dockerconfigjson` requiert `.dockerconfigjson`

  `kubernetes.io/service-account-token` requiert `token`, `namespace`, `ca.crt`

  (la liste complète est consultable dans [la documentation](https://kubernetes.io/docs/concepts/configuration/secret/#secret-types))

* Cela sert principalement de repère pour les humains :

  « Ah oui, ce secret est un... »

---

## Accéder à des dépôts privés

* Voyons comment accéder à une image située sur un registry privé !

* Ces images sont protégées par un nom d'utilisateur et un mot de passe

  (sur certains registres, il s'agit d'un token et d'un mot de passe, mais le principe reste identique)

* Pour accéder à une image privée, nous devons :

  * créer un secret

  * référencer ce secret dans le template d'un Pod

  * ou référencer ce secret dans le ServiceAccount utilisé par un Pod

---

## En pratique

* Tentons d'accéder à une image sur un registry privé !

  * image = docker-registry.enix.io/jpetazzo/private:latest
  * user = reader
  * password = VmQvqdtXFwXfyy4Jb5DR

.lab[

- Créez un Deployment utilisant cette image :
  ```bash
    kubectl create deployment priv \
            --image=docker-registry.enix.io/jpetazzo/private
  ```

- Vérifiez que le Pod ne parvient pas à démarrer :
  ```bash
  kubectl get pods --selector=app=priv
  ```

]

---

## Créer un secret

* Créons un secret avec les informations fournies précédemment

.lab[

- Créez le secret de type registry :
  ```bash
    kubectl create secret docker-registry enix \
            --docker-server=docker-registry.enix.io \
            --docker-username=reader \
            --docker-password=VmQvqdtXFwXfyy4Jb5DR
  ```

]

Pourquoi devons-nous spécifier l'adresse du registry ?

Si nous employons plusieurs jeux d'identifiants pour différents registries, cela évite de divulguer les identifiants d'un registry à un *autre* registry.

---

## Utiliser le secret

* La première méthode pour utiliser un secret consiste à l'ajouter à `imagePullSecrets`

  (dans la section `spec` du template de Pod)

.lab[

- Patcher le Deployment `priv` créé précédemment :
  ```bash
    kubectl patch deploy priv --patch='
    spec:
      template:
        spec:
          imagePullSecrets:
          - name: enix
    '
  ```

]

---

## Vérifier le résultat

.lab[

- Confirmez que notre Pod démarre désormais convenablement :
  ```bash
  kubectl get pods --selector=app=priv
  ```

]

---

## Une autre méthode pour utiliser le secret

* Nous pouvons ajouter le secret au ServiceAccount

* C'est très pratique pour appliquer automatiquement les identifiants à *tous* les pods

  (tant qu'ils utilisent ce ServiceAccount spécifique, bien entendu)

.lab[

- Ajoutez le secret au ServiceAccount :
  ```bash
    kubectl patch serviceaccount default --patch='
    imagePullSecrets:
    - name: enix
    '
  ```

]

---

## Les Secrets sont affichés encodés en base64

* Lorsqu'ils sont affichés avec par ex. `kubectl get secrets -o yaml`, les secrets apparaissent encodés en base64

* De même, lors de leur définition en YAML, les valeurs sous `data` sont encodées en base64

* Exemple :
  ```yaml
    kind: Secret
    apiVersion: v1
    metadata:
      name: pin-codes
    data:
      onetwothreefour: MTIzNA==
      zerozerozerozero: MDAwMA==
  ```

* Gardez à l'esprit qu'il s'agit d'un simple *encodage*, et non d'un *chiffrement*

* Il est extrêmement facile d'[extraire et décoder automatiquement des secrets](https://medium.com/@mveritym/decoding-kubernetes-secrets-60deed7a96a3)

---

class: extra-details

## Utiliser `stringData`

* Lors de la création d'un Secret, il est possible de se dispenser du base64

* Utilisez simplement `stringData` à la place de `data` :
  ```yaml
    kind: Secret
    apiVersion: v1
    metadata:
      name: pin-codes
    stringData:
      onetwothreefour: 1234
      zerozerozerozero: 0000
  ```

* Il apparaîtra néanmoins en base64 si vous faites un `kubectl get -o yaml`

* Aucun `type` n'ayant été spécifié, il prend la valeur par défaut `Opaque`

---

class: extra-details

## Chiffrement au repos (Encryption at rest)

* Il est possible de [chiffrer les secrets au repos](https://kubernetes.io/docs/tasks/administer-cluster/encrypt-data/)

* Cela garantit que les secrets restent protégés si quelqu'un...

  * vole nos serveurs etcd

  * dérobe nos sauvegardes

  * espionne par ex. la liaison iSCSI entre nos serveurs etcd et le SAN

* Cependant, le démarrage de l'API server exigera désormais une intervention humaine

  (afin de fournir les clés de déchiffrement)

* Cela est réservé à des environnements soumis à une réglementation extrême (militaire, souveraineté étatique...)

---

class: extra-details

## ConfigMaps et Secrets immuables

* Depuis Kubernetes 1.19, il est possible de déclarer un ConfigMap ou un Secret comme *immutable*

  ```bash
  kubectl patch configmap xyz --patch='{"immutable": true}'
  ```

* Cela apporte des gains de performance lors de l'utilisation d'un volume important de ConfigMaps et de Secrets

  (important = plusieurs dizaines de milliers)

* Une fois qu'un ConfigMap ou un Secret a été déclaré immutable :

  * son contenu ne peut plus être modifié
  * le champ `immutable` ne peut pas être révoqué
  * la seule façon de le modifier est de le supprimer et de le recréer
  * les Pods qui l'utilisent devront également être recréés

???

:FR:- Manipulation de mots de passe, clés API etc.