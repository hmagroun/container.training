# Running our first containers on Kubernetes

- Première chose essentielle : nous ne pouvons pas exécuter directement un conteneur

--

- Nous allons exécuter un Pod, et dans ce Pod il y aura un unique conteneur

--

- Dans ce conteneur au sein du Pod, nous allons exécuter une simple commande `ping`

---

class: extra-details

## Si vous utilisez Kubernetes 1.17 (ou une version antérieure)...

- Ce support de cours suppose que vous utilisez une version récente de Kubernetes

  (au moins 1.19) <!-- ##VERSION## -->

- Vous pouvez vérifier votre numéro de version avec `kubectl version`

  (regardez la section serveur)

- Dans Kubernetes 1.17 et antérieur, `kubectl run` crée un Deployment

- Si vous utilisez une version aussi ancienne :

  - elle est obsolète et n'est plus maintenue

  - Kubernetes 1.17 est [EOL depuis janvier 2021][nonactive]

  - **mettez à niveau IMMÉDIATEMENT !**

[nonactive]: https://kubernetes.io/releases/patch-releases/#non-active-branch-history

---

## Démarrer un simple Pod avec `kubectl run`

- `kubectl run` est pratique pour démarrer un seul Pod

- Nous devons spécifier au minimum un *nom* et l'image que nous voulons utiliser

- En option, nous pouvons spécifier la commande à exécuter dans le Pod

.lab[

- Envoyons un ping vers l'adresse de `localhost`, l'interface loopback :
  ```bash
  kubectl run pingpong --image alpine ping 127.0.0.1
  ```

<!-- ```hide kubectl wait pod --selector=run=pingpong --for condition=ready``` -->

]

La sortie indique qu'un Pod a été créé :
```
pod/pingpong created
```

---

## Visualiser la sortie du conteneur

- Utilisons la commande `kubectl logs`

- Elle prend un nom de Pod en argument

- Sauf indication contraire, elle affichera uniquement les logs du premier conteneur du Pod

  (Heureusement qu'il n'y en a qu'un seul dans le nôtre !)

.lab[

- Visualiser le résultat de notre commande `ping` :
  ```bash
  kubectl logs pingpong
  ```

]

---

## Suivre les logs en temps réel (Streaming)

- Tout comme `docker logs`, `kubectl logs` propose des options très pratiques :

  - `-f`/`--follow` pour streamer les logs en temps réel (à la `tail -f`)

  - `--tail` pour indiquer combien de lignes vous souhaitez afficher (depuis la fin)

  - `--since` pour récupérer les logs uniquement après un horodatage donné

.lab[

- Visualiser les derniers logs de notre commande `ping` :
  ```bash
  kubectl logs pingpong --tail 1 --follow
  ```

- Interrompez la commande avec Ctrl-C

<!--
```wait seq=3```
```keys ^C```
-->

]

---

## Mettre à l'échelle notre application

- `kubectl` met à disposition une commande simple pour scaler un workload :

  `kubectl scale TYPE NAME --replicas=HOWMANY`

- Essayons-la sur notre Pod, afin d'obtenir plus de Pods !

.lab[

- Tenter de scaler le Pod :
  ```bash
  kubectl scale pod pingpong --replicas=3
  ```

]

🤔 Nous obtenons l'erreur suivante, que signifie-t-elle ?

```
Error from server (NotFound): the server could not find the requested resource
```

---

## Scaler un Pod

- Nous ne pouvons pas "scaler un Pod"

  (ce n'est pas tout à fait exact ; nous pourrions lui attribuer plus de CPU/RAM)

- Si nous voulons plus de Pods, nous devons créer plus de Pods

  (c'est-à-dire exécuter `kubectl run` plusieurs fois)

- Il doit bien y avoir une méthode plus efficace !

  (spoiler alert : oui, il existe une bien meilleure méthode !)

---

class: extra-details

## `NotFound`

- Que signifie cette erreur ?
  ```
  Error from server (NotFound): the server could not find the requested resource
  ```

- Lorsque nous exécutons `kubectl scale THAT-RESOURCE --replicas=THAT-MANY`,
  <br/>
  cela revient à dire à Kubernetes :

  *va sur CETTE-RESSOURCE et règle le bouton de dimensionnement sur la position CETTE-QUANTITÉ*

- Les Pods ne disposent pas d'un "bouton de dimensionnement"

- Essayez d'exécuter la commande `kubectl scale pod` avec `-v6`

- Nous constatons une requête `PATCH` vers `/scale` : voilà le "bouton de dimensionnement"

  (techniquement, cela s'appelle une *subresource* du Pod)

---

## Créer plus de Pods

- Nous allons créer un ReplicaSet

  (= ensemble de replicas = ensemble de Pods identiques)

- En réalité, nous allons créer un Deployment, qui créera lui-même un ReplicaSet

- Pourquoi autant de couches d'abstraction ? Nous l'expliquerons dans un instant, rassurez-vous !

---

## Créer un Deployment exécutant `ping`

- Créons un Deployment à la place d'un simple Pod

.lab[

- Créer le Deployment ; notez bien le séparateur `--` :
  ```bash
  kubectl create deployment pingpong --image=alpine -- ping 127.0.0.1
  ```

]

- Le séparateur `--` permet de distinguer :

  - les options/flags de `kubectl create`

  - la commande à exécuter dans le conteneur

---

## Qu'est-ce qui a été créé ?

.lab[

<!-- ```hide kubectl wait pod --selector=app=pingpong --for condition=ready ``` -->

- Inspecter les ressources qui ont été créées :
  ```bash
  kubectl get all
  ```

]

Note : `kubectl get all` ne reflète pas tout à fait la réalité. Elle n'affiche pas l'intégralité des objets.

(Mais elle liste la plupart des "suspects habituels", c'est-à-dire les ressources couramment employées.)

---

## Il se passe beaucoup de choses ici !

```
NAME                            READY   STATUS        RESTARTS   AGE
pod/pingpong                    1/1     Running       0          4m17s
pod/pingpong-6ccbc77f68-kmgfn   1/1     Running       0          11s

NAME                 TYPE        CLUSTER-IP   EXTERNAL-IP   PORT(S)   AGE
service/kubernetes   ClusterIP   10.96.0.1    <none>        443/TCP   3h45

NAME                       READY   UP-TO-DATE   AVAILABLE   AGE
deployment.apps/pingpong   1/1     1            1           11s

NAME                                  DESIRED   CURRENT   READY   AGE
replicaset.apps/pingpong-6ccbc77f68   1         1         1       11s
```

Notre nouveau Pod ne s'appelle pas `pingpong`, mais `pingpong-xxxxxxxxxxx-yyyyy`.

Nous avons un Deployment nommé `pingpong`, ainsi qu'un ReplicaSet additionnel. Que se passe-t-il ?

---

## Du Deployment au Pod

Nous disposons des ressources suivantes :

- `deployment.apps/pingpong`

  Il s'agit du Deployment que nous venons de créer.

- `replicaset.apps/pingpong-xxxxxxxxxx`

  Il s'agit du Replica Set créé par ce Deployment.

- `pod/pingpong-xxxxxxxxxx-yyyyy`

  Il s'agit d'un *pod* créé par le Replica Set.

Détaillons ce que représentent ces différents composants.

---

## Pod

- Peut embarquer un ou plusieurs conteneurs

- S'exécute sur un seul et unique Node

  (Un Pod ne peut pas être à cheval sur plusieurs nodes)

- Les Pods ne peuvent pas être déplacés

  (par ex. lors d'une panne de node)

- Les Pods ne peuvent pas être scalés horizontalement

  (sauf en créant manuellement davantage de Pods)

---

class: extra-details

## Détails sur le Pod

- Un Pod n'est pas un processus ; c'est un environnement d'exécution pour conteneurs

  - il ne peut pas être "redémarré"

  - il ne peut pas "crasher"

- Les conteneurs au sein d'un Pod peuvent crasher

- Ils peuvent être redémarrés ou non

  (selon la restart policy définie sur le Pod)

- Si tous les conteneurs se terminent avec succès, le Pod passe en phase "Succeeded"

- Si certains conteneurs échouent et ne sont pas redémarrés, le Pod passe en phase "Failed"

---

## Replica Set

- Ensemble de Pods identiques (répliqués)

- Défini par un template de Pod + un nombre de réplicas cibles

- S'il n'y a pas assez de Pods, le Replica Set en crée d'autres

  (par ex. lors d'une panne de node ; ou tout simplement lors d'un scale up)

- S'il y a trop de Pods, le Replica Set en supprime

  (par ex. si un node déconnecté redevient joignable ; ou lors d'un scale down)

- Nous pouvons dimensionner (scale up/down) un Replica Set

  - nous modifions le manifest du Replica Set

  - en conséquence, le contrôleur du Replica Set crée ou supprime des Pods

---

## Deployment

- Les Replica Sets pilotent des Pods *identiques*

- Les Deployments sont exploités pour effectuer des rollouts de Pods différents

  (image différente, commande, variables d'environnement, ...)

- Lorsque nous mettons à jour un Deployment avec une nouvelle définition de Pod :

  - un nouveau Replica Set est créé avec cette nouvelle définition

  - ce nouveau Replica Set est progressivement mis à l'échelle ascendante (scaled up)

  - en parallèle, l'ancien (ou les anciens) Replica Set(s) est(sont) mis à l'échelle descendante (scaled down)

- Il s'agit d'un *rolling update*, réduisant au maximum l'indisponibilité de l'application

- Lorsque nous dimensionnons (scale up/down) un Deployment, il applique cette action à son Replica Set

---

## Peut-on scaler maintenant ?

- Réessayons `kubectl scale`, mais cette fois sur le Deployment !

.lab[

- Scaler notre Deployment `pingpong` :
  ```bash
  kubectl scale deployment pingpong --replicas 3
  ```

- Notez qu'on peut aussi l'écrire sous cette forme :
  ```bash
  kubectl scale deployment/pingpong --replicas 3
  ```

- Vérifier que nous avons désormais plusieurs pods :
  ```bash
  kubectl get pods
  ```

]

---

class: extra-details

## Scaler un Replica Set

- Que se passerait-il si nous scalions directement le Replica Set plutôt que le Deployment ?

- Le Deployment le constaterait immédiatement et réalignerait le nombre de réplicas sur sa configuration d'origine

- Le Replica Set garantit que nous avons le bon nombre de Pods

- Le Deployment garantit que le Replica Set dispose de la taille adéquate

  (conceptuellement, il délègue la gestion brute des Pods au Replica Set)

- Cela peut paraître curieux (pourquoi cette couche superflue ?), mais prendra tout son sens rapidement

  (notamment lorsque nous analyserons le fonctionnement des rolling updates !)

---

## Consulter les logs d'un Deployment

- `kubectl logs` attend un nom de Pod

- Mais la commande peut aussi fonctionner avec une référence *type/name*

  (par ex. `deployment/pingpong`)

.lab[

- Consulter le résultat de notre commande `ping` :
  ```bash
  kubectl logs deploy/pingpong --tail 2
  ```

]

- Cela nous affiche les logs du premier Pod du Deployment

- Nous verrons plus loin comment extraire les logs de *tous* les Pods !

---

## Résilience

- Le *Deployment* `pingpong` surveille son *ReplicaSet*

- Le *ReplicaSet* garantit que le bon nombre de *Pods* est en cours d'exécution

- Que se passe-t-il si des Pods disparaissent ?

.lab[

- Dans un terminal séparé, surveillez la liste des pods :
  ```bash
  watch kubectl get pods
  ```

<!--
```wait Every 2.0s```
```tmux split-pane -v```
-->

- Supprimez le Pod actuellement affiché par `kubectl logs` :
  ```
  kubectl delete pod pingpong-xxxxxxxxxx-yyyyy
  ```

<!--
```tmux select-pane -t 0```
```copy pingpong-[^-]*-.....```
```tmux last-pane```
```keys kubectl delete pod ```
```paste```
```key ^J```
```check```
-->

]

---

## Que s'est-il passé ?

- `kubectl delete pod` arrête le Pod de manière propre (graceful termination)

  (en lui transmettant le signal TERM et en attendant son extinction complète)

- Dès que le Pod bascule à l'état "Terminating", le Replica Set le remplace

- Mais nous continuons de voir la sortie du Pod en cours d'arrêt ("Terminating") dans `kubectl logs`

- Jusqu'à 30 secondes plus tard, moment où le délai de grâce expire

- Le Pod est alors tué de force, et `kubectl logs` se termine

---

## Supprimer un Pod autonome (standalone)

- Que se passe-t-il si nous supprimons un Pod isolé ?
 
  (comme le premier Pod `pingpong` que nous avons créé)

.lab[

- Supprimer le Pod :
  ```bash
  kubectl delete pod pingpong
  ```

<!--
```key ^D```
```key ^C```
-->

]

- Aucun Pod de remplacement n'est créé car aucun *controller* ne le supervise

- C'est la raison pour laquelle nous n'utiliserons presque jamais de Pods autonomes en pratique

  (hormis par ex. pour du debugging ponctuel ou l'exécution rapide d'une tâche supervisée)

???

:FR:- Créer un pod et un déploiement