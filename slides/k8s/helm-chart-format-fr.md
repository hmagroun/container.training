# Format d'un chart Helm

- Qu'est-ce qu'un chart exactement ?

- De quoi est-il composé ?

- Quelles sont les étapes impliquées dans la création d'un chart ?

  (nous n'allons pas créer de chart de zéro, mais nous verrons les étapes nécessaires)

---

## Qu'est-ce qu'un chart

- Un chart est un ensemble de fichiers

- Certains de ces fichiers sont obligatoires pour que le chart soit valide

  (nous y reviendrons en détail)

- Ces fichiers sont généralement packagés sous forme d'archive tarball

- Ces tarballs sont stockés dans des "repos"

  (qui peuvent être de simples serveurs HTTP statiques)

- Il est possible d'installer un chart depuis un repo, depuis un tarball local ou depuis un répertoire décompressé

  (cette dernière option est privilégiée lors de la phase de développement d'un chart)

---

## Contenu d'un chart

- Un chart doit comporter au minimum :

  - un répertoire `templates/`, contenant les manifests YAML des ressources Kubernetes

  - un fichier `values.yaml`, contenant les paramètres configurables du chart

  - un fichier `Chart.yaml`, contenant les métadonnées (nom, version, description...)

- Examinons un chart simple pour une application de démonstration basique

---

## Ajouter le repo

- Si vous ne l'avez pas encore fait, vous devez ajouter le repo hébergeant ce chart

.lab[

- Ajouter le repo contenant le chart pour OWASP Juice Shop :
  ```bash
  helm repo add juice https://charts.securecodebox.io
  ```

]

---

## Télécharger un chart

- Nous pouvons utiliser la commande `helm pull` pour télécharger un chart depuis un repo

.lab[

- Télécharger l'archive tarball pour `juice/juice-shop` :
  ```bash
  helm pull juice/juice-shop
  ```
  (Cette commande génère un fichier nommé `juice-shop-X.Y.Z.tgz`.)

- Ou télécharger et extraire directement `juice/juice-shop` :
  ```bash
  helm pull juice/juice-shop --untar
  ```
  (Cette commande crée un répertoire nommé `juice-shop`.)

]

---

## Examiner le contenu du chart

- Observons l'arborescence des fichiers et répertoires du chart `juice-shop`

.lab[

- Afficher l'arborescence du chart que nous venons de télécharger :
  ```bash
  tree juice-shop
  ```

]

Nous retrouvons bien les composants mentionnés précédemment : `Chart.yaml`, `templates/`, `values.yaml`.

---

## Templates

- Le répertoire `templates/` contient les manifests YAML des ressources Kubernetes

  (Deployments, Services, etc.)

- Ces manifests peuvent intégrer des balises de template

  (utilisant la bibliothèque standard Go template)

.lab[

- Consulter le fichier de template associé à la ressource Service :
  ```bash
  cat juice-shop/templates/service.yaml
  ```

]

---

## Analyse du fichier de template

- Les balises sont délimitées par `{{ ... }}`

- `{{ template "x.y" }}` permet d'évaluer un [named template](https://helm.sh/docs/chart_template_guide/named_templates/#declaring-and-using-templates-with-define-and-template)

  (défini au préalable avec `{{ define "x.y" }}...contenu...{{ end }}`)

- Le point `.` dans `{{ template "x.y" . }}` représente le *context* passé à ce named template

  (afin que le bloc de template nommé puisse accéder aux variables du contexte local)

- `{{ .Release.xyz }}` fait référence aux [variables intégrées (built-in objects)](https://helm.sh/docs/chart_template_guide/builtin_objects/) initialisées par Helm

  (indiquant le nom du chart, la version, le fait qu'il s'agisse d'une installation ou d'un upgrade...)

- `{{ .Values.xyz }}` fait référence aux [values](https://helm.sh/docs/chart_template_guide/values_files/) configurables

  (nous y revenons dans un instant)

---

## Values

- Chaque chart est fourni avec un [fichier de values](https://helm.sh/docs/chart_template_guide/values_files/)

- Il s'agit d'un fichier YAML contenant un jeu de paramètres par défaut pour le chart

- Les values sont accessibles au sein des templates via la syntaxe `{{ .Values.x.y }}`

  (correspondant au champ `y` dans la map `x` du fichier de values)

- Ces valeurs peuvent être définies ou surchargées lors de l'installation ou de l'upgrade d'un chart :

  - avec l'option `--set x.y=z` (pouvant être répétée pour définir plusieurs valeurs)

  - avec l'option `--values un-fichier.yaml` (pour injecter un ensemble de valeurs depuis un fichier)

- Les charts respectant les bonnes pratiques structurent leurs values selon des conventions éprouvées

  (par exemple via une section `service` permettant de paramétrer `service.type`, etc.)

---

## Autres balises utiles

- `{{ if x }} y {{ end }}` permet d'inclure `y` si l'expression `x` est évaluée à `true`

  (utilisé par exemple pour des healthchecks, des annotations ou une ressource complète)

- `{{ range x }} y {{ end }}` itère sur la collection `x` en évaluant `y` à chaque itération

  (les éléments de `x` sont assignés au contexte courant `.` dans la portée de la boucle)

- `{{- x }}` / `{{ x -}}` supprime les espaces blancs respectivement à gauche ou à droite

- L'ensemble des fonctions de la bibliothèque [Sprig](http://masterminds.github.io/sprig/) est disponible, complété par des fonctions spécifiques :

  `lower`, `upper`, `quote`, `trim`, `default`, `b64enc`, `b64dec`, `sha256sum`, `indent`, `toYaml`...

---

## Pipelines

- `{{ quote blah }}` peut également s'écrire sous forme de pipeline : `{{ blah | quote }}`

- Avec plusieurs arguments, `{{ x y z }}` peut s'écrire `{{ z | x y }}`

- Exemple : `{{ .Values.annotations | toYaml | indent 4 }}`

  - convertit la map sous la clé `annotations` en chaîne YAML

  - applique une indentation de 4 espaces (pour l'aligner avec le contexte environnant)

- Les pipelines ne sont pas spécifiques à Helm, il s'agit d'une fonctionnalité des Go templates

  (consultez la [documentation Go text/template](https://golang.org/pkg/text/template/) pour plus de détails et d'exemples)

---

## README et NOTES.txt

- À la racine du chart, il est fortement recommandé d'inclure un fichier README

- Il sera notamment consultable via la commande `helm show readme juice/juice-shop`

- Dans le répertoire `templates/`, on peut également ajouter un fichier `NOTES.txt`

- Lors de l'installation ou de l'upgrade du template, `NOTES.txt` est également interprété

  (ses balises `{{ ... }}` sont évaluées dynamiquement)

- Ce contenu s'affiche dans le terminal à l'issue de l'installation ou de la mise à jour

- C'est l'emplacement idéal pour fournir des instructions pratiques à l'utilisateur :

  - comment se connecter à la release nouvellement déployée

  - les mots de passe ou autres secrets générés automatiquement

---

## Fichiers complémentaires

- Il est possible de placer des fichiers arbitraires dans le chart (en dehors du répertoire `templates/`)

- Ils sont accessibles dans les templates via l'objet `.Files`

- Ils peuvent être transformés en ConfigMaps ou en Secrets via les fonctions `AsConfig` et `AsSecrets`

  (voir [cet exemple](https://helm.sh/docs/chart_template_guide/accessing_files/#configmap-and-secrets-utility-functions) dans la documentation officielle de Helm)

---

## Hooks et tests

- Nous pouvons déclarer des *hooks* au sein de nos templates

- Les hooks sont des ressources annotées avec `"helm.sh/hook": NOM-DU-HOOK`

- Les hooks disponibles incluent `pre-install`, `post-install`, `test`, [et bien d'autres](https://helm.sh/docs/topics/charts_hooks/#the-available-hooks)

- Les ressources définies dans les hooks sont exécutées à des étapes bien précises du cycle de déploiement

- L'exécution d'un hook est *synchrone*

  (si la ressource est un Job ou un Pod, Helm attend la fin de son exécution)

- Cela s'applique typiquement aux migrations de bases de données, aux sauvegardes, aux notifications ou aux smoke tests

- Les hooks nommés `test` sont exécutés uniquement lors du lancement de `helm test RELEASE-NAME`

???

:FR:- Le format des *Helm charts*