# Introduction au YAML

- YAML Ain't Markup Language (selon [yaml.org][yaml])

- *Quasiment* indispensable lorsque l'on travaille avec des conteneurs :

  - Fichiers Docker Compose

  - Manifests Kubernetes

  - De nombreux pipelines CI (GitHub, GitLab...)

- Si vous ne connaissez pas bien le YAML, ce module est fait pour vous !

[yaml]: https://yaml.org/

---

## Qu'est-ce que c'est ?

- Un langage de représentation de données

```yaml
- country: France
  capital: Paris
  code: fr
  population: 68042591
- country: Germany
  capital: Berlin
  code: de
  population: 84270625
- country: Norway
  capital: Oslo
  code: no # It's a trap!
  population: 5425270
```

- Même sans connaître le YAML, on est probablement capable d'ajouter un pays à ce fichier :)

---

## Tester le YAML

- Méthode 1 : dans le navigateur

  https://onlineyamltools.com/convert-yaml-to-json

  https://onlineyamltools.com/highlight-yaml

- Méthode 2 : dans un shell

  ```bash
  yq . foo.yaml
  ```

- Méthode 3 : en Python

  ```python
    import yaml; yaml.safe_load("""
    - country: France
      capital: Paris
    """)
  ```

---

## Les bases

- Chaînes de caractères (strings), nombres, booléens, `null`

- Séquences (= tableaux, listes)

- Mappings (= objets, dictionnaires)

- C'est un superset de JSON

  (si vous connaissez le JSON, vous pouvez simplement écrire du JSON)

- Les commentaires commencent par `#`

- Un unique *fichier* peut contenir plusieurs *documents*

  (séparés par `---` sur une seule ligne)

---

## Séquences

- Exemple : séquence de chaînes de caractères
  ```yaml
  [ "france", "germany", "norway" ]
  ```

- Exemple : la même séquence, sans les guillemets
  ```yaml
  [ france, germany, norway ]
  ```

- Exemple : la même séquence, en "block collection style" (= multi-lignes)
  ```yaml
  - france
  - germany
  - norway
  ```

---

## Mappings

- Exemple : mapper des chaînes à des nombres
  ```yaml
  { "france": 68042591, "germany": 84270625, "norway": 5425270 }
  ```

- Exemple : le même mapping, sans les guillemets
  ```yaml
  { france: 68042591, germany: 84270625, norway: 5425270 }
  ```

- Exemple : le même mapping, en "block collection style"
  ```yaml
  france: 68042591
  germany: 84270625
  norway: 5425270
  ```

---

## Combiner les types

- Dans une séquence (ou un mapping), nous pouvons mélanger différents types

  (y compris d'autres séquences ou mappings)

- Exemple :
  ```yaml
  questions: [ name, quest, favorite color ]
  answers: [ "Arthur, King of the Britons", Holy Grail, purple, 42 ]
  ```

- Remarquez qu'il est nécessaire de mettre "Arthur" entre guillemets à cause de la virgule

- Remarquez aussi que nous n'avons pas le même nombre d'éléments dans questions et answers

---

## Combinaisons supplémentaires

- Exemple :
  ```yaml
    - service: nginx
      ports: [ 80, 443 ]
    - service: bind
      ports: [ 53/tcp, 53/udp ]
    - service: ssh
      ports: 22
  ```

- Notez que `ports` n'a pas toujours le même type

  (le code traitant ces données devra probablement être bien pensé !)

---

## ⚠️ Booléens automatiques

```yaml
codes:
  france: fr
  germany: de
  norway: no
```

--

```json
{
  "codes": {
    "france": "fr",
    "germany": "de",
    "norway": false
  }
}
```

---

## ⚠️ Booléens automatiques

- `no` peut devenir `false`

  (cela dépend du parseur YAML utilisé)

- Il convient donc de l'encadrer de guillemets :

  ```yaml
    codes:
      france: fr
      germany: de
      norway: "no"
  ```

---

## ⚠️ Flottants automatiques

```yaml
version:
  libfoo: 1.10
  fooctl: 1.0
```

--

```json
{
  "version": {
    "libfoo": 1.1,
    "fooctl": 1
  }
}
```

---

## ⚠️ Flottants automatiques

- Les zéros finaux disparaissent

- Ceux-ci doivent également être mis entre guillemets :

  ```yaml
    version:
      libfoo: "1.10"
      fooctl: "1.0"
  ```

---

## ⚠️ Horaires automatiques

```yaml
portmap:
- 80:80
- 22:22
```

--

```json
{
  "portmap": [
    "80:80",
    1342
  ]
}
```

---

## ⚠️ Horaires automatiques

- `22:22` devient `1342`

- Cela représente 22 minutes et 22 secondes = 1342 secondes

- Là encore, il faut utiliser des guillemets

---

## Séparateur de documents

- Un unique *fichier* YAML peut comporter plusieurs *documents* séparés par `---` :

  ```yaml
    This is a document consisting of a single string.
    --- 💡
    name: The second document
    type: This one is a mapping (key→value)
    --- 💡
    - Third document
    - This one is a sequence
  ```

- Certains aiment ajouter un `---` supplémentaire au début et/ou à la fin

  (ce n'est pas obligatoire, mais cela facilite par ex. la concaténation de plusieurs fichiers avec `cat`)

.footnote[💡 Ignorez ceci ; c'est une astuce pour contourner [cette issue][remarkyaml].]

[remarkyaml]: https://github.com/gnab/remark/issues/679

---

## Chaînes multi-lignes

Testez le bloc suivant dans un parseur YAML :

```yaml
add line breaks: "in double quoted strings\n(like this)"
preserve line break: |
  by using
  a pipe (|)
  (this is great for embedding shell scripts, configuration files...)
do not preserve line breaks: >
  by using
  a greater-than (>)
  (this is great for embedding very long lines)
```

Consultez https://yaml-multiline.info/ pour des astuces avancées sur les chaînes multi-lignes !

(Par ex. pour supprimer ou conserver les sauts de ligne `\n` supplémentaires à la fin du bloc.)

---

class: extra-details

## Fonctionnalités avancées

Les ancres (anchors) permettent de "mémoriser" et de réutiliser du contenu :

```yaml
debian: &debian
  packages: deb
  latest-stable: bullseye

also-debian: *debian

ubuntu:
  <<: *debian
  latest-stable: jammy
```

---

class: extra-details

## Le YAML, bon ou mauvais ?

- Évolution naturelle du XML vers le JSON puis vers le YAML

- Il existe d'autres langages de données

  (par ex. HCL, DSL conçus avec Ruby, CUE...)

- Des compromis sont nécessaires, par exemple :

  - plus convivial à lire → plus de "magie" avec des effets de bord

  - plus puissant → courbe d'apprentissage plus raide

- Qu'on l'adore ou qu'on le déteste, c'est une excellente idée d'en comprendre les rouages !

- Outil intéressant si vous appréciez YAML : https://carvel.dev/ytt/

???

:FR:- Comprendre le YAML et ses subtilités