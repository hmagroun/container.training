# Notre application d'exemple

- Nous allons cloner le repository GitHub sur notre `node1`

- Le repository contient également des scripts et des outils que nous utiliserons tout au long du workshop

.lab[

<!--
```bash
cd ~
if [ -d container.training ]; then
  mv container.training container.training.$RANDOM
fi
```
-->

- Cloner le repository sur `node1` :
  ```bash
  git clone https://@@GITREPO@@
  ```

]

(Vous pouvez également forker le repository sur GitHub et cloner votre fork si vous préférez.)

---

## Téléchargement et exécution de l'application

Lançons cela avant d'explorer l'environnement, car le téléchargement va prendre un peu de temps...

.lab[

- Aller dans le répertoire `dockercoins`, au sein du repository cloné :
  ```bash
  cd ~/container.training/dockercoins
  ```

- Utiliser Compose pour builder et exécuter tous les conteneurs :
  ```bash
  docker-compose up
  ```

<!--
```longwait units of work done```
-->

]

Compose demande à Docker de builder toutes les images de conteneurs (en pullant
les images de base correspondantes), démarre ensuite tous les conteneurs,
puis affiche les logs agrégés.

---

## En quoi consiste cette application ?

--

- C'est un mineur de DockerCoins ! 💰🐳📦🚢

--

- Non, vous ne pouvez pas acheter de café avec des DockerCoins

--

- Comment fonctionne dockercoins :

  - générer quelques octets aléatoires

  - calculer le hash de ces octets

  - incrémenter un compteur (pour suivre la vitesse)

  - répéter en boucle indéfiniment !

--

- DockerCoin n'est *pas* une cryptomonnaie

  (les seuls points communs sont l'"aléatoire", le "hashing", et "coins" dans le nom)

---

## DockerCoin à l'ère des microservices

- L'application dockercoins est composée de 5 services :

  - `rng` = web service générant des octets aléatoires

  - `hasher` = web service calculant le hash des données reçues en POST

  - `worker` = processus d'arrière-plan appelant `rng` et `hasher`

  - `webui` = interface web pour visualiser la progression

  - `redis` = data store (conserve un compteur mis à jour par le `worker`)

- Ces 5 services sont visibles dans le fichier Compose de l'application,
  [docker-compose.yml](
  https://@@GITREPO@@/blob/master/dockercoins/docker-compose.yml)

---

## Fonctionnement de dockercoins

- Le `worker` appelle le web service `rng` pour générer des octets aléatoires

- Le `worker` appelle le web service `hasher` pour calculer le hash de ces octets

- Le `worker` exécute cela dans une boucle infinie

- Chaque seconde, le `worker` met à jour `redis` pour indiquer le nombre de boucles effectuées

- `webui` interroge `redis`, puis calcule et affiche la "vitesse de hashing" dans notre navigateur

*(Voir le schéma sur la diapositive suivante !)*

---

class: pic

![Diagramme montrant les 5 conteneurs de l'application](images/dockercoins-diagram.png)

---

## Service discovery dans le monde des conteneurs

Comment chaque service découvre-t-il l'adresse des autres ?

--

- Nous ne codons pas d'adresses IP en dur dans le code

- Nous ne codons pas non plus de FQDN en dur dans le code

- Nous nous connectons simplement à un nom de service, et la magie des conteneurs opère

  (Et par "magie des conteneurs", nous entendons "un serveur DNS embarqué, dynamique et astucieux")

---

## Exemple dans `worker/worker.py`

```python
redis = Redis("`redis`")


def get_random_bytes():
    r = requests.get("http://`rng`/32")
    return r.content


def hash_bytes(data):
    r = requests.post("http://`hasher`/",
                      data=data,
                      headers={"Content-Type": "application/octet-stream"})
```

(Code source complet disponible [ici](
https://@@GITREPO@@/blob/8279a3bce9398f7c1a53bdd95187c53eda4e6435/dockercoins/worker/worker.py#L17
))

---

class: extra-details

## Liens, nommage et service discovery

- Les conteneurs peuvent avoir des alias réseau (résolubles via DNS)

- Le format de fichier Compose version 2+ rend chaque conteneur accessible via son nom de service

- Le format Compose version 1 nécessitait des sections "links" pour y parvenir

- Les alias réseau sont automatiquement isolés par namespace

  - vous pouvez avoir plusieurs applications qui déclarent et utilisent un service nommé `database`

  - les conteneurs de l'application bleue résoudront `database` vers l'IP de la base de données bleue

  - les conteneurs de l'application verte résoudront `database` vers l'IP de la base de données verte

---

## Montrez-moi le code !

- Vous pouvez consulter le repository GitHub contenant l'ensemble des supports de ce workshop :
  <br/>https://@@GITREPO@@

- L'application se trouve dans le sous-répertoire [dockercoins](
  https://@@GITREPO@@/tree/master/dockercoins)

- Le fichier Compose ([docker-compose.yml](
  https://@@GITREPO@@/blob/master/dockercoins/docker-compose.yml))
  liste l'ensemble des 5 services

- `redis` utilise une image officielle issue de Docker Hub

- `hasher`, `rng`, `worker` et `webui` sont chacun construits à partir d'un Dockerfile

- Le Dockerfile et le code source de chaque service se trouvent dans leur propre répertoire

  (`hasher` est dans le répertoire [hasher](https://@@GITREPO@@/blob/master/dockercoins/hasher/),
  `rng` est dans le répertoire [rng](https://@@GITREPO@@/blob/master/dockercoins/rng/), etc.)

---

class: extra-details

## Version du format de fichier Compose

*Ceci n'est pertinent que si vous avez utilisé Compose avant 2016...*

- Compose 1.6 a introduit la prise en charge d'un nouveau format de fichier Compose (dit "v2")

- Les services ne sont plus déclarés au niveau racine, mais sous une section `services`

- Une clé `version` doit obligatoirement figurer au niveau racine, avec la valeur `"2"` (sous forme de chaîne de caractères, pas un entier)

- Les conteneurs sont placés sur un réseau dédié, rendant les links inutiles

- Il existe d'autres différences mineures, mais la mise à niveau est simple et directe

---

## Notre application en action

- Sur le côté gauche, la "bande multicolore" affiche les noms des conteneurs

- Sur le côté droit, nous visualisons la sortie de nos conteneurs

- Nous pouvons voir le service `worker` émettre des requêtes vers `rng` et `hasher`

- Pour `rng` et `hasher`, nous observons les logs d'accès HTTP

---

## Connexion à l'interface web (Web UI)

- "Les logs, c'est passionnant et tellement amusant !" (Personne, jamais)

- Le conteneur `webui` expose un dashboard web ; allons le consulter

.lab[

- Depuis un navigateur web, connectez-vous à `node1` sur le port 8000

- Rappel : les alias `nodeX` ne sont valides que sur les nodes eux-mêmes

- Dans votre navigateur, vous devez saisir l'adresse IP de votre node

<!-- ```open http://node1:8000``` -->

]

Une zone d'affichage doit apparaître, et après quelques secondes, un
graphique bleu va se tracer.

---

class: self-paced, extra-details

## Si le graphique ne se charge pas

Si vous observez simplement une erreur `Page not found`, cela peut signifier que votre
Docker Engine s'exécute sur une machine distincte. Cela peut être le cas si :

- vous utilisez Docker Toolbox

- vous utilisez une VM (locale ou distante) créée avec Docker Machine

- vous contrôlez un Docker Engine distant

Lorsque vous exécutez DockerCoins en mode développement, les fichiers statiques de la web UI
sont montés dans le conteneur via un volume. Hélas, les volumes ne fonctionnent
que dans un environnement local, ou lors de l'utilisation de Docker Desktop pour Mac ou Windows.

Comment corriger cela ?

Arrêtez l'application avec `^C`, éditez `dockercoins.yml`, commentez la section `volumes`, puis réessayez.

---

class: extra-details

## Pourquoi la vitesse semble-t-elle irrégulière ?

- Il *semble* que la vitesse tourne autour de 4 hashes/seconde

- Ou plus exactement : 4 hashes/seconde, avec des chutes régulières à zéro

- Pourquoi ?

--

class: extra-details

- L'application fonctionne en réalité à une vitesse constante et stable : 3,33 hashes/seconde
  <br/>
  (ce qui correspond à 1 hash toutes les 0,3 secondes, pour certaines *raisons*)

- Oui, et alors ?

---

class: extra-details

## Pourquoi ce graphique n'est-il *pas idéal* ?

- Le worker ne met pas à jour le compteur à chaque boucle, mais au maximum une fois par seconde

- La vitesse est calculée par le navigateur, qui interroge le compteur environ une fois par seconde

- Entre deux rafraîchissements consécutifs, le compteur s'incrémente soit de 4, soit de 0

- La vitesse perçue oscillera donc selon la séquence 4 - 4 - 4 - 0 - 4 - 4 - 0 etc.

- Que pouvons-nous en déduire ?

--

class: extra-details

- "Je suis manifestement incapable d'écrire du bon code frontend !" 😀 — Jérôme

---

## Arrêter l'application

- Si nous interrompons Compose (avec `^C`), il demandera poliment au Docker Engine d'arrêter l'application

- Le Docker Engine enverra un signal `TERM` aux conteneurs

- Si les conteneurs ne s'arrêtent pas dans les délais impartis, le moteur enverra un signal `KILL`

.lab[

- Arrêter l'application en faisant `^C`

<!--
```key ^C```
-->

]

--

Certains conteneurs se ferment immédiatement, d'autres prennent plus de temps.

Les conteneurs qui ne gèrent pas le signal `SIGTERM` finissent par être killés après un timeout de 10s. Si nous sommes très impatients, nous pouvons faire `^C` une deuxième fois !