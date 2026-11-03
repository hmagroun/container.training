# Reconsidérer `kubectl logs`

- Dans cette section, nous partons du principe que nous disposons d'un Deployment comprenant plusieurs Pods

  (par ex. `pingpong` que nous avons mis à l'échelle à au moins 3 pods)

- Nous allons mettre en lumière certaines limitations de `kubectl logs`

---

## Streamer les logs de plusieurs pods

- Par défaut, `kubectl logs` affiche la sortie d'un seul Pod

.lab[

- Tenter d'inspecter la sortie des Pods associés à un Deployment :
  ```bash
  kubectl logs deploy/pingpong --tail 1 --follow
  ```

<!--
```wait using pod/pingpong-```
```keys ^C```
-->

]

`kubectl logs` n'affiche les logs que d'un unique Pod.

---

## Visualiser les logs de plusieurs pods

- Lorsque nous spécifions un nom de Deployment, les logs d'un seul pod sont affichés

- Nous pouvons afficher les logs de plusieurs pods en spécifiant un *selector*

- Si nous examinons les pods créés par le Deployment, ils portent tous le label `app=pingpong`

  (il s'agit simplement du label appliqué par défaut lors de l'utilisation de `kubectl create deployment`)

.lab[

- Afficher la dernière ligne de log de tous les pods dotés du label `app=pingpong` :
  ```bash
  kubectl logs -l app=pingpong --tail 1
  ```

]

---

## Streamer les logs de plusieurs pods

- Peut-on streamer les logs de l'ensemble de nos pods `pingpong` ?

.lab[

- Combiner les flags `-l` et `-f` :
  ```bash
  kubectl logs -l app=pingpong --tail 1 -f
  ```

<!--
```wait seq=```
```key ^C```
-->

]

*Note : combiner `-l` et `-f` n'est possible que depuis Kubernetes 1.14 !*

*Voyons pourquoi...*

---

class: extra-details

## Streamer les logs d'un grand nombre de pods

- Voyons ce qui se produit si nous tentons de streamer les logs de plus de 5 pods

.lab[

- Mettre à l'échelle (scale up) notre Deployment :
  ```bash
  kubectl scale deployment pingpong --replicas=8
  ```

- Streamer les logs :
  ```bash
  kubectl logs -l app=pingpong --tail 1 -f
  ```

<!-- ```wait error:``` -->

]

Nous obtenons un message similaire au suivant :
```
error: you are attempting to follow 8 log streams,
but maximum allowed concurency is 5,
use --max-log-requests to increase the limit
```

---

class: extra-details

## Pourquoi ne peut-on pas streamer les logs d'un grand nombre de pods ?

- `kubectl` ouvre une connexion vers l'API server par pod

- Pour chaque pod, l'API server ouvre une connexion supplémentaire vers le kubelet correspondant

- S'il y a 1000 pods dans notre Deployment, cela représente 1000 connexions entrantes + 1000 connexions sortantes sur l'API server

- Cela pourrait rapidement saturer l'API server

- Avant Kubernetes 1.14, le choix a été fait de *ne pas* autoriser les connexions multiples

- Depuis Kubernetes 1.14, cela est permis, mais limité par défaut à 5 connexions simultanées

  (cette limite peut être ajustée via `--max-log-requests`)

- Pour en savoir plus sur les motivations de ce choix, consultez la
  [PR #67573](https://github.com/kubernetes/kubernetes/pull/67573)

---

## Limites de `kubectl logs`

- Nous ne distinguons pas quel pod émet quelle ligne de log

- Si des pods sont redémarrés ou remplacés, le flux de logs s'interrompt

- Si de nouveaux pods sont ajoutés, leurs logs n'apparaissent pas automatiquement

- Pour streamer les logs de plusieurs pods, il est nécessaire de formuler un selector

- Des outils tiers existent pour combler ces lacunes

  (par ex. : [Stern](https://github.com/stern/stern))

---

class: extra-details

## `kubectl logs -l ... --tail N`

- Sous Kubernetes 1.12, cette commande affiche plusieurs lignes

- Il s'agit d'une régression apparaissant lorsque `--tail` est associé à `-l`/`--selector`

- Elle renvoie systématiquement les 10 dernières lignes de sortie pour chaque conteneur

  (au lieu du nombre de lignes spécifié dans la ligne de commande)

- Ce dysfonctionnement a été corrigé dans Kubernetes 1.13

*Consultez l'issue [#70554](https://github.com/kubernetes/kubernetes/issues/70554) pour plus de détails.*

???

:FR:- Consulter les logs avec "kubectl logs"