# Scaling our demo app

- Notre objectif final est d'obtenir davantage de DockerCoins

  (c'est-à-dire augmenter le nombre de boucles par seconde affiché sur la web UI)

- Regardons à nouveau l'architecture :

  ![DockerCoins architecture](images/dockercoins-diagram.png)

- La boucle est exécutée dans le worker ;
  pourrions-nous essayer d'ajouter davantage de workers ?

---

## Adding another worker

- Tout ce que nous avons à faire, c'est de scaler le Deployment `worker`

.lab[

- Ouvrez un nouveau terminal pour garder un œil sur nos pods :
  ```bash
  kubectl get pods -w
  ```

<!--
```wait RESTARTS```
```tmux split-pane -h```
-->

- À présent, créez davantage de replicas de `worker` :
  ```bash
  kubectl scale deployment worker --replicas=2
  ```

]

Après quelques secondes, le graphe dans la web UI devrait apparaître.

---

## Adding more workers

- Si 2 workers doublent notre vitesse (2x), qu'en est-il avec 3 workers ?

.lab[

- Scaler encore le Deployment `worker` :
  ```bash
  kubectl scale deployment worker --replicas=3
  ```

]

Le graphe dans la web UI devrait monter à nouveau.

(C'est très prometteur ! Nous allons être RICHES !)

---

## Adding even more workers

- Voyons si 10 workers nous donnent une vitesse multipliée par 10 (10x) !

.lab[

- Scaler le Deployment `worker` avec un nombre plus élevé :
  ```bash
  kubectl scale deployment worker --replicas=10
  ```

<!--
```key ^D```
```key ^C```
-->

]

--

Le graphe va plafonner à 10 hashes/seconde.

(Nous pouvons ajouter autant de workers que nous voulons : nous ne dépasserons jamais 10 hashes/seconde.)

---

class: extra-details

## Didn't we briefly exceed 10 hashes/second?

- On peut en avoir *l'impression*, parce que la web UI affiche la vitesse instantanée

- La vitesse instantanée peut temporairement dépasser 10 hashes/seconde

- La vitesse moyenne ne le peut pas

- La vitesse instantanée peut être biaisée en raison de son mode de calcul

---

class: extra-details

## Why instant speed is misleading

- La vitesse instantanée est calculée côté client par la web UI

- La web UI vérifie le compteur de hashes une fois par seconde
  <br/>
  (et effectue un calcul de vitesse classique (h2-h1)/(t2-t1))

- Le compteur est mis à jour une fois par seconde par les workers

- Ces intervalles ne sont pas parfaitement exacts
  <br/>
  (par ex. l'intervalle d'évaluation de la web UI dépend du JavaScript côté client)

- Parfois, entre deux mesures du compteur par la web UI,
  <br/>
  les workers ont le temps de mettre à jour le compteur *deux fois*

- Pendant ce cycle, la vitesse instantanée semblera nettement plus élevée
  <br/>
  (mais cela sera compensé par une vitesse instantanée plus faible avant et après)

---

## Why are we stuck at 10 hashes per second?

- S'il s'agissait de code de production de haute qualité, nous disposerions d'instrumentation

  (Datadog, Honeycomb, New Relic, statsd, Sumologic, ...)

- Ce n'est pas le cas ici !

- Peut-être pourrions-nous benchmarker nos web services ?

  (avec des outils comme `ab`, ou encore plus simple, `httping`)

---

## Benchmarking our web services

- Nous souhaitons examiner `hasher` et `rng`

- Nous allons utiliser `httping`

- Cela fonctionne exactement comme `ping`, mais avec des requêtes HTTP `GET`

  (cela mesure le temps nécessaire pour exécuter une requête `GET`)

- L'outil s'utilise ainsi :
  ```
  httping [-c count] http://host:port/path
  ```

- Ou encore plus simplement :
  ```
  httping ip.ad.dr.ess
  ```

- Nous allons utiliser `httping` sur les adresses ClusterIP de nos services

---

## Obtaining ClusterIP addresses

- Nous pouvons simplement examiner la sortie de `kubectl get services`

- Ou le faire de manière programmatique, comme dans l'exemple ci-dessous

.lab[

- Récupérer les adresses IP :
  ```bash
  HASHER=$(kubectl get svc hasher -o go-template={{.spec.clusterIP}})
  RNG=$(kubectl get svc rng -o go-template={{.spec.clusterIP}})
  ```

]

Nous pouvons désormais accéder aux adresses IP de nos services via `$HASHER` et `$RNG`.

---

## Checking `hasher` and `rng` response times

.lab[

- Vérifier les temps de réponse pour les deux services :
  ```bash
  httping -c 3 $HASHER
  httping -c 3 $RNG
  ```

]

- `hasher` fonctionne bien (la réponse devrait prendre quelques millisecondes)

- `rng` ne fonctionne pas bien (la réponse devrait prendre environ 700 millisecondes s'il y a 10 workers)

- Quelque chose ne va pas avec `rng`, mais... quoi ?

???

:FR:- *Scale up* de l'application de démo