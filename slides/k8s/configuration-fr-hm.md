# Managing configuration

* Certaines applications ont besoin d'être configurées (évidemment !)

* Il existe de multiples manières pour notre code de récupérer sa configuration :

  * arguments en ligne de commande

  * variables d'environnement

  * fichiers de configuration

  * serveurs de configuration (récupération de la configuration depuis une base de données, une API...)

  * ... et bien d'autres (les développeurs savent être très créatifs !)

* Comment mettre en œuvre ces approches avec les conteneurs et Kubernetes ?

---

## Transmettre la configuration aux conteneurs

* Il existe de nombreuses façons de transmettre de la configuration au code exécuté dans un conteneur :

  * l'intégrer directement dans une image personnalisée (baking)

  * arguments en ligne de commande

  * variables d'environnement

  * injection de fichiers de configuration

  * exposition via l'API Kubernetes

  * serveurs de configuration

* Passons en revue ces différentes stratégies !

---

## Intégrer la configuration dans des images sur mesure

* Placer la configuration directement dans l'image

  (cela peut être un fichier de configuration, mais aussi des directives `ENV` ou `CMD`)

* C'est facile ! C'est simple !

* Malheureusement, cela présente aussi des inconvénients :

  * multiplication des images

  * images différentes entre dev, staging, prod...

  * la moindre reconfiguration nécessite un cycle complet build/push/pull

* Évitez cette approche à moins de ne pas avoir le temps d'explorer d'autres options

---

## Arguments en ligne de commande

* Indiquer ce qui doit s'exécuter dans le conteneur

* Transmettre `command` et/ou `args` dans les options du conteneur au sein du template du Pod

* `command` et `args` sont tous deux des tableaux

* Exemple ([source](https://github.com/jpetazzo/container.training/blob/main/k8s/consul-1.yaml#L70)) :
  ```yaml
    args:
    - "agent"
    - "-bootstrap-expect=3"
    - "-retry-join=provider=k8s label_selector=\"app=consul\" namespace=\"$(NS)\""
    - "-client=0.0.0.0"
    - "-data-dir=/consul/data"
    - "-server"
    - "-ui"
  ```

---

## `args` ou `command` ?

* Utilisez `command` pour surcharger l'`ENTRYPOINT` défini dans l'image

* Utilisez `args` pour conserver l'`ENTRYPOINT` défini dans l'image

  (les paramètres spécifiés dans `args` sont ajoutés à l'`ENTRYPOINT`)

* En cas de doute, utilisez `command`

* Il est également possible d'utiliser *à la fois* `command` et `args`

  (ils seront concaténés, exactement comme `ENTRYPOINT` et `CMD`)

* Consultez la [documentation](https://kubernetes.io/docs/tasks/inject-data-application/define-command-argument-container/#notes) pour observer leurs interactions

---

## Arguments en ligne de commande : avantages et inconvénients

* Très efficace lorsque les options sont passées directement au binaire exécuté

  (sinon, un script wrapper peut contourner le problème)

* Très efficace lorsqu'il n'y a pas trop de paramètres

  (pour éviter un tableau `args` de 20 lignes)

* Nécessite de la documentation et/ou une bonne compréhension de l'application sous-jacente

  (« de quels paramètres et flags ai-je besoin, déjà ? »)

* Bien adapté aux paramètres obligatoires (sans valeurs par défaut)

* Pas idéal lorsqu'il faut de toute façon fournir un véritable fichier de configuration

---

## Variables d'environnement

* Transmettre les options via le bloc `env` dans la spécification du conteneur

* Exemple :
  ```yaml
      env:
      - name: ADMIN_PORT
        value: "8080"
      - name: ADMIN_AUTH
        value: Basic
      - name: ADMIN_CRED
        value: "admin:0pensesame!"
  ```

.warning[`value` doit être une chaîne de caractères ! Assurez-vous que les nombres et les chaînes complexes soient entre guillemets.]

🤔 Pourquoi cette structure verbeuse `{name: xxx, value: yyy}` ? La réponse arrive très vite !

---

## La downward API

* Dans l'exemple précédent, les variables d'environnement ont des valeurs fixes

* Nous pouvons également exploiter un mécanisme appelé la *downward API*

* La downward API permet d'exposer des informations relatives au Pod ou au conteneur

  * soit à travers des fichiers spéciaux (nous ne l'aborderons pas pour le moment)

  * soit au moyen de variables d'environnement

* La valeur de ces variables d'environnement est calculée au moment du démarrage du conteneur

* Rappel : les variables d'environnement ne changeront pas (et ne peuvent pas changer) après le lancement du conteneur

* Voyons quelques exemples concrets !

---

## Exposer le namespace du pod

```yaml
    - name: MY_POD_NAMESPACE
      valueFrom:
        fieldRef:
          fieldPath: metadata.namespace
```

* Pratique pour construire le FQDN des services

  (dans certains contextes, un nom court ne suffit pas)

* Par exemple, les deux commandes suivantes devraient être équivalentes :
  ```
  curl api-backend
  curl api-backend.$MY_POD_NAMESPACE.svc.cluster.local
  ```

---

## Exposer l'adresse IP du pod

```yaml
    - name: MY_POD_IP
      valueFrom:
        fieldRef:
          fieldPath: status.podIP
```

* Pratique si nous devons connaître notre propre adresse IP

  (nous pourrions aussi la lire depuis `eth0`, mais cette méthode est plus fiable)

---

## Exposer les limites de ressources du conteneur

```yaml
    - name: MY_MEM_LIMIT
      valueFrom:
        resourceFieldRef:
          containerName: test-container
          resource: limits.memory
```

* Très utile pour les runtimes où la mémoire est gérée par un garbage collector

* Exemple : la JVM

  (la mémoire allouée à la JVM doit être configurée avec le flag `-Xmx`)

* Bonne pratique : définir une limite de mémoire et la transmettre au runtime

* Note : les versions récentes de la JVM peuvent le faire automatiquement

  (voir [JDK-8146115](https://bugs.java.com/bugdatabase/view_bug.do?bug_id=JDK-8146115))
  et
  [cet article de blog](https://very-serio.us/2017/12/05/running-jvms-in-kubernetes/)
  pour des exemples détaillés)

---

## En savoir plus sur la downward API

* [Cette page de documentation](https://kubernetes.io/docs/tasks/inject-data-application/environment-variable-expose-pod-information/) détaille ces variables d'environnement

* Et [celle-ci](https://kubernetes.io/docs/tasks/inject-data-application/downward-api-volume-expose-pod-information/) explique l'autre méthode d'utilisation de la downward API

  (via des fichiers injectés dans le système de fichiers du conteneur)

* Ce second lien liste également l'ensemble des champs exploitables avec la downward API

---

## Variables d'environnement : avantages et inconvénients

* Idéal lorsque le programme exécuté attend nativement ces variables

* Idéal pour les paramètres optionnels avec des valeurs par défaut pertinentes

  (puisque l'image du conteneur peut fournir ces valeurs par défaut)

* En quelque sorte auto-documenté

  (on peut voir quelles variables d'environnement sont définies dans l'image, et leurs valeurs)

* Peut être détourné avec des valeurs très volumineuses...

* ... Vous *pouvez* mettre un fichier de configuration Tomcat complet dans une variable...

* ... Mais *devriez-vous* le faire ?

(Faites-le si vous n'avez pas d'autre choix, nous ne jugeons pas ! Mais nous allons voir de meilleures approches.)

---

## Injecter des fichiers de configuration

* Parfois, il n'y a pas d'alternative : nous devons injecter un fichier de configuration complet

* Kubernetes met à disposition un mécanisme dédié : les `configmaps`

* Un ConfigMap est une ressource Kubernetes qui réside au sein d'un namespace

* Conceptuellement, il s'agit d'une table clé/valeur

  (les valeurs sont des chaînes de caractères arbitraires)

* On peut les envisager sous (au moins) deux angles :

  * comme contenant un ou plusieurs fichiers de configuration entiers

  * comme contenant des paramètres de configuration individuels

*Note : pour stocker des données sensibles, nous utilisons les "Secrets", un
autre type de ressource au comportement très proche des configmaps.
Nous les aborderons juste après !*

---

## Configmaps stockant des fichiers complets

* Dans ce cas, chaque paire clé/valeur correspond à un fichier de configuration

* Clé = nom du fichier

* Valeur = contenu du fichier

* Il peut y avoir une seule paire clé/valeur, ou autant que nécessaire

  (pour des applications complexes nécessitant plusieurs fichiers de configuration)

* Exemples :
  ```
  # Créer un configmap avec une seule clé, "app.conf"
  kubectl create configmap my-app-config --from-file=app.conf
  # Créer un configmap avec une seule clé, "app.conf", mais un fichier source différent
  kubectl create configmap my-app-config --from-file=app.conf=app-prod.conf
  # Créer un configmap avec plusieurs clés (une par fichier dans le répertoire config.d)
  kubectl create configmap my-app-config --from-file=config.d/
  ```

---

## Configmaps stockant des paramètres individuels

* Dans ce cas, chaque paire clé/valeur correspond à un paramètre

* Clé = nom du paramètre

* Valeur = valeur du paramètre

* Exemples :
  ```
  # Créer un configmap avec deux clés
  kubectl create cm my-app-config \
      --from-literal=foreground=red \
      --from-literal=background=blue
  
  # Créer un configmap depuis un fichier contenant des paires clé=valeur
  kubectl create cm my-app-config \
      --from-env-file=app.conf
  ```

---

## Exposer des configmaps aux conteneurs

* Les ConfigMaps peuvent être exposés sous forme de fichiers dans le filesystem d'un conteneur

  * cela se fait en déclarant un volume et en le montant dans le conteneur

  * cette méthode est particulièrement adaptée aux configmaps stockant des fichiers entiers

* Les ConfigMaps peuvent être exposés sous forme de variables d'environnement dans le conteneur

  * cela se fait grâce à la downward API

  * cette méthode est particulièrement adaptée aux configmaps contenant des paramètres individuels

* Voyons comment mettre en œuvre les deux méthodes !

---

## Exemple : configuration de HAProxy

* Nous allons déployer HAProxy, un load balancer très populaire

* Il s'attend à trouver sa configuration à un emplacement précis :

  `/usr/local/etc/haproxy/haproxy.cfg`

* Nous allons créer un ConfigMap contenant le fichier de configuration

* Puis nous monterons ce ConfigMap dans un Pod exécutant HAProxy

---

## Blue/green load balancing

* Dans cet exemple, nous allons déployer deux versions de notre application :

  * la version "blue" dans le namespace `blue`

  * la version "green" dans le namespace `green`

* Dans chacun des deux namespaces, nous aurons un Deployment et un Service

  (tous deux nommés `color`)

* Nous voulons équilibrer la charge du trafic (load balance) entre les deux namespaces

  (nous ne pouvons pas le faire avec un simple service selector : ils ne traversent pas les namespaces)

---

## Déployer l'application

* Nous allons utiliser l'image `jpetazzo/color`

  (il s'agit d'un serveur "HTTP echo" simple qui indique quel pod a traité la requête)

* Nous pouvons créer chaque Namespace, Deployment et Service manuellement, ou bien...

.lab[

* Nous pouvons déployer l'application à l'aide d'un manifeste YAML :
  ```bash
  kubectl apply -f ~/container.training/k8s/rainbow.yaml
  ```

]

---

## Tester l'application

* Rappel : le Service `x` dans le Namespace `y` est accessible via :

  `x.y`, `x.y.svc`, `x.y.svc.cluster.local`

* Comme le suffixe `cluster.local` peut varier, nous utiliserons `x.y.svc`

.lab[

* Vérifier que l'application est opérationnelle :
  ```bash
    kubectl run --rm -it --restart=Never --image=nixery.dev/curl my-test-pod \
            curl color.blue.svc
  ```

]

---

## Créer la configuration de HAProxy

Voici le fichier que nous allons utiliser, @@LINK[k8s/haproxy.cfg] :

```
@@INCLUDE[k8s/haproxy.cfg]
```

---

## Créer le ConfigMap

.lab[

* Créer un ConfigMap nommé `haproxy` contenant le fichier de configuration :
  ```bash
  kubectl create configmap haproxy --from-file=~/container.training/k8s/haproxy.cfg
  ```

* Examiner la structure de notre configmap :
  ```bash
  kubectl get configmap haproxy -o yaml
  ```

]

---

## Utiliser le ConfigMap

Voici @@LINK[k8s/haproxy.yaml], un manifeste de Pod utilisant ce ConfigMap :

```yaml
@@INCLUDE[k8s/haproxy.yaml]
```

---

## Créer le Pod

.lab[

* Créer le Pod HAProxy :
  ```bash
  kubectl apply -f ~/container.training/k8s/haproxy.yaml
  ```

<!-- ```hide kubectl wait pod haproxy --for condition=ready``` -->

* Vérifier l'adresse IP allouée au pod :
  ```bash
  kubectl get pod haproxy -o wide
  IP=$(kubectl get pod haproxy -o json | jq -r .status.podIP)
  ```

]

---

## Tester notre load balancer

* Si tout s'est déroulé correctement, nous devrions observer une alternance round robin parfaite

  (une requête vers `blue`, une requête vers `green`, une requête vers `blue`, etc.)

.lab[

* Envoyer quelques requêtes :
  ```bash
  for i in $(seq 10); do
  curl $IP
  done
  ```

]

---

## Exposer des configmaps avec la downward API

* Nous allons exécuter un registre Docker sur un port personnalisé

* Par défaut, le registry écoute sur le port 5000

* Cela peut être modifié en définissant la variable d'environnement `REGISTRY_HTTP_ADDR`

* Nous allons stocker le numéro de port dans un configmap

* Puis nous exposerons ce configmap en tant que variable d'environnement du conteneur

---

## Créer le configmap

.lab[

* Notre configmap comportera une unique clé, `http.addr` :
  ```bash
  kubectl create configmap registry --from-literal=http.addr=0.0.0.0:80
  ```

* Vérifier notre configmap :
  ```bash
  kubectl get configmap registry -o yaml
  ```

]

---

## Utiliser le configmap

Nous allons utiliser la définition de pod suivante :

```yaml
@@INCLUDE[k8s/registry.yaml]
```

---

## Utiliser le configmap

* La définition de la ressource issue de la slide précédente se trouve dans @@LINK[k8s/registry.yaml]

.lab[

* Créer le pod registry :
  ```bash
  kubectl apply -f ~/container.training/k8s/registry.yaml
  ```

<!-- ```hide kubectl wait pod registry --for condition=ready``` -->

* Vérifier l'adresse IP allouée au pod :
  ```bash
  kubectl get pod registry -o wide
  IP=$(kubectl get pod registry -o json | jq -r .status.podIP)
  ```

* Confirmer que le registry est bien accessible sur le port 80 :
  ```bash
  curl $IP/v2/_catalog
  ```

]

???

:FR:- Gérer la configuration des applications
:FR:- Configuration au travers de la *downward API*
:FR:- Configurer les applications avec des *Config Maps*