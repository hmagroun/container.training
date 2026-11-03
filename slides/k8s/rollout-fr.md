# Rolling updates

- Comment devrions-nous mettre à jour une application en cours d'exécution ?

- Stratégie 1 : supprimer l'ancienne version, puis déployer la nouvelle version

  (pas idéal, car cela provoque évidemment une indisponibilité !)

- Stratégie 2 : déployer la nouvelle version, puis supprimer l'ancienne version

  (consomme beaucoup de ressources ; et comment basculer le trafic ?)

- Stratégie 3 : remplacer les pods en cours d'exécution un par un

  (très intéressant ; et bonne nouvelle : Kubernetes s'en charge pour nous !)

---

## Rolling updates

- Avec les rolling updates, la mise à jour d'un Deployment se fait de manière progressive

- Le Deployment pilote plusieurs Replica Sets

- Chaque Replica Set est un groupe de Pods identiques

  (avec la même image, les mêmes arguments, paramètres...)

- Durant le rolling update, nous avons au moins deux Replica Sets :

  - le "nouveau" set (correspondant à la version "cible")

  - au moins un "ancien" set

- Nous pouvons avoir plusieurs "anciens" sets

  (si nous déclenchons une nouvelle mise à jour avant que la première ne soit achevée)

---

## Update strategy

- Deux paramètres déterminent le rythme du rollout : `maxUnavailable` et `maxSurge`

- Ils peuvent être spécifiés en nombre absolu de pods ou en pourcentage du nombre de `replicas`

- À tout instant...

  - il y aura toujours au moins `replicas`-`maxUnavailable` pods disponibles

  - il n'y aura jamais plus de `replicas`+`maxSurge` pods au total

  - il y aura donc jusqu'à `maxUnavailable`+`maxSurge` pods en cours de mise à jour

- Nous disposons de la possibilité d'effectuer un rollback vers la version précédente
  <br/>(si la mise à jour échoue ou s'avère insatisfaisante de quelque manière que ce soit)

---

## Checking current rollout parameters

- Rappelons-nous comment générer des rapports sur mesure avec `kubectl` et `jq` :

.lab[

- Afficher la stratégie de rollout pour nos deployments :
  ```bash
    kubectl get deploy -o json |
            jq ".items[] | {name:.metadata.name} + .spec.strategy.rollingUpdate"
  ```

]

---

## Rolling updates in practice

- Depuis Kubernetes 1.8, nous pouvons réaliser des rolling updates avec :

  `deployments`, `daemonsets`, `statefulsets`

- Modifier l'une de ces ressources déclenchera automatiquement un rolling update

- Les rolling updates peuvent être suivis avec la sous-commande `kubectl rollout`

---

## Rolling out the new `worker` service

.lab[

- Observons ce qu'il se passe en ouvrant plusieurs terminaux, et lancez :
  ```bash
  kubectl get pods -w
  kubectl get replicasets -w
  kubectl get deployments -w
  ```

<!--
```wait NAME```
```key ^C```
-->

- Mettez à jour `worker` soit avec `kubectl edit`, soit en exécutant :
  ```bash
  kubectl set image deploy worker worker=dockercoins/worker:v0.2
  ```

]

--

Ce rollout devrait être relativement rapide. Que constate-t-on sur la web UI ?

---

## Give it some time

- Au début, on dirait que rien ne se passe (le graphique reste au même niveau)

- D'après `kubectl get deploy -w`, le `deployment` a été mis à jour très rapidement

- Mais `kubectl get pods -w` indique une autre réalité

- Les anciens `pods` sont toujours là, et ils restent à l'état `Terminating` pendant un certain temps

- Finalement, ils sont arrêtés ; et le graphique chute alors significativement

- Ce délai provient du fait que notre worker ne gère pas les signaux

- Kubernetes envoie une demande d'arrêt propre ("polite") au worker, qui l'ignore

- Après un délai de grâce (grace period), Kubernetes s'impatiente et tue le conteneur

  (La grace period est de 30 secondes, mais [peut être modifiée](https://kubernetes.io/docs/concepts/workloads/pods/pod/#termination-of-pods) si nécessaire)

---

## Rolling out something invalid

- Que se passe-t-il si nous commettons une erreur ?

.lab[

- Mettre à jour `worker` en indiquant une image inexistante :
  ```bash
  kubectl set image deploy worker worker=dockercoins/worker:v0.3
  ```

- Vérifier ce qu'il se passe :
  ```bash
  kubectl rollout status deploy worker
  ```

<!--
```wait Waiting for deployment```
```key ^C```
-->

]

--

Notre rollout est bloqué. Pour autant, l'application n'est pas hors service.

(Après une minute, elle va se stabiliser à un niveau environ 20 à 25 % plus lent.)

---

## What's going on with our rollout?

- Pourquoi notre application est-elle légèrement plus lente ?

- Parce que `MaxUnavailable=25%`

  ... Le rollout a donc arrêté 2 replicas sur les 10 disponibles

- Très bien, mais pourquoi voyons-nous 5 nouveaux replicas en cours de déploiement ?

- Parce que `MaxSurge=25%`

  ... Donc en plus de remplacer 2 replicas, le rollout en démarre 3 supplémentaires

- Le calcul a arrondi à l'inférieur de manière conservatrice le nombre de pods MaxUnavailable,
  <br/>
  mais le nombre total de pods en cours de déploiement est autorisé à atteindre 25+25=50%

---

class: extra-details

## The nitty-gritty details

- Nous partons de 10 pods en cours d'exécution pour le deployment `worker`

- Paramètres actuels : MaxUnavailable=25% et MaxSurge=25%

- Lorsque nous lançons le rollout :

  - deux replicas sont arrêtés (selon MaxUnavailable=25%)
  - deux autres sont créés (avec la nouvelle version) pour les remplacer
  - trois autres sont créés (avec la nouvelle version) selon MaxSurge=25%

- Nous avons désormais 8 replicas fonctionnels, et 5 en cours de déploiement

- Notre rollout est bloqué à ce stade !

---

## Checking the dashboard during the bad rollout

Si vous n'avez pas déployé le dashboard Kubernetes précédemment, ignorez cette diapositive.

.lab[

- Connectez-vous au dashboard déployé plus tôt

- Constatez que nous avons des échecs sur les Deployments, Pods et Replica Sets

- Est-il possible d'identifier la cause de l'échec ?

]

---

## Recovering from a bad rollout

- Nous pourrions pusher une image `v0.3`

  (la logique de retry des pods finira par la récupérer et le rollout pourra se poursuivre)

- Ou nous pourrions déclencher un rollback manuel

.lab[

<!-- ```key ^C``` -->

- Annuler le deployment et attendre la stabilisation :
  ```bash
  kubectl rollout undo deploy worker
  kubectl rollout status deploy worker
  ```

]

---

## Rolling back to an older version

- Nous sommes revenus à la version `v0.2`

- Mais cette version présente toujours un problème de performances

- Comment pouvons-nous revenir à la version précédente ?

---

## Multiple "undos"

- Que se passe-t-il si nous réexécutons `kubectl rollout undo` ?

.lab[

- Essayez :
  ```bash
  kubectl rollout undo deployment worker
  ```

- Observez la web UI, la liste des pods...

]

🤔 Cela n'a pas fonctionné.

---

## Multiple "undos" don't work

- Si nous représentons les versions successives comme une pile (stack) :

  - `kubectl rollout undo` ne dépile pas ("pop") le dernier élément

  - il copie l'élément N-1 au sommet de la pile

- Exécuter plusieurs "undos" d'affilée ne fait qu'alterner entre les deux dernières versions !

.lab[

- Revenez à nouveau à la version v0.2 :
  ```bash
  kubectl rollout undo deployment worker
  ```

]

---

## In this specific scenario

- Nos numéros de version sont faciles à deviner

- Qu'en serait-il si nous avions utilisé des hashes git ?

- Et si nous avions modifié d'autres paramètres dans la spec du Pod ?

---

## Listing versions

- Nous pouvons lister les versions successives d'un Deployment avec `kubectl rollout history`

.lab[

- Afficher l'historique de nos versions successives :
  ```bash
  kubectl rollout history deployment worker
  ```

]

Nous ne voyons pas *toutes* les révisions.

Nous pourrions par exemple voir 1, 4, 5.

(Selon le nombre d'"undos" effectués précédemment.)

---

## Explaining deployment revisions

- Ces révisions correspondent à nos Replica Sets

- Cette information est conservée dans les annotations des Replica Sets

.lab[

- Examiner les annotations de nos replica sets :
  ```bash
  kubectl describe replicasets -l app=worker | grep -A3 ^Annotations
  ```

]

---

class: extra-details

## What about the missing revisions?

- Les révisions manquantes sont stockées dans une autre annotation :

  `deployment.kubernetes.io/revision-history`

- Celles-ci ne figurent pas dans la sortie de `kubectl rollout history`

- Nous pourrions facilement reconstituer la liste complète à l'aide d'un script

  (si nous le souhaitions !)

---

## Rolling back to an older version

- `kubectl rollout undo` peut être combiné avec un numéro de révision

.lab[

- Revenir à la version "reconnue comme stable" du deployment :
  ```bash
  kubectl rollout undo deployment worker --to-revision=1
  ```

- Vérifier la web UI ou la liste des pods

]

---

class: extra-details

## Changing rollout parameters

- Nous souhaitons :

  - revenir à `v0.1`
  - être prudents sur la disponibilité (toujours conserver le nombre souhaité de workers disponibles)
  - ralentir la vitesse du rollout (ne mettre à jour qu'un seul pod à la fois) 
  - laisser le temps à nos workers de "chauffer" avant d'en démarrer d'autres

Les modifications correspondantes peuvent être déclarées dans l'extrait YAML suivant :

.small[
```yaml
spec:
  template:
    spec:
      containers:
      - name: worker
        image: dockercoins/worker:v0.1
  strategy:
    rollingUpdate:
      maxUnavailable: 0
      maxSurge: 1
  minReadySeconds: 10
```
]

---

class: extra-details

## Applying changes through a YAML patch

- Nous pourrions utiliser `kubectl edit deployment worker`

- Mais nous pouvons aussi employer `kubectl patch` avec l'extrait YAML précis présenté plus haut

.lab[

.small[

- Appliquer l'ensemble de nos modifications et attendre leur prise d'effet :
  ```bash
  kubectl patch deployment worker -p "
    spec:
      template:
        spec:
          containers:
          - name: worker
            image: dockercoins/worker:v0.1
      strategy:
        rollingUpdate:
          maxUnavailable: 0
          maxSurge: 1
      minReadySeconds: 10
    "
  kubectl rollout status deployment worker
  kubectl get deploy -o json worker |
          jq "{name:.metadata.name} + .spec.strategy.rollingUpdate"
  ```
  ] 

]

???

:FR:- Mettre à jour un déploiement
:FR:- Concept de *rolling update* et *rollback*
:FR:- Paramétrer la vitesse de déploiement