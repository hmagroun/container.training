# Running our application on Kubernetes

- Nous pouvons à présent déployer notre code (ainsi qu'une instance redis)

.lab[

- Déployer `redis` :
  ```bash
  kubectl create deployment redis --image=redis
  ```

- Déployer l'ensemble des autres composants :
  ```bash
  kubectl create deployment hasher --image=dockercoins/hasher:v0.1
  kubectl create deployment rng --image=dockercoins/rng:v0.1
  kubectl create deployment webui --image=dockercoins/webui:v0.1
  kubectl create deployment worker --image=dockercoins/worker:v0.1
  ```

]

---

class: extra-details

## Déployer d'autres images

- Si nous souhaitions déployer des images depuis un autre registry...

- ... Ou avec un tag différent...

- ... Nous pourrions utiliser le script suivant :

```bash
  REGISTRY=dockercoins
  TAG=v0.1
  for SERVICE in hasher rng webui worker; do
    kubectl create deployment $SERVICE --image=$REGISTRY/$SERVICE:$TAG
  done
```

---

## Est-ce que cela fonctionne ?

- Après avoir attendu la fin du déploiement, examinons les logs !

  (Astuce : utilisez `kubectl get deploy -w` pour surveiller les événements des deployments)

.lab[

<!-- ```hide
kubectl wait deploy/rng --for condition=available
kubectl wait deploy/worker --for condition=available
``` -->

- Examiner les logs :
  ```bash
  kubectl logs deploy/rng
  kubectl logs deploy/worker
  ```

]

--

🤔 `rng` fonctionne correctement... Mais pas `worker`.

--

💡 Ah oui, bien sûr ! Nous avons oublié d'`expose`.

---

## Interconnecter les conteneurs

- Trois deployments doivent être joignables par les autres : `hasher`, `redis`, `rng`

- `worker` n'a pas besoin d'être exposé

- `webui` sera traité plus tard

.lab[

- Exposer chaque deployment en indiquant le port adéquat :
  ```bash
  kubectl expose deployment redis --port 6379
  kubectl expose deployment rng --port 80
  kubectl expose deployment hasher --port 80
  ```

]

---

## Est-ce que cela fonctionne maintenant ?

- Le `worker` intègre une boucle infinie qui réessaie 10 secondes après une erreur

.lab[

- Streamer les logs du worker :
  ```bash
  kubectl logs deploy/worker --follow
  ```

  (Laissez-lui environ 10 secondes pour récupérer)

<!--
```wait units of work done, updating hash counter```
```key ^C```
-->

]

--

Nous devrions maintenant voir le `worker`, enfin, travailler sans encombre.

---

## Exposer des services pour un accès externe

- Nous souhaitons désormais accéder à la Web UI

- Nous allons l'exposer avec un `NodePort`

  (exactement comme nous l'avons fait pour le registry)

.lab[

- Créer un service `NodePort` pour la Web UI :
  ```bash
  kubectl expose deploy/webui --type=NodePort --port=80
  ```

- Vérifier le port qui a été alloué :
  ```bash
  kubectl get svc
  ```

]

---

## Accéder à la web UI

- Nous pouvons à présent nous connecter à *n'importe quel node*, sur le NodePort alloué, pour afficher la web UI

.lab[

- Ouvrir la web UI dans votre navigateur (http://node-ip-address:3xxxx/)

<!-- ```open http://node1:3xxxx/``` -->

]

--

Oui, la mise à jour peut prendre un petit moment. *(Narrateur : c'était le DNS.)*

--

*Parfait, nous sommes revenus au point de départ, lorsque tout tournait sur un unique node !*

???

:FR:- Faire tourner l'application de démo sur Kubernetes