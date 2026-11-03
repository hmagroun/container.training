# Accessing internal services

* Lorsque nous sommes connectés directement sur un node du cluster, nous pouvons accéder aux services internes

  (grâce au modèle réseau de Kubernetes : tous les nodes peuvent joindre l'ensemble des pods et services)

* Lorsque nous accédons à un cluster distant, la situation est différente

  (généralement, notre machine locale n'a pas accès au sous-réseau interne du cluster)

* Comment pouvons-nous accéder temporairement à un service sans l'exposer à tout le monde ?

\--

* `kubectl proxy` : nous donne accès à l'API, ce qui inclut un proxy pour les ressources HTTP

* `kubectl port-forward` : permet de transférer des ports TCP vers des pods, services, ... arbitraires

## Suspension de l'incrédulité

Les exercices et démonstrations de cette section partent du principe que nous avons configuré `kubectl` sur notre
machine locale afin d'accéder à un cluster distant.

Nous allons par conséquent montrer comment accéder aux services et pods du cluster distant
depuis notre machine locale.

Vous pouvez également exécuter ces commandes directement sur le cluster (si vous n'avez pas
installé et configuré `kubectl` localement).

Exécuter ces commandes localement aura moins d'intérêt pratique
(puisque vous pourriez accéder aux services et pods directement),
mais gardez à l'esprit que ces commandes fonctionneront de n'importe où tant que vous avez
installé et configuré `kubectl` pour communiquer avec votre cluster.

## `kubectl proxy` en théorie

* Exécuter `kubectl proxy` nous donne accès à l'intégralité de l'API Kubernetes

* L'API intègre des routes pour relayer le trafic HTTP (proxy)

* Ces routes se présentent sous la forme suivante :

  `/api/v1/namespaces/<namespace>/services/<service>/proxy`

* Il nous suffit d'ajouter l'URI à la fin de la requête, par exemple :

  `/api/v1/namespaces/<namespace>/services/<service>/proxy/index.html`

* Nous pouvons accéder aux `services` et aux `pods` de cette manière

## `kubectl proxy` en pratique

* Accédons au service `webui` à travers `kubectl proxy`

.lab\[

* Lancer un proxy d'API en arrière-plan :

  ```
  kubectl proxy &
  
  ```

* Accéder au service `webui` :

  ```
  curl localhost:8001/api/v1/namespaces/default/services/webui/proxy/index.html
  
  ```

* Arrêter le proxy :

  ```
  kill %1
  
  ```

\]

## `kubectl port-forward` en théorie

* Que faire si nous voulons accéder à un service TCP ?

* Nous pouvons utiliser `kubectl port-forward` à la place

* Cela va créer un relais TCP pour transférer les connexions vers un port spécifique

  (d'un pod, service, deployment...)

* La syntaxe est :

  `kubectl port-forward service/name_of_service local_port:remote_port`

* Si un seul numéro de port est spécifié, il est utilisé à la fois pour le port local et le port distant

## `kubectl port-forward` en pratique

* Accédons à notre serveur Redis distant

.lab\[

* Transférer les connexions du port local 10000 vers le port distant 6379 :

  ```
  kubectl port-forward svc/redis 10000:6379 &
  
  ```

* Se connecter au serveur Redis :

  ```
  telnet localhost 10000
  
  ```

* Exécuter quelques commandes, par ex. `INFO server` puis `QUIT`


* Arrêter le port forwarder :

  ```
  kill %1
  
  ```

\]

???

:EN:- Securely accessing internal services
:FR:- Accès sécurisé aux services internes

:T: Accéder aux services internes depuis notre machine locale

:Q: Quel est l'avantage de "kubectl port-forward" par rapport à un NodePort ?
:A: Il peut transférer des protocoles arbitraires
:A: Il ne nécessite pas d'identifiants d'API Kubernetes
:A: Il offre un load balancing déterministe (au lieu d'aléatoire)
:A: ✔️Il n'expose pas le service publiquement

:Q: Quel est le concept de sécurité derrière "kubectl port-forward" ?
:A: ✔️Nous nous authentifions auprès de l'API Kubernetes, et elle relaie les connexions pour notre compte
:A: Il détecte notre adresse IP source et n'autorise que les connexions en provenance de celle-ci
:A: Il utilise du mTLS (mutual TLS) de bout en bout pour authentifier nos connexions
:A: Il n'y a aucune sécurité (tant qu'il est actif, n'importe qui peut se connecter depuis n'importe où)