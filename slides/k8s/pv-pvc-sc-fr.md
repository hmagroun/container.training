# PV, PVC, and Storage Classes

- Lorsqu'une application a besoin de stockage, elle crée un PersistentVolumeClaim

  (soit directement, soit via un template de volume claim dans un Stateful Set)

- Le PersistentVolumeClaim est initialement à l'état `Pending`

- Kubernetes recherche alors un PersistentVolume adéquat

  (qui peut être immédiatement disponible, ou nécessiter un provisioning)

- Dès qu'un PersistentVolume adapté est trouvé, le PVC passe à l'état `Bound`

- Le PVC peut ensuite être utilisé par un Pod

  (tant que le PVC est `Pending`, le Pod ne peut pas s'exécuter)

---

## Access modes

- Les PV et PVC disposent d'*access modes* :

  - ReadWriteOnce (un seul node peut accéder au volume à la fois)

  - ReadWriteMany (plusieurs nodes peuvent accéder au volume simultanément)

  - ReadOnlyMany (plusieurs nodes peuvent accéder au volume, mais sans droit d'écriture)

  - ReadWriteOncePod (un seul pod peut accéder au volume ; introduit dans Kubernetes 1.22)

- Un PVC liste les access modes qu'il requiert

- Un PV liste les access modes qu'il prend en charge

⚠️ Un PV disposant uniquement de ReadWriteMany ne satisfera pas un PVC demandant ReadWriteOnce !

---

## Capacity

- Un PVC doit exprimer une requête de taille de stockage

  (champ `spec.resources.requests.storage`, en octets)

- Un PV doit indiquer sa taille

  (champ `spec.capacity.storage`, en octets)

- Kubernetes n'associera un PV et un PVC que si le PV est suffisamment grand

- Ces champs ne sont utilisés qu'à des fins de mise en correspondance ("matchmaking") :

  - rien n'empêche le Pod montant le PVC d'utiliser plus d'espace

  - rien n'oblige le PV à être réellement de cette taille

---

## Storage Class

- Que faire si nous disposons de plusieurs systèmes de stockage ?

  (par ex. NFS et iSCSI ; ou AzureFile et AzureDisk ; ou Cinder et Ceph...)

- Que faire si nous avons un système de stockage avec plusieurs tiers ?

  (par ex. un SAN avec RAID1 et RAID5 ; des volumes EBS general purpose vs io optimized...)

- Kubernetes nous permet de définir des *storage classes* pour modéliser ces cas

  (vérifiez celles disponibles actuellement avec `kubectl get storageclasses`)

---

## Utilisation des storage classes

- De manière facultative, chaque PV et chaque PVC peut référencer une StorageClass

  (champ `spec.storageClassName`)

- Lors de la création d'un PVC, spécifier une StorageClass signifie :

  « utilise ce système de stockage précis pour provisionner le volume ! »

- Les storage classes sont indispensables au [dynamic provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/)

  (mais nous pouvons également ne pas les employer et effectuer un provisioning manuel)

---

## Storage class par défaut

- Nous pouvons définir une *default storage class*

  (en lui associant l'annotation `storageclass.kubernetes.io/is-default-class=true`)

- Lorsqu'un PVC est créé,

  **SI** il n'indique pas quelle storage class utiliser

  **ET** qu'une default storage class existe

  **ALORS** le champ `storageClassName` du PVC prend la valeur de la default storage class

---

## Contraintes additionnelles

- Un PersistentVolumeClaim peut également spécifier un selector de volume

  (faisant référence à des labels sur le PV)

- Un PersistentVolume peut aussi être créé avec une directive `claimRef`

  (indiquant à quel PVC précis il doit être lié)

---

class: extra-details

## Quel PV est associé à un PVC ?

- Le PV doit être `Available`

- Le PV doit satisfaire l'ensemble des contraintes du PVC

  (access mode, taille, selector optionnel, storage class optionnelle)

- Les PV présentant l'access mode le plus proche sont retenus

- Puis ceux dont la taille est la plus proche

- Il est possible de spécifier un `claimRef` lors de la création d'un PV

  (cela l'associera au PVC spécifié, mais uniquement s'il satisfait tous les prérequis de ce PVC ; sinon un autre PV pourrait être sélectionné)

- Pour tous les détails concernant le PersistentVolumeClaimBinder, consultez [cette documentation](https://github.com/kubernetes/design-proposals-archive/blob/main/storage/persistent-storage.md#matching-and-binding)

---

## Créer un PVC

- Créons un PVC autonome et observons le résultat !

.lab[

- Vérifier si nous avons une StorageClass :
  ```bash
  kubectl get storageclasses
  ```

- Créer le PVC :
  ```bash
  kubectl create -f ~/container.training/k8s/pvc.yaml
  ```

- Examiner le PVC :
  ```bash
  kubectl get pvc
  ```

]

---

## Quatre scénarios possibles

1. Si nous avons une default StorageClass avec binding *immédiat* :

   *un PV a été créé et associé au PVC*

2. Si nous avons une default StorageClass configurée en *wait for first consumer* :

  *le PVC reste `Pending` mais possède une `STORAGECLASS`* ⚠️

3. Si nous n'avons pas de default StorageClass :

  *le PVC reste `Pending`, sans `STORAGECLASS`*

4. Si nous avons une StorageClass, mais qu'elle ne fonctionne pas :

  *le PVC reste `Pending` mais possède une `STORAGECLASS`* ⚠️

---

## Immediate vs WaitForFirstConsumer

- Immediate = dès qu'un PVC est `Pending`, créer un PV

- Que se passe-t-il si :

  - le PV n'est disponible que sur un seul node (par ex. un volume local)

  - ...ou sur un sous-ensemble de nodes (par ex. SAN HBA, AZ EBS...)

  - le Pod qui utilisera le PVC a des contraintes de scheduling

  - ces contraintes s'avèrent incompatibles avec le PV

- WaitForFirstConsumer = ne provisionner le PV que lorsqu'un Pod monte le PVC

---

## Utiliser le PVC

- Montons le PVC dans un Pod

- Nous utiliserons un Pod autonome (sans Deployment, StatefulSet, etc.)

- Nous allons exploiter @@LINK[k8s/mounter.yaml], présenté dans la section suivante

- Il faudra adapter le `claimName` ! ⚠️

---

```yaml
@@INCLUDE[k8s/mounter.yaml]
```

---

## Lancer le Pod

.lab[

- Éditer le manifest `mounter.yaml`

- Mettre à jour `claimName` pour indiquer le nom de notre PVC

- Créer le Pod

- Vérifier l'état du PV et du PVC

]

Note : ce Pod "mounter" est pratique pour inspecter directement le contenu d'un PVC.

---

## Scénarios 1 & 2

Si nous avons une default Storage Class capable de provisionner le PVC dynamiquement...

- Nous devrions à présent avoir un nouveau PV

- Le PV et le PVC devraient être mutuellement `Bound`

---

## Scénario 3

Si nous n'avons pas de default Storage Class, nous devons créer le PV manuellement.

```bash
kubectl create -f ~/container.training/k8s/pv.yaml
```

Après quelques secondes, vérifiez que le PV et le PVC sont bound :

```bash
kubectl get pv,pvc
```

---

## Scénario 4

Si notre default Storage Class ne peut pas provisionner de PV, faisons-le manuellement.

Le PV doit spécifier le bon `storageClassName`.

```bash
STORAGECLASS=$(kubectl get pvc --selector=container.training/pvc \
               -o jsonpath={..storageClassName})
kubectl patch -f ~/container.training/k8s/pv.yaml --dry-run=client -o yaml \
        --patch '{"spec": {"storageClassName": "'$STORAGECLASS'"}}' \
        | kubectl create -f-
```

Vérifiez que le PV et le PVC sont bound :

```bash
kubectl get pv,pvc
```

---

## Vérifier le Pod

- Si le PVC était `Pending`, le Pod était également `Pending`

- Une fois le PVC `Bound`, le Pod peut être planifié et exécuté

- Dès que le Pod est `Running`, connectez-vous-y via `kubectl attach -ti`

---

## Cycle de vie des PV et PVC

- Nous ne pouvons pas supprimer un PV s'il est `Bound`

- Si nous lançons `kubectl delete`, il passe à l'état `Terminating`

- Nous ne pouvons pas supprimer un PVC s'il est en cours d'utilisation par un Pod

- De même, lors d'un `kubectl delete`, il passe à l'état `Terminating`

- La suppression est bloquée par des *finalizers*

  (= l'équivalent d'un post-it indiquant « ne pas supprimer ! »)

- Lorsque les Pods qui le montent sont supprimés, leurs PVC sont libérés

- Lorsque les PVC sont supprimés, leurs PV sont libérés

???

:FR:- Création de volumes
:FR:- PV, PVC, et StorageClass