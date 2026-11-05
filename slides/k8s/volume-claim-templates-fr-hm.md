
## Persistent Volume Claims et Stateful sets

- Un Stateful set peut définir un (ou plusieurs) `volumeClaimTemplate`

- Chaque `volumeClaimTemplate` va générer un Persistent Volume Claim par Pod

- Chaque Pod disposera donc de son propre volume individuel

- Ces volumes sont numérotés (au même titre que les Pods)

- Exemple :

  - un Stateful set est nommé `consul`
  - il est dimensionné à plusieurs replicas
  - il définit un `volumeClaimTemplate` nommé `data`
  - il va alors créer les pods `consul-0`, `consul-1`, `consul-2`
  - ces pods disposeront de volumes nommés `data`, référençant des PersistentVolumeClaims
    nommés `data-consul-0`, `data-consul-1`, `data-consul-2`

---

## Les Persistent Volume Claims sont persistants et attachés

- Lors de la mise à jour du Stateful set (ex. mise à jour d'image), chaque pod conserve son volume

- Lorsque les pods sont reprogrammés (ex. panne d'un node), ils conservent leur volume

  (cela nécessite un système de stockage qui ne soit pas local au node)

- Ces volumes ne sont pas supprimés automatiquement

  (lorsque le Stateful set fait l'objet d'un scale down ou d'une suppression)

- Si le Stateful set est à nouveau augmenté ultérieurement, les pods retrouvent leurs données

---

## Déployer Consul

- Utilisons un nouveau manifest pour notre cluster Consul

- Les seules différences entre ce fichier et le précédent sont :

  - le `volumeClaimTemplate` défini dans la spec du Stateful Set

  - les `volumeMounts` correspondants dans la spec du Pod

.lab[

- Appliquer le fichier YAML persistent pour Consul :
  ```bash
  kubectl apply -f ~/container.training/k8s/consul-3.yaml
  ```

]

---

## Absence de provisioner dynamique

- En l'absence de provisioner dynamique, nous devons créer les PV manuellement

- Nous allons utiliser des local volumes

  (conceptuellement similaires aux volumes `hostPath`)

- Nous pouvons utiliser des local volumes sans installer de plugins additionnels

- Cependant, ils sont liés à un node spécifique

- Si ce node devient indisponible, le volume devient inaccessible

---

## Observer la situation

- Observons l'état des Persistent Volume Claims et des Pods

.lab[

- Vérifier que nous avons désormais un Persistent Volume Claim non lié :
  ```bash
  kubectl get pvc
  ```

- Nous n'avons aucun Persistent Volume :
  ```bash
  kubectl get pv
  ```

- Le Pod `consul-0` n'est pas encore planifié :
  ```bash
  kubectl get pods -o wide
  ```

]

*Astuce : laissez ces commandes s'exécuter avec `-w` dans des terminaux séparés.*

---

## Explications

- Dans un Stateful Set, les Pods démarrent de manière séquentielle, un par un

- `consul-1` ne sera pas créé tant que `consul-0` n'est pas en cours d'exécution

- `consul-0` dépend d'un Persistent Volume Claim non lié

- Le scheduler ne planifiera pas le Pod tant que le PVC n'est pas bound

  (car le PVC pourrait être lié à un volume uniquement disponible sur un sous-ensemble de nodes ; par exemple, les volumes EBS sont cantonnés à une Availability Zone)

---

## Créer les Persistent Volumes

- Créons 3 répertoires locaux (`/mnt/consul`) sur node2, node3, node4

- Créons ensuite 3 Persistent Volumes correspondant à ces répertoires

.lab[

- Créer les répertoires locaux :
  ```bash
    for NODE in node2 node3 node4; do
      ssh $NODE sudo mkdir -p /mnt/consul
    done
  ```

- Créer les objets PV :
  ```bash
  kubectl apply -f ~/container.training/k8s/volumes-for-consul.yaml
  ```

]

---

## Vérifier notre cluster Consul

- Les PV que nous avons créés seront automatiquement mis en correspondance avec les PVC

- Une fois un PVC bound, son pod peut démarrer normalement

- Dès que le pod `consul-0` a démarré, `consul-1` peut être créé, et ainsi de suite

- Enfin, notre cluster Consul est opérationnel, adossé à des volumes "persistants"

.lab[

- Vérifier que notre cluster Consul compte bien 3 membres :
  ```bash
  kubectl exec consul-0 -- consul members
  ```

]

---

## Les pièges à éviter (1/2)

- La taille déclarée des Persistent Volumes est fictive

  (elle sert lors de l'association entre PV et PVC, mais il n'y a pas de quota ou de limite technique réelle appliquée ici)

- Le Pod peut finir par utiliser plus que la taille demandée

- Le PV peut ou non disposer réellement de la capacité annoncée

- Cela fonctionne très bien avec des block volumes provisionnés dynamiquement

- ...Beaucoup moins bien dans d'autres scénarios !

---

## Les pièges à éviter (2/2)

- Cet exemple précis a fonctionné car nous disposions d'exactement 1 PV libre par node :

  - si nous avions créé plusieurs PV par node...

  - nous aurions pu nous retrouver avec deux PVC liés à des PV sur le même node...

  - ce qui aurait nécessité que deux pods soient placés sur le même node...

  - ce qui est formellement interdit par les contraintes d'anti-affinité configurées dans le StatefulSet

- Pour éviter cela, il est nécessaire d'associer les PV à une Storage Class configurée avec :
  ```yaml
  volumeBindingMode: WaitForFirstConsumer
  ```
  (cela garantit qu'un PVC ne sera bound à un PV qu'après avoir été programmé pour un Pod)

- Consultez [cet article de blog](https://kubernetes.io/blog/2018/04/13/local-persistent-volumes-beta/) pour plus de détails

---

## Avec un provisioner dynamique

Voici les étapes franchies lors du dynamic provisioning :

1. Le Stateful Set génère les PVC conformément au `volumeClaimTemplate`.

2. Le Stateful Set instancie les Pods utilisant ces PVC.

3. Les PVC sont automatiquement annotés avec notre Storage Class.

4. Le provisioner dynamique alloue les volumes et crée les PV correspondants.

5. Le PersistentVolumeClaimBinder associe les PV et les PVC ensemble.

6. Les PVC passent à l'état bound, les Pods peuvent démarrer.

---

## Valider la persistance (1)

- Lorsque le StatefulSet est supprimé, le PVC et le PV continuent d'exister

- Et si nous recréons un StatefulSet identique, le PVC et le PV sont réutilisés

- Vérifions-le !

.lab[

- Écrire une donnée dans Consul :
  ```bash
  kubectl exec consul-0 -- consul kv put answer 42
  ```

- Supprimer le cluster Consul :
  ```bash
  kubectl delete -f ~/container.training/k8s/consul-3.yaml
  ```

]

---

## Valider la persistance (2)

.lab[

- Attendre la suppression effective du dernier Pod :
  ```bash
  kubectl wait pod consul-0 --for=delete
  ```

- Vérifier que le PV et le PVC sont toujours présents :
  ```bash
  kubectl get pv,pvc
  ```

]

---

## Valider la persistance (3)

.lab[

- Recréer le cluster :
  ```bash
  kubectl apply -f ~/container.training/k8s/consul-3.yaml
  ```

- Attendre qu'il soit opérationnel

- Accéder ensuite à la clé définie précédemment :
  ```bash
  kubectl exec consul-0 -- consul kv get answer
  ```

]

---

## Nettoyage

- Les PV et PVC ne sont pas supprimés automatiquement

- C'est un avantage (réduction du risque de perte de données accidentelle)

- C'est un inconvénient (consommation accrue de stockage)

- Gérer le cycle de vie des PVC :

  - les supprimer manuellement

  - ajouter leur StatefulSet dans leurs `ownerReferences`

  - supprimer le Namespace auquel ils appartiennent

???

:FR:- Définir des volumeClaimTemplates