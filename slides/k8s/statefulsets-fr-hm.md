# Stateful sets

- Les Stateful sets sont un type de ressource de l'API Kubernetes

  (au même titre que les pods, deployments, services...)

- Ils fournissent des mécanismes pour déployer des applications stateful à l'échelle

- Au premier abord, ils ressemblent aux Deployments :

  - un Stateful set définit une spec de pod et un nombre de replicas *R*

  - il garantit que *R* copies du pod sont en cours d'exécution

  - ce nombre peut être modifié pendant l'exécution du Stateful set

  - la mise à jour de la spec du pod déclenche un rolling update

- Mais ils présentent également des différences fondamentales

---

## Fonctionnalités uniques des Stateful sets

- Les pods d'un Stateful set sont indexés (de 0 à *R-1*) et ordonnés

- Ils sont démarrés et mis à jour dans l'ordre (de 0 à *R-1*)

- Un pod n'est démarré (ou mis à jour) que lorsque le précédent est ready

- Ils sont arrêtés dans l'ordre inverse (de *R-1* à 0)

- Chaque pod connaît son identité (c'est-à-dire son index numérique au sein du set)

- Chaque pod peut résoudre facilement l'adresse IP des autres pods

- Les pods peuvent persister des données sur des volumes attachés

🤔 Un instant... Ne pouvons-nous pas déjà attacher des volumes à des pods et des deployments ?

---

## Retour sur les volumes

- Les [Volumes](https://kubernetes.io/docs/concepts/storage/volumes/) répondent à plusieurs cas d'usage :

  - partager des données entre conteneurs au sein d'un même pod

  - exposer des informations de configuration et des secrets aux conteneurs

  - accéder à des systèmes de stockage

- Examinons des exemples illustrant ce dernier usage

---

## Types de volumes

- Il existe de nombreux [types de volumes](https://kubernetes.io/docs/concepts/storage/volumes/#types-of-volumes) disponibles :

  - stockage cloud public (GCEPersistentDisk, AWSElasticBlockStore, AzureDisk...)

  - stockage cloud privé (Cinder, VsphereVolume...)

  - systèmes de stockage traditionnels (NFS, iSCSI, FC...)

  - stockage distribué (Ceph, Glusterfs, Portworx...)

- L'utilisation d'un volume persistant nécessite :

  - de créer le volume hors bande (en dehors de l'API Kubernetes)

  - de référencer le volume dans la description du pod, avec tous ses paramètres

---

## Utiliser un volume cloud

Voici une définition de pod utilisant un volume AWS EBS (qui doit avoir été créé préalablement) :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-using-my-ebs-volume
spec:
  containers:
  - image: ...
    name: container-using-my-ebs-volume
    volumeMounts:
    - mountPath: /my-ebs
      name: my-ebs-volume
  volumes:
  - name: my-ebs-volume
    awsElasticBlockStore:
      volumeID: vol-049df61146c4d7901
      fsType: ext4
```

---

## Utiliser un volume NFS

Voici un autre exemple utilisant un volume hébergé sur un serveur NFS :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: pod-using-my-nfs-volume
spec:
  containers:
  - image: ...
    name: container-using-my-nfs-volume
    volumeMounts:
    - mountPath: /my-nfs
      name: my-nfs-volume
  volumes:
  - name: my-nfs-volume
    nfs:
      server: 192.168.0.55
      path: "/exports/assets"
```

---

## Limites des volumes classiques

- Leur cycle de vie (création, suppression...) est géré en dehors de l'API Kubernetes

  (nous ne pouvons pas simplement utiliser `kubectl apply/create/delete/...` pour les administrer)

- Si un Deployment utilise un volume, tous les replicas partagent le même volume

- Ce volume doit alors supporter les accès concurrents :

  - certains volumes le permettent (ex. les serveurs NFS gèrent les accès multiples en lecture/écriture)

  - certains volumes supportent des lectures concurrentes

  - certains volumes permettent un accès concurrent uniquement pour des pods colocalisés sur le même nœud

- Ce dont nous avons réellement besoin, c'est d'un mécanisme garantissant un volume dédié par replica

---

## Volumes individuels

- Les Pods d'un Stateful set peuvent disposer de volumes individuels

  (dans un Stateful set à 3 replicas, il y aura 3 volumes distincts)

- Ces volumes peuvent être :

  - alloués depuis un pool de volumes préexistants (disques, partitions...)

  - créés dynamiquement via un backend de stockage

- Cela introduit une série de nouveaux types de ressources Kubernetes :

  Persistent Volumes, Persistent Volume Claims, Storage Classes

  (ainsi que les `volumeClaimTemplates`, intégrés directement dans les manifests de Stateful Sets !)

---

## Récapitulatif sur les Stateful sets

- Un Stateful set gère un ensemble de pods identiques

  (à l'image d'un Deployment)

- Ces pods sont indexés numériquement, puis démarrés, mis à jour et arrêtés selon un ordre strict

- Ces pods ont conscience de leur index

  (ex. le #0 peut s'élire primary, et le #1 secondary)

- Ces pods peuvent découvrir les adresses IP des autres pods du set

  (via un *headless service*)

- Ces pods peuvent chacun bénéficier de leur propre stockage persistant

---

## Obtenir un stockage par pod

- Les Stateful Sets peuvent définir des *persistent volume claim templates*

  (déclarés dans `spec.volumeClaimTemplates` au sein du manifest du Stateful set)

- Un claim template instancie un Persistent Volume Claim par pod

  (le PVC sera nommé `<claim-name>.<stateful-set-name>.<pod-index>`)

- Les Persistent Volume Claims sont associés de manière biunivoque (1-to-1) à des Persistent Volumes

- Le provisioning des Persistent Volumes peut s'effectuer :

  - automatiquement (en exploitant le *dynamic provisioning* via une Storage Class)

  - manuellement (l'opérateur crée les volumes en amont ou à la demande)

???

:FR:- Déployer une application avec un Stateful Set
:FR:- Comprendre les Persistent Volume Claims et Storage Classes