# Shipping images with a registry

- Initialement, notre application s'exécutait sur un unique node

- Nous pouvions *builder* et *exécuter* au même endroit

- Par conséquent, nous n'avions pas besoin de *distribuer* (ship) quoi que ce soit

- Maintenant que nous souhaitons l'exécuter sur un cluster, la situation est différente

- La méthode la plus simple pour distribuer des images de conteneurs consiste à utiliser un registry

---

## Fonctionnement des registries Docker (rappel)

- Que se passe-t-il lorsque nous exécutons `docker run alpine` ?

- Si l'Engine doit puller l'image `alpine`, il résout son nom en `library/alpine`

- `library/alpine` est ensuite développé en `index.docker.io/library/alpine`

- L'Engine contacte `index.docker.io` pour récupérer `library/alpine:latest`

- Pour utiliser un autre emplacement qu'`index.docker.io`, nous le spécifions directement dans le nom de l'image

- Exemples :
  ```bash
  docker pull gcr.io/google-containers/alpine-with-bash:1.0

  docker build -t registry.mycompany.io:5000/myimage:awesome .
  docker push registry.mycompany.io:5000/myimage:awesome
  ```

---

## Exécuter DockerCoins sur Kubernetes

- Créer un deployment pour chaque composant

  (hasher, redis, rng, webui, worker)

- Exposer les deployments devant accepter des connexions

  (hasher, redis, rng, webui)

- Pour redis, nous pouvons utiliser l'image officielle redis

- Pour les 4 autres, nous devons builder les images et les pusher sur un registry

---

## Builder et distribuer des images

- Il existe de *nombreuses* approches !

- Manuellement :

  - builder localement (avec `docker build` ou un autre outil)

  - pusher vers le registry

- Automatiquement :

  - builder et tester localement

  - une fois prêt, commit et push sur un repository de code

  - le repository de code notifie un système de build automatisé

  - ce système récupère le code, le builde et pushe l'image vers le registry

---

## Quel registry utiliser ?

- Il existe des solutions SaaS comme Docker Hub, Quay...

- Chaque fournisseur cloud majeur propose également son option

  (ACR sur Azure, ECR sur AWS, GCR sur Google Cloud...)

- Il existe également des offres commerciales pour opérer notre propre registry

  (Docker EE, Quay...)

- Ainsi que des solutions open source !

- Lors du choix d'un registry, prêtez attention à son système de build intégré

  (lorsqu'il en propose un)

---

## Builder à la volée

- Conceptuellement, il est envisageable de builder des images à la volée depuis un repository

- Exemple : [ctr.run](https://ctr.run/)

  (déprécié en août 2020, après son rachat par Datadog)

- Cela permettait d'exécuter une commande comme celle-ci :
  ```bash
  docker run ctr.run/github.com/jpetazzo/container.training/dockercoins/hasher
  ```

- Aucune alternative à ce jour

  (une idée de startup libre, avis aux amateurs ?)

???

:FR:- Déployer des images sur notre cluster