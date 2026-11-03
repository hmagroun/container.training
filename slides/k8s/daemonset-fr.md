# Daemon sets

- Nous souhaitons dimensionner (scaler) `rng` d'une façon différente de ce que nous avons fait pour `worker`

- Nous voulons une (et exactement une) instance de `rng` par node

- Nous *ne voulons pas* de deux instances de `rng` sur le même node

- Nous allons accomplir cela à l'aide d'un *daemon set*

---

## Pourquoi pas un deployment ?

- Ne pouvons-nous pas simplement exécuter `kubectl scale deployment rng --replicas=...` ?

--

- Rien ne garantit que les conteneurs `rng` seront distribués de manière homogène

- Si nous ajoutons des nodes ultérieurement, ils n'exécuteront pas automatiquement une copie de `rng`

- Si nous retirons (ou redémarrons) un node, un conteneur `rng` redémarrera ailleurs

  (et nous nous retrouverons avec deux instances de `rng` sur le même node)

- À l'inverse, un daemon set démarre un pod par node et maintient cet état

  (au fur et à mesure que des nodes sont ajoutés ou retirés)

---

## Les daemon sets en pratique

- Les daemon sets sont parfaits pour les processus s'exécutant par node à l'échelle du cluster :

  - composants réseau comme `kube-proxy`, `calico`, `cilium`...

  - composants de stockage (par ex. plugins CSI)

  - agents de métriques et monitoring

  - outils d'administration matérielle (par ex. agents HBA SCSI/FC)

  - etc.

- Ils peuvent également être restreints pour s'exécuter [uniquement sur certains nodes](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/#running-pods-on-only-some-nodes)

---

## Créer un daemon set

<!-- ##VERSION## -->

- Malheureusement, avec Kubernetes 1.36, la CLI ne permet pas de créer directement des daemon sets

--

- Plus précisément : elle ne dispose pas de sous-commande dédiée pour créer un daemon set

--

- Mais tout type de ressource peut systématiquement être créé en fournissant une description YAML :
  ```bash
  kubectl apply -f foo.yaml
  ```

--

- Comment générer le fichier YAML de notre daemon set ?

--

  - option 1 : [lire la documentation](https://kubernetes.io/docs/concepts/workloads/controllers/daemonset/#create-a-daemonset)

--

  - option 2 : s'en sortir avec `vi`

---

## Créer le fichier YAML pour notre daemon set

- Les DaemonSets et les Deployments devraient être *très similaires*

- Ils définissent tous les deux la manière d'instancier des Pods

- Peut-on transformer un Deployment en DaemonSet ? 🤔

- Essayons !

---

## Générer un manifest de Deployment

- Utilisons `kubectl create deployment -o yaml --dry-run=client`

.lab[

- Générer le YAML d'un Deployment :
  ```bash
    kubectl create deployment rng --image=dockercoins/rng:v0.1 \
            -o yaml --dry-run=client
  ```
  
- L'enregistrer dans un fichier :
  ```bash
    kubectl create deployment rng --image=dockercoins/rng:v0.1 \
            -o yaml --dry-run=client \
            > rng.yaml
  ```
  
]

---

## Modifier le `kind`

- Éditez le manifest YAML et remplacez `Deployment` par `DaemonSet`

.lab[

- Éditer le fichier YAML et effectuer la modification

<!--
```bash vim rng.yaml```
```wait kind: Deployment```
```keys /Deployment```
```key ^J```
```keys cwDaemonSet```
```key ^[``` ]
```keys :wq```
```key ^J```
-->

- Ou, alternativement :
  ```bash
  sed -i "s/kind: Deployment/kind: DaemonSet/"
  ```

]

---

## Créer le DaemonSet

- Voyons si notre manifest de DaemonSet est valide !

.lab[

- Tenter d'appliquer notre nouveau YAML avec `kubectl apply` :
  ```bash
  kubectl apply -f rng.yaml
  ```

]

--

- Malheureusement, cela ne fonctionne pas !

---

## Comprendre l'erreur

- Le message d'erreur principal est le suivant :
  ```
  unknown field "spec.replicas", unknown field "spec.strategy"
  ```

--

- *De toute évidence,* cela n'a aucun sens de définir un nombre de replicas pour un daemon set

  (le champ `spec.strategy` est également spécifique aux Deployments)

--

- Solution de contournement : corriger le YAML en supprimant ces champs

---

## Résoudre le problème

- Retirons le champ `replicas` et réessayons

.lab[

- Éditer le fichier `rng.yaml` et supprimer la ligne `replicas:`

- Tenter ensuite de créer à nouveau le DaemonSet :
  ```bash
  kubectl apply -f rng.yaml
  ```

]

- Cette fois-ci, cela devrait fonctionner !

---

## Vérifier ce qui a été réalisé

- Avons-nous transformé notre `deployment` en `daemonset` ?

.lab[

- Examiner les ressources actuellement présentes :
  ```bash
  kubectl get all
  ```

]

--

Nous constatons la présence de deux ressources nommées `rng` :

- le *deployment* qui existait déjà

- le *daemon set* que nous venons de créer

Nous avons également un pod de trop.
<br/>
(Le pod associé au *deployment* est toujours présent.)

---

## `deploy/rng` et `ds/rng`

- Vous pouvez avoir différents types de ressources portant le même nom

  (c'est-à-dire un *deployment* et un *daemon set* nommés tous deux `rng`)

- Nous avons toujours l'ancien *deployment* `rng`

  ```
NAME                       DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/rng        1         1         1            1           18m
  ```

- Mais désormais, nous avons aussi le nouveau *daemon set* `rng`

  ```
NAME                DESIRED  CURRENT  READY  UP-TO-DATE  AVAILABLE  NODE SELECTOR  AGE
daemonset.apps/rng  2        2        2      2           2          <none>         9s
  ```

---

## Trop de pods

- Si nous vérifions avec `kubectl get pods`, nous observons :

  - *un pod* pour le deployment (nommé `rng-xxxxxxxxxx-yyyyy`)

  - *un pod par node* pour le daemon set (nommé `rng-zzzzz`)

  ```
  NAME                        READY     STATUS    RESTARTS   AGE
  rng-54f57d4d49-7pt82        1/1       Running   0          11m
  rng-b85tm                   1/1       Running   0          25s
  rng-hfbrr                   1/1       Running   0          25s
  [...]
  ```

--

Le daemon set a créé un pod par node, excepté sur le node du control plane.

Le node du control plane porte des [taints](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/) empêchant les pods de s'y exécuter.

(Pour ordonnancer un pod sur ce node malgré tout, le pod nécessitera les [tolerations](https://kubernetes.io/docs/concepts/configuration/taint-and-toleration/) adaptées.)

.footnote[(Un pod manquant ? Nous n'exécutons pas ces pods sur le node hébergeant le control plane.)]

---

## Est-ce que cela fonctionne ?

- Observez la web UI

--

- Le graphe devrait désormais dépasser les 10 hashes par seconde !

--

- Il semble que les nouveaux pods créés traitent le trafic correctement

- Comment et pourquoi cela s'est-il produit ?

  (Nous n'avons fait aucune manipulation spécifique pour les rattacher au load balancer du service `rng` !)

---

# Labels et selectors

- Le *service* `rng` répartit la charge (load balancing) des requêtes vers un ensemble de pods

- Cet ensemble de pods est déterminé par le *selector* du service `rng`

.lab[

- Vérifier le *selector* dans la définition du service `rng` :
  ```bash
  kubectl describe service rng
  ```

]

- Le selector est `app=rng`

- Cela signifie « tous les pods possédant le label `app=rng` »

  (Ils peuvent également porter d'autres labels en plus, aucun problème !)

---

## Évaluation des selectors

- Nous pouvons utiliser les selectors avec de nombreuses commandes `kubectl`

- Par exemple, avec `kubectl get`, `kubectl logs`, `kubectl delete`... et bien d'autres

.lab[

- Obtenir la liste des pods correspondant au selector `app=rng` :
  ```bash
  kubectl get pods -l app=rng
  kubectl get pods --selector app=rng
  ```

]

Mais... d'où vient ce label `app=rng` sur ces pods (en particulier sur les *nouveaux*) ?

---

## D'où proviennent les labels ?

- Lorsque nous créons un deployment avec `kubectl create deployment rng`,
  <br/>ce deployment reçoit le label `app=rng`

- Les replica sets créés par ce deployment reçoivent également le label `app=rng`

- Les pods créés par ces replica sets reçoivent également le label `app=rng`

- Lorsque nous avons créé le daemon set à partir du deployment, nous avons réutilisé la même spec

- Par conséquent, les pods créés par le daemon set héritent des mêmes labels

.footnote[Note : lorsque nous utilisons `kubectl run stuff`, le label attribué est `run=stuff`.]

---

## Mettre à jour la configuration du load balancer

- Nous souhaitons retirer un pod du load balancer

- Que se passerait-il si nous supprimions ce pod avec `kubectl delete pod ...` ?

--

  Il serait immédiatement recréé (par le replica set ou le daemon set)

--

- Que se passerait-il si nous retirions le label `app=rng` de ce pod ?

--

  Il serait *également* immédiatement recréé

--

  Pourquoi ?!?

---

## Les selectors pour les replica sets et les daemon sets

- La « mission » d'un replica set est :

  « S'assurer que le nombre attendu de pods correspondant à cette spec est respecté ! »

- La « mission » d'un daemon set est :

  « S'assurer qu'un pod correspondant à cette spec tourne sur chaque node ! »

--

- *En réalité,* les replica sets et les daemon sets ne vérifient pas les spécifications des pods

- Ils possèdent simplement un *selector*, et recherchent les pods correspondant à ce selector

- Oui, nous pouvons les tromper en créant manuellement des pods portant les « bons » labels

- Conclusion : si nous retirons notre label `app=rng`...

 ... Le pod « disparaît » aux yeux de son parent, qui en recrée immédiatement un autre pour le remplacer

---

class: extra-details

## Isolation des replica sets et des daemon sets

- Puisque le daemon set `rng` et le replica set `rng` utilisent tous deux `app=rng`...

  ... Pourquoi ne « capturent-ils » pas les pods de l'un et de l'autre ?

--

- Les *replica sets* utilisent un selector plus restrictif, visible avec `kubectl describe`

  (De la forme `app=rng,pod-template-hash=abcd1234`)

- Les *daemon sets* utilisent également un selector plus restrictif, mais invisible

  (De la forme `app=rng,controller-revision-hash=abcd1234`)

- En conséquence, chaque controller ne « voit » que les pods qu'il supervise directement

---

## Retirer un pod du load balancer

- Actuellement, le service `rng` est défini par le selector `app=rng`

- La seule façon d'exclure un pod consisterait à retirer ou modifier son label `app`

- ... Mais cela déclencherait immédiatement la création d'un nouveau pod en remplacement !

- Quelle est donc la solution ?

--

- Nous devons modifier le selector du service `rng` !

- Ajoutons un label supplémentaire à ce selector (par ex. `active=yes`)

---

## Selectors avec plusieurs labels

- Si un selector spécifie plusieurs labels, ils sont combinés par un *ET* logique

  (en d'autres termes : les pods doivent correspondre à l'ensemble des labels)

- Nous ne pouvons pas exprimer de *OU* logique

  (par ex. `app=api ET (release=prod OU release=preprod)`)

- Nous pouvons néanmoins appliquer autant de labels supplémentaires que souhaité à nos pods :

  - utiliser le selector `app=api ET prod-or-preprod=yes`

  - ajouter `prod-or-preprod=yes` aux deux groupes de pods

- Nous verrons plus tard que dans d'autres contextes, nous pouvons utiliser des selectors plus avancés

---

## Le plan d'action

1. Ajouter le label `active=yes` à l'ensemble de nos pods `rng`

2. Mettre à jour le selector du service `rng` pour y inclure également `active=yes`

3. Activer/désactiver l'acheminement du trafic vers un pod en ajoutant/retirant manuellement le label `active`

4. Victoire !

*Note : si nous inversons les étapes 1 et 2, cela provoquera une brève
interruption de service, car il y aura un intervalle de temps
pendant lequel le selector du service ne correspondra à aucun pod.
Pendant cette période, les requêtes vers le service tomberont en timeout.
En procédant dans l'ordre décrit ci-dessus, nous garantissons
une transition sans coupure.*

---

## Ajouter des labels aux pods

- Nous voulons ajouter le label `active=yes` à tous les pods portant `app=rng`

- Nous pourrions éditer chaque pod un par un avec `kubectl edit`...

- ... Ou utiliser `kubectl label` pour les labelliser d'un coup

- `kubectl label` accepte lui-même des selectors

.lab[

- Ajouter `active=yes` à tous les pods ayant `app=rng` :
  ```bash
  kubectl label pods -l app=rng active=yes
  ```

]

---

## Mettre à jour le selector du service

- Nous devons éditer la spécification du service

- Rappel : dans la définition du service, nous verrons `app: rng` à deux endroits

  - le label du service lui-même (nous n'avons pas besoin d'y toucher)

  - le selector du service (c'est celui que nous souhaitons modifier)

.lab[

- Mettre à jour le service pour ajouter `active: yes` à son selector :
  ```bash
  kubectl edit service rng
  ```

<!--
```wait Please edit the object below```
```keys /app: rng```
```key ^J```
```keys noactive: yes```
```key ^[``` ]
```keys :wq```
```key ^J```
-->

]

--

... Et nous obtenons alors *l'erreur la plus étrange qui soit.* Pourquoi ?

---

## Quand le parseur YAML est un peu trop zélé

- Les parseurs YAML tentent d'interpréter les types :

  - `xyz` devient la chaîne `"xyz"`

  - `42` devient l'entier `42`

  - `yes` devient la valeur booléenne `true`

- Si nous voulons la chaîne `"42"` ou la chaîne `"yes"`, nous devons impérativement les entourer de guillemets

- Il faut donc écrire `active: "yes"`

.footnote[Pour la petite histoire : si nous avions utilisé « ja », « oui », « si »... comme valeur, cela aurait fonctionné !]

---

## Mettre à jour le selector du service, acte 2

.lab[

- Mettre à jour le manifest YAML du service

- Ajouter `active: "yes"` à son selector

<!--
```wait Please edit the object below```
```keys /yes```
```key ^J```
```keys cw"yes"```
```key ^[``` ]
```keys :wq```
```key ^J```
-->

]

Cette fois, cela devrait fonctionner !

Si tout s'est déroulé correctement, la web UI ne devrait afficher aucune interruption.

---

## Mettre à jour les labels

- Nous voulons désactiver le pod généré par le deployment

- Il nous suffit de retirer le label `active` de ce pod

- Pour cibler ce pod, nous pouvons utiliser son nom

- ... Ou nous appuyer sur le fait qu'il est le seul à porter un label `pod-template-hash`

- Bon à savoir :

  - `kubectl label ... foo=` ne supprime pas un label (il lui affecte une chaîne vide)

  - pour supprimer le label `foo`, utilisez `kubectl label ... foo-`

  - pour modifier un label existant, il faut ajouter le flag `--overwrite`

---

## Retirer un pod du load balancer

.lab[

- Dans un terminal, observez les logs de ce pod :
  ```bash
  POD=$(kubectl get pod -l app=rng,pod-template-hash -o name)
  kubectl logs --tail 1 --follow $POD
  ```
  (Nous devrions observer un flux continu de logs HTTP)

<!--
```wait HTTP/1.1```
```tmux split-pane -v```
-->

- Dans un autre terminal, retirez le label du pod :
  ```bash
  kubectl label pod -l app=rng,pod-template-hash active-
  ```
  (Le flux de logs HTTP devrait s'interrompre instantanément)

<!--
```key ^D```
```key ^C```
-->

]

Une légère variation peut être visible sur la web UI (étant donné que nous avons réduit un peu
la capacité du service `rng`). Si nous retirons d'autres pods,
l'effet sera plus marqué.

---

class: extra-details

## Mettre à jour le daemon set

- Si nous agrandissons le cluster (scale up) en ajoutant de nouveaux nodes, le daemon set créera de nouveaux pods

- Ces pods n'auront pas le label `active=yes`

- Si nous souhaitons que ces pods portent ce label dès leur création, nous devons mettre à jour la spec du daemon set

- Nous pouvons faire cela avec par exemple `kubectl edit daemonset rng`

---

class: extra-details

## Des ressources imbriquées dans vos ressources

- Rappel : un daemon set est une ressource chargée de créer d'autres ressources !

- Il ne faut pas confondre :

  - le ou les label(s) de la ressource (dans le bloc `metadata` au début)

  - le selector de la ressource (dans le bloc `spec`)

  - le ou les label(s) de la ou des ressource(s) créée(s) par la première ressource (dans le bloc `template`)

- Nous devrions mettre à jour le selector ainsi que le template

  (les labels dans `metadata` sont facultatifs)

- Le template doit obligatoirement correspondre au selector

  (la ressource refusera de créer des ressources qu'elle ne sélectionnerait pas elle-même)

---

## Labels et debugging

- Lorsqu'un pod a un comportement anormal, nous pouvons le supprimer : un autre sera recréé

- Mais nous pouvons aussi modifier ses labels

- Il sera retiré du load balancer (il ne recevra plus de trafic)

- Un autre pod sera recréé immédiatement

- Mais le pod défaillant reste présent, et nous pouvons l'inspecter et le debugger à loisir

- Nous pouvons même le réinsérer dans la rotation si nécessaire

  (Très pratique pour investiguer des bugs intermittents ou difficiles à reproduire)

---

## Labels et contrôle avancé des déploiements

- Réciproquement, nous pouvons ajouter des pods correspondant au selector d'un service

- Ces pods recevront alors des requêtes et traiteront du trafic

- Exemples :

  - un pod ponctuel avec tous les flags de debug activés, pour collecter des logs

  - des pods instanciés automatiquement, mais intégrés à la rotation dans un second temps
    <br/>
    (en appliquant leur label au moment opportun)

- Cela fournit les briques de base pour des déploiements de type canary et blue/green

---

class: extra-details

## Selectors de labels avancés

- Comme vu plus haut, les selectors de services sont restreints à un `ET` logique

- Mais dans de nombreux autres pans de l'API Kubernetes, nous pouvons utiliser des selectors complexes

  (par ex. Deployment, ReplicaSet, DaemonSet, NetworkPolicy...)

- Ceux-ci autorisent des opérations supplémentaires, notamment :

  - vérifier la présence (ou l'absence) d'un label

  - vérifier si la valeur d'un label appartient (ou n'appartient pas) à un ensemble donné

- Documentation de référence :

  [Service spec](https://kubernetes.io/docs/reference/kubernetes-api/service-resources/service-v1/#ServiceSpec),
  [LabelSelector spec](https://kubernetes.io/docs/reference/kubernetes-api/common-definitions/label-selector/),
  [label selector doc](https://kubernetes.io/docs/concepts/overview/working-with-objects/labels/#label-selectors)

---

class: extra-details

## Exemple de selector avancé

```yaml
  theSelector:
    matchLabels:
      app: portal
      component: api
    matchExpressions:
    - key: release
      operator: In
      values: [ production, preproduction ]
    - key: signed-off-by
      operator: Exists
```

Ce selector cible les pods qui satisfont *toutes* les conditions énumérées.

`operator` peut prendre les valeurs `In`, `NotIn`, `Exists`, `DoesNotExist`.

Un selector `nil` ne sélectionne *rien*, un selector `{}` sélectionne *tout*.
<br/>
(Car cela équivaut à « sélectionner tous les pods satisfaisant au moins zéro condition ».)

---

class: extra-details

## Services et Endpoints

- Chaque Service est associé à une ressource Endpoints correspondante

  (voir `kubectl get endpoints` ou `kubectl get ep`)

- Cette ressource Endpoints est consommée par divers controllers

  (par ex. `kube-proxy` lors de la configuration des règles `iptables` pour les services ClusterIP)

- Ces Endpoints sont alimentés (et mis à jour) automatiquement à l'aide du selector du Service

- Nous pouvons modifier les Endpoints manuellement, mais nos changements seront écrasés

- ... Sauf si le selector du Service est vide !

---

class: extra-details

## Selector de Service vide

- Si le selector d'un service est vide, les Endpoints ne sont pas mis à jour automatiquement

  (mais nous pouvons continuer à les configurer manuellement)

- Cela nous permet de créer des Services pointant vers des cibles arbitraires

  (potentiellement hors du cluster, ou vers des services qui ne tournent pas dans des pods)

- Autre cas d'usage : le service `kubernetes` dans le namespace `default`

  (ses Endpoints sont administrés automatiquement par l'API server)

???

:FR:- Utilisation de Daemon Sets