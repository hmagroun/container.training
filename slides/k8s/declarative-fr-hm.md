## Declarative vs imperative in Kubernetes

- Avec Kubernetes, nous ne pouvons pas dire : « exécute ce conteneur »

- Tout ce que nous pouvons faire, c'est rédiger une *spec* et la pousser vers l'API server

  (en créant une ressource telle qu'un Pod ou un Deployment)

- L'API server va valider cette spec (et la rejeter si elle est invalide)

- Ensuite, il va la stocker dans etcd

- Un *controller* va « détecter » cette spec et agir en conséquence

---

## Reconciling state

- Prêtez attention aux champs `spec` dans les fichiers YAML plus tard !

- La *spec* décrit *l'état souhaité de la ressource*

- Kubernetes va *réconcilier* l'état actuel avec la spec
  <br/>(techniquement, cela est opéré par un ensemble de *controllers*)

- Lorsque nous souhaitons modifier une ressource, nous mettons à jour la *spec*

- Kubernetes va ensuite faire *converger* cette ressource

???

:FR:- Modèles déclaratifs et impératifs