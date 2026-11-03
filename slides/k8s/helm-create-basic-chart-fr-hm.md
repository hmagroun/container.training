# Créer un chart basique

- Nous allons voir comment concevoir un chart *très simplifié*

- Dans un chart réel en production, de *nombreux éléments* seraient rendus dynamiques via des templates

  (Noms des ressources, types de Service, nombre de replicas...)

.lab[

- Créer un chart d'exemple :
  ```bash
  helm create dockercoins
  ```

- Déplacer les templates générés par défaut et créer un répertoire `templates` vide :
  ```bash
  mv dockercoins/templates dockercoins/default-templates
  mkdir dockercoins/templates
  ```

]

---

## Ajouter les manifests de notre application

- Le repository fournit un fichier `dockercoins.yaml` prêt à l'emploi

.lab[

- Copier le fichier YAML dans le sous-répertoire `templates` du chart :
  ```bash
  cp ~/container.training/k8s/dockercoins.yaml dockercoins/templates
  ```

]

- Note : il est généralement préférable de découper les manifests en plusieurs fichiers YAML distincts

---

## Tester notre chart Helm

- Notre chart Helm est désormais prêt

.lab[

- Installons le chart :
  ```bash
  helm install helmcoins dockercoins
  ```
  (`helmcoins` est le nom de la release ; `dockercoins` est le chemin local vers le chart)

]

--

- Si l'application est déjà déployée dans le cluster, la commande échouera :
```
Error: rendered manifests contain a resource that already exists.
Unable to continue with install: existing resource conflict:
kind: Service, namespace: default, name: hasher
```

---

## Basculer vers un autre namespace

- Si une instance de `dockercoins` est déjà active dans le namespace courant :

  - nous pouvons changer de namespace avec `kubens` ou `kubectl config set-context`

  - nous pouvons également demander explicitement à Helm de cibler un namespace différent

.lab[

- Créer un nouveau namespace :
  ```bash
  kubectl create namespace helmcoins
  ```

- Déployer notre chart dans ce namespace :
  ```bash
  helm install helmcoins dockercoins --namespace=helmcoins
  ```

]

---

## Les releases Helm sont isolées par namespace

- Tentons de visualiser la release que nous venons de déployer

.lab[

- Lister les releases Helm :
  ```bash
  helm list
  ```

]

Notre release n'apparaît pas !

Il est impératif de spécifier son namespace (ou de basculer le contexte sur ce namespace).

---

## Spécifier le namespace

- Réessayons en indiquant le namespace approprié

.lab[

- Lister les releases Helm dans le namespace `helmcoins` :
  ```bash
  helm list --namespace=helmcoins
  ```

]

---

## Vérifier notre nouvelle instance de DockerCoins

- Nous pouvons inspecter les logs du worker ou accéder à l'interface web (web UI)

.lab[

- Récupérer le numéro de port NodePort du Service `webui` :
  ```bash
  kubectl get service webui --namespace=helmcoins
  ```

- Ouvrir cette URL dans un navigateur web

- Consulter les logs du worker :
  ```bash
  kubectl logs deploy/worker --tail=10 --follow --namespace=helmcoins
  ```

]

Note : le démarrage du worker peut nécessiter une à deux minutes.

---

## Analyse critique et limitations

- Les bonnes pratiques Helm (et Kubernetes) préconisent d'ajouter un ensemble d'annotations et de labels standard  (par exemple `app.kubernetes.io/name`, `helm.sh/chart`, `app.kubernetes.io/instance`...)

- Notre chart basique ne définit aucun de ces métadonnées

- Il n'exploite aucune balise de templating

- Est-il pertinent d'utiliser Helm dans ce cas précis ?

  - *Oui,* car Helm assure tout de même :

    - le suivi exhaustif des ressources créées par le chart

    - l'historique des révisions successives, rendant possible le rollback


Consultez la [documentation Helm][helm-labels] et la [documentation Kubernetes][k8s-labels]
pour plus de détails sur les labels et annotations recommandés.

[helm-labels]: https://helm.sh/docs/chart_best_practices/labels/
[k8s-labels]: https://kubernetes.io/docs/concepts/overview/working-with-objects/common-labels/

---

## Nettoyage

- Supprimons ce déploiement avant de poursuivre

.lab[

- Supprimer la release (ne pas oublier de spécifier le namespace) :
  ```bash
  helm delete helmcoins --namespace=helmcoins
  ```

]

---

## Bonnes pratiques pour l'écriture de charts

- Il n'est pas obligatoire d'exécuter `helm install` ou `helm upgrade` pour valider un chart

- Si nous souhaitons uniquement inspecter le YAML généré par le moteur de rendu, utilisons `helm template` :
  ```bash
  helm template ./my-chart
  helm template release-name ./my-chart
  ```

- Les options `--set` et `--values` sont évidemment utilisables ici

- Attention : cela ne valide pas entièrement la conformité Kubernetes des ressources !

  (par exemple, une entrée telle que `apiVersion: blabla` ne provoquera pas d'erreur de rendu)

- C'est un outil très pratique pour tester rapidement le rendu local

---

## Explorer le moteur de templating

Essayez d'ajouter le bloc suivant dans un fichier sous le répertoire `templates` :

```yaml
hello: {{ .Values.service.port }}
comment: {{/* something completely.invalid !!!  */}}
type: {{ .Values.service | typeOf | printf }}
### print complex value
{{ .Values.service | toYaml }}
### indent it
indented:
{{ .Values.service | toYaml | indent 2 }}
```

Exécutez ensuite `helm template`.

Le fichier généré ne constitue pas un manifest YAML valide pour Kubernetes, mais représente une excellente technique de débogage pour vos templates !

