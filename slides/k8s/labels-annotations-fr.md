# Labels et annotations

- La plupart des ressources Kubernetes peuvent comporter des *labels* et des *annotations*

- Les labels et les annotations sont tous deux des chaînes de caractères arbitraires

  (avec certaines restrictions que nous détaillerons dans un instant)

- Les labels ainsi que les annotations peuvent être ajoutés, supprimés et modifiés dynamiquement

- Cela peut être réalisé avec :

  - la commande `kubectl edit`

  - les commandes `kubectl label` et `kubectl annotate`

  - ... et bien d'autres approches ! (`kubectl apply -f`, `kubectl patch`, ...)

---

## Visualiser les labels et les annotations

- Voyons ce que nous obtenons lors de la création d'un Deployment

.lab[

- Créer un Deployment :
  ```bash
  kubectl create deployment clock --image=jpetazzo/clock
  ```

- Inspecter ses annotations et labels :
  ```bash
  kubectl describe deployment clock
  ```

]

Alors, qu'obtenons-nous ?

---

## Labels et annotations pour notre Deployment

- Nous constatons la présence d'un label :
  ```
  Labels: app=clock
  ```

- Celui-ci est automatiquement injecté par `kubectl create deployment`

- Et d'une annotation :
  ```
  Annotations: deployment.kubernetes.io/revision: 1
  ```

- Elle sert à conserver l'historique des versions successives lors des rolling updates

---

## Qu'en est-il du Pod associé ?

- Retrouvons le Pod généré pour l'inspecter également

.lab[

- Trouver le nom du Pod :
  ```bash
  kubectl get pods
  ```

- Afficher ses détails :
  ```bash
  kubectl describe pod clock-xxxxxxxxxx-yyyyy
  ```

]

Alors, qu'obtenons-nous ?

---

## Labels et annotations pour notre Pod

- Nous observons deux labels :
  ```
    Labels: app=clock
            pod-template-hash=xxxxxxxxxx
  ```

- `app=clock` provient également de la commande `kubectl create deployment`

- `pod-template-hash` a été assigné par le Replica Set

  (lorsque nous effectuerons des rolling updates, chaque groupe de Pods possédera un hash distinct)

- Il n'y a aucune annotation :
  ```
  Annotations: <none>
  ```

---

## Selectors

- Un *selector* est une expression permettant de filtrer selon des labels

- Il restreint la portée d'une commande aux objets correspondant *au moins* à l'ensemble de ces labels

.lab[

- Lister tous les pods ayant au moins `app=clock` :
  ```bash
  kubectl get pods --selector=app=clock
  ```

- Lister tous les pods possédant un label `app`, quelle que soit sa valeur :
  ```bash
  kubectl get pods --selector=app
  ```

]

---

## Définir des labels et des annotations

- La méthode la plus directe consiste à utiliser `kubectl label` et `kubectl annotate`

.lab[

- Définir un label sur le Deployment `clock` :
  ```bash
  kubectl label deployment clock color=blue
  ```

- Vérifier le résultat :
  ```bash
  kubectl describe deployment clock
  ```

]

---

## Autres moyens de visualiser les labels

- `kubectl get` offre plusieurs flags pratiques pour examiner les labels

- `kubectl get --show-labels` affiche l'ensemble des labels

- `kubectl get -L xyz` ajoute une colonne avec la valeur du label `xyz`

.lab[

- Lister tous les labels présents sur nos pods :
  ```bash
  kubectl get pods --show-labels
  ```

- Afficher la valeur du label `app` pour ces pods :
  ```bash
  kubectl get pods -L app
  ```

]

---

class: extra-details

## Précisions sur les selectors

- Si un selector spécifie plusieurs labels, cela signifie "correspondre au minimum à tous ces labels"

  Exemple : `--selector=app=frontend,release=prod`

- `--selector` peut être abrégé en `-l` (pour **l**abels)

  Nous pouvons également utiliser des selectors négatifs (d'exclusion)

  Exemple : `--selector=app!=clock`

- Les selectors sont utilisables avec la grande majorité des commandes `kubectl`

  Exemples : `kubectl delete`, `kubectl label`, ...

---

## Autres manières d'afficher les labels

- Nous pouvons employer le flag `--show-labels` avec `kubectl get`

.lab[

- Afficher les labels pour différents types d'objets :
  ```bash
  kubectl get --show-labels po,rs,deploy,svc,no
  ```

]

---

## Différences entre labels et annotations

- La *clé* (key) des labels comme des annotations :

  - doit commencer et se terminer par une lettre ou un chiffre

  - peut contenir les caractères `.` `-` `_` (mais ni en première ni en dernière position)

  - peut mesurer jusqu'à 63 caractères, ou 253 + `/` + 63

- La *valeur* d'un label est limitée à 63 caractères, avec les mêmes contraintes de syntaxe

- La *valeur* d'une annotation peut comporter n'importe quel caractère arbitraire (y compris du binaire)

- La longueur maximale d'une annotation n'est pas strictement fixée

  (plusieurs dizaines de kilo-octets ne posent pas de problème, plusieurs centaines sont à éviter)

???

:FR:- *Labels* et annotations