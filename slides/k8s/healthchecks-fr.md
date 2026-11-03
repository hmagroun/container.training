# Healthchecks

- Les healthchecks peuvent améliorer la fiabilité de nos applications, par exemple :

  - détecter lorsqu'un conteneur a crashé, et le redémarrer automatiquement

  - suspendre un rolling update jusqu'à ce que les nouveaux conteneurs soient prêts à traiter le trafic

  - retirer temporairement un backend surchargé d'un load balancer

- Il existe trois types de healthchecks, correspondant à différents cas d'usage :

  `startupProbe`, `readinessProbe`, `livenessProbe`

- Les healthchecks sont optionnels

  (en l'absence de healthcheck, Kubernetes considère que le conteneur est sain)

---

## Cas d'usage en résumé

1. *Mon conteneur met beaucoup de temps à démarrer avant de pouvoir servir du trafic.*

    → utiliser une `startupProbe` (mais souvent une `readinessProbe` peut également faire l'affaire¹)

2. *Parfois, mon conteneur est indisponible ou surchargé, et doit par exemple être retiré temporairement de la rotation du load balancer.*

    → utiliser une `readinessProbe`

3. *Parfois, mon conteneur entre dans un état défaillant qui ne peut être résolu que par un redémarrage.*

    → utiliser une `livenessProbe`

.footnote[¹En réalité, nous verrons que dans de nombreux cas, une `readinessProbe` est tout ce dont nous avons besoin. Restez à l'écoute !]

---

## Startup probes

*Mon conteneur met beaucoup de temps à démarrer avant de pouvoir servir du trafic.*

- Après la création d'un conteneur, Kubernetes exécute sa startup probe

- Le conteneur sera considéré comme "non sain" (unhealthy) jusqu'à ce que la sonde réussisse

- Tant que le conteneur est "unhealthy", son Pod... :

  - n'est pas ajouté aux Endpoints des Services

  - n'est pas considéré comme "disponible" dans le cadre d'un rolling update

- Les sondes de readiness et de liveness ne sont activées qu'*après* que la startup probe a signalé un succès

  (s'il n'y a pas de startup probe, les sondes de readiness et de liveness sont activées immédiatement)

---

## Quand utiliser une startup probe

- Pour les conteneurs qui mettent beaucoup de temps à démarrer

  (plus de 30 secondes)

- En particulier si cette durée peut fortement varier

  (par ex. rapide en dev, lente en prod, ou l'inverse)

.footnote[⚠️ Veillez à bien lire les avertissements plus loin dans cette section !]

---

## Readiness probes

*Parfois, mon conteneur "a besoin d'une pause".*

- Vérifier si le conteneur est prêt ou non

- Si le conteneur n'est pas prêt, son Pod n'est pas prêt

- Si le Pod appartient à un Service, il est retiré de ses Endpoints

  (il cesse de recevoir de nouvelles connexions, mais les connexions existantes ne sont pas affectées)

- Si un rolling update est en cours, il peut être mis en pause

  (Kubernetes s'efforcera de respecter le paramètre MaxUnavailable)

- Dès que la readiness probe réussit à nouveau, tout revient à la normale

---

## Quand utiliser une readiness probe

- Pour signaler une défaillance due à une cause externe

  - la base de données est indisponible ou inaccessible

  - un service d'authentification ou un autre service backend indispensable est indisponible

- Pour signaler une défaillance ou une indisponibilité temporaire

  - le runtime est occupé par un garbage collection ou par le (re)chargement de données

  - l'application ne peut traiter que *N* connexions simultanées

  - les nouvelles connexions seront alors dirigées vers d'autres Pods

---

## Liveness probes

*Ce conteneur est mort, nous ne savons pas comment le réparer autrement qu'en le redémarrant.*

- Vérifier si le conteneur est vivant ou mort

- Si Kubernetes détermine que le conteneur est mort :

  - il arrête proprement le conteneur (graceful shutdown)

  - il redémarre le conteneur (sauf si la `restartPolicy` du Pod est `Never`)

- Avec les paramètres par défaut, il faut :

  - jusqu'à 30 secondes pour déterminer que le conteneur est mort

  - jusqu'à 30 secondes pour l'arrêter

---

## Quand utiliser une liveness probe

- Pour détecter des défaillances irrécupérables

  - des deadlocks (entraînant un timeout sur toutes les requêtes)

  - une corruption interne (provoquant des erreurs sur toutes les requêtes)

- Toute situation où la réponse à l'incident consisterait à "simplement le redémarrer / rebooter"

.footnote[⚠️ Veillez à bien lire les avertissements plus loin dans cette section !]

---

## Différents types de sondes

- Kubernetes prend en charge les mécanismes suivants :

  - `httpGet` (requête HTTP GET)

  - `exec` (exécution d'un programme arbitraire)

  - `tcpSocket` (vérification si un port TCP accepte les connexions)

  - `grpc` (standard [GRPC Health Checking Protocol][grpc])

- Toutes les sondes retournent un résultat binaire ("ça fonctionne" ou "ça ne fonctionne pas")

- Voyons les détails spécifiques à chacune d'entre elles !

---

## `httpGet`

- Effectue une requête HTTP GET vers le conteneur

- La requête sera émise par le Kubelet

  (ne nécessite aucun binaire supplémentaire dans l'image du conteneur)

- Le `port` doit être obligatoirement spécifié

- Le chemin (`path`) et des en-têtes supplémentaires (`httpHeaders`) peuvent être spécifiés en option

- Kubernetes utilise le code de statut HTTP de la réponse :

  - 200-399 = succès

  - toute autre valeur = échec

---

## Exemple `httpGet`

La readiness probe suivante vérifie que le conteneur répond sur `/healthz` :

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: frontend
spec:
  containers:
  - name: frontend
    image: myregistry.../frontend:v1.0
    readinessProbe:
      httpGet:
        port: 80
        path: /healthz
```

---

## `exec`

- Exécute un programme arbitraire *à l'intérieur* du conteneur

  (comme avec `kubectl exec` ou `docker exec`)

- Le programme doit être disponible dans l'image du conteneur

- Kubernetes utilise le code de sortie (exit status) du programme

  (convention UNIX standard : 0 = succès, toute autre valeur = échec)

---

## Exemple `exec`

Lorsque le worker est prêt, il doit créer `/tmp/ready`.
<br/>
La sonde suivante lui accorde 5 minutes pour y parvenir.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: queueworker
spec:
  containers:
  - name: worker
    image: myregistry.../worker:v1.0
    startupProbe:
      exec:
        command:
        - test
        - -f
        - /tmp/ready
      failureThreshold: 30
```

---

class: extra-details

## `startupProbe` et `failureThreshold`

- Remarquez la présence de `failureThreshold: 30` dans le manifest précédent

- Cela est crucial lors de la définition d'une `startupProbe`

- Sinon, si le conteneur ne parvient pas à démarrer en moins de 30 secondes...

- ... Kubernetes le redémarre !

- Nous y reviendrons plus en détail un peu plus loin

---

## Utilisation de structures shell

- Si nous souhaitons utiliser des pipes, des conditions, etc., il convient d'invoquer un shell

- Exemple :
  ```yaml
    exec:
      command:
      - sh
      - -c
      - "curl http://localhost:5000/status | jq .ready | grep true"
  ```

- Tous ces programmes (`curl`, `jq`, `grep`) doivent être présents dans l'image du conteneur

---

## `tcpSocket`

- Kubernetes vérifie si le port TCP spécifié accepte les connexions

- Il n'y a pas de vérification supplémentaire

.warning[Il est tout à fait possible qu'un processus soit bloqué ou corrompu, tout en continuant à accepter des connexions TCP !]

---

## `grpc`

<!-- ##VERSION## -->

<!-- https://kubernetes.io/docs/tasks/configure-pod-container/configure-liveness-readiness-startup-probes/#define-a-grpc-liveness-probe -->

- Disponible en bêta depuis Kubernetes 1.24

- S'appuie sur le protocole standard [GRPC Health Checking Protocol][grpc]

---

## Intervalles et seuils (Timing and thresholds)

- Les sondes sont exécutées à intervalles réguliers définis par `periodSeconds` (par défaut : 10)

- Le timeout d'une sonde est configuré avec `timeoutSeconds` (par défaut : 1)

.warning[Si une sonde met plus de temps que cette valeur pour répondre, elle est considérée en ÉCHEC (FAIL)]

.warning[Pour les liveness probes **et les startup probes**, cela entraîne l'arrêt et le redémarrage du conteneur]

- Une sonde est considérée comme réussie après `successThreshold` succès consécutifs (par défaut : 1)

- Une sonde est considérée en échec après `failureThreshold` échecs consécutifs (par défaut : 3)

- Tous ces paramètres peuvent être configurés de manière indépendante pour chaque sonde

---

class: extra-details

## `initialDelaySeconds`

- Une sonde peut comporter un paramètre `initialDelaySeconds` (par défaut : 0)

- Kubernetes attendra cette durée avant d'exécuter la sonde pour la toute première fois

- Il est généralement préférable d'utiliser une `startupProbe` à la place

  (mais ce paramètre existait bien avant l'introduction des startup probes)

---

## Soyez prudents lors de l'ajout de healthchecks

- Il est tentant de vouloir "ajouter systématiquement tous les healthchecks"

- Cela peut s'avérer contre-productif et engendrer des problèmes :

  - pannes en cascade (cascading failures)

  - conteneurs qui échouent à démarrer lorsque le système est sous forte charge

  - gaspillage de ressources lié au redémarrage de conteneurs volumineux

- Analysons ces problématiques !

---

## Pièges des liveness probes

.warning[**N'utilisez pas** de liveness probes pour des problèmes qui ne peuvent pas être résolus par un redémarrage]

- Sinon, nous ne faisons que redémarrer nos pods sans raison, créant une charge inutile

.warning[**Ne dépendez pas** d'autres services au sein d'une liveness probe]

- Sinon, vous risquez de provoquer des pannes en cascade

  (exemple : la liveness probe d'un serveur web qui effectue une requête vers une base de données)

.warning[**Assurez-vous** que les liveness probes répondent rapidement]

- Le timeout par défaut d'une sonde est de 1 seconde (cela peut être ajusté !)

- Si la sonde met plus de temps à répondre, elle finira par déclencher un redémarrage

---

## Pièges des startup probes

- Si une `startupProbe` échoue, Kubernetes redémarre le conteneur associé

- En d'autres termes : avec les paramètres par défaut, le conteneur doit avoir démarré en moins de 30 secondes

  (`failureThreshold` × `periodSeconds`)

- C'est la raison pour laquelle on souhaite presque systématiquement ajuster les paramètres d'une `startupProbe`

  (en particulier son `failureThreshold`)

- Parfois, il est plus simple d'utiliser directement une `readinessProbe` à la place

  (voir la slide suivante pour plus de détails)

---

## Quand avons-nous besoin de startup probes ?

- Uniquement bénéfique pour les conteneurs qui nécessitent un temps de démarrage prolongé

  (plus de 30 secondes)

- S'il n'y a pas de liveness probe, il est plus simple d'utiliser uniquement une readiness probe

  (puisqu'on souhaite probablement disposer d'une readiness probe dans tous les cas)

- En d'autres termes, les startup probes sont utiles dans une situation précise :

  *nous avons une liveness probe, ET le conteneur nécessite beaucoup de temps pour démarrer*

- N'oubliez pas d'augmenter le `failureThreshold`

  (sinon le conteneur échouera à démarrer et sera tué)

---

class: extra-details

## `readinessProbe` vs `startupProbe`

- De nombreux articles de blog / documentations / tutoriels recommandent les readiness probes...

- ... même dans des scénarios où une startup probe semblerait plus appropriée !

- Cela s'explique par le fait que les startup probes sont relativement récentes

  (elles ont atteint le statut GA dans Kubernetes 1.20)

- Lorsqu'il n'y a pas de `livenessProbe`, l'utilisation d'une `readinessProbe` est plus simple :

  - une `startupProbe` nécessite généralement d'adapter le `failureThreshold`

  - une `startupProbe` nécessite généralement aussi une `readinessProbe`

  - une unique `readinessProbe` peut remplir les deux rôles

---

## Bonnes pratiques pour les healthchecks

- Les readiness probes sont presque toujours bénéfiques

  - n'hésitez pas à les ajouter dès le départ !

  - on peut même les rendre *obligatoires*

- Soyez plus mesurés avec les liveness et startup probes

  - elles ne sont pas toujours nécessaires

  - elles peuvent même être nuisibles

---

## Readiness probes

- Presque toujours bénéfiques

- Exceptions :

  - service web ne disposant pas d'une route dédiée "health" ou "ping"

  - ... et où toutes les requêtes sont "coûteuses" (par ex. nombreux appels externes)

---

## Liveness probes

- Si nous ne sommes pas vigilants, nous finissons par redémarrer des conteneurs sans raison

  (ce qui peut générer une surcharge sur le cluster, des pannes en cascade, des pertes de données, etc.)

- Recommandation :

  - n'ajoutez pas de liveness probes immédiatement

  - attendez d'avoir un retour d'expérience en production sur ce code

  - ajoutez ensuite des healthchecks ciblés pour détecter des modes d'échec spécifiques

- Les readiness et liveness probes doivent être différentes

  (vérification différente *ou* timeouts différents *ou* seuils différents)

---

## Récapitulatif des pièges à éviter

- Le timeout par défaut est de 1 seconde

  - si une sonde met plus de 1 seconde à répondre, Kubernetes considère qu'elle a échoué

  - cela peut être modifié en configurant le paramètre `timeoutSeconds`
    <br/>(ou en refactorisant la sonde)

- Les liveness probes ne doivent pas être influencées par l'état de services externes

- Les liveness probes et les readiness probes doivent avoir des paramètres différents

- Pour les startup probes, pensez bien à augmenter le `failureThreshold`

---

## Healthchecks pour les workers

(Dans ce contexte, worker = processus qui n'accepte pas de connexions)

- Une solution relativement simple consiste à utiliser des fichiers

- Pour une startup ou readiness probe :

  - le worker crée `/tmp/ready` lorsqu'il est prêt
  - la sonde vérifie l'existence de `/tmp/ready`

- Pour une liveness probe :

  - le worker met à jour (`touch`) `/tmp/alive` régulièrement
    <br/>(par ex. juste avant de démarrer le traitement d'une tâche)
  - la sonde vérifie que l'horodatage (timestamp) de `/tmp/alive` est récent
  - si l'horodatage est ancien, cela signifie que le worker est bloqué

- Parfois, il peut également être judicieux d'intégrer un serveur web au worker

[grpc]: https://grpc.github.io/grpc/core/md_doc_health-checking.html

???

:EN:- Using healthchecks to improve availability
:FR:- Utiliser des *healthchecks* pour améliorer la disponibilité