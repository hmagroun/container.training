# Exécuter un cluster Consul

- Voici un cas d'usage pertinent pour les Stateful sets !

- Nous allons déployer un cluster Consul à 3 nodes

- Consul est un magasin clé/valeur hautement disponible

  (similaire à etcd ou Zookeeper)

- Une méthode simple pour initialiser (bootstrap) un cluster consiste à fournir à chaque node :

  - les adresses des autres nodes

  - le nombre de nodes attendus (pour déterminer l'atteinte du quorum)

---

## Initialisation d'un cluster Consul

*Après une lecture attentive de la documentation de Consul (et/ou quelques échanges avec la communauté),
on identifie la ligne de commande minimale pour exécuter notre cluster Consul.*

```
consul agent -data-dir=/consul/data -client=0.0.0.0 -server -ui \
       -bootstrap-expect=3 \
       -retry-join=`X.X.X.X` \
       -retry-join=`Y.Y.Y.Y`
```

- Remplacez X.X.X.X et Y.Y.Y.Y par les adresses des autres nodes

- Un node peut inclure sa propre adresse sans problème

- ... Ce qui signifie qu'on peut utiliser la même ligne de commande sur l'ensemble des nodes (très pratique !)

---

## Cloud Auto-join

- Depuis la version 1.4.0, Consul sait exploiter l'API Kubernetes pour localiser ses pairs

- C'est la fonctionnalité [Cloud Auto-join]

- Au lieu de passer une adresse IP fixe, on transmet un paramètre sous cette forme :

  ```
  consul agent -retry-join "provider=k8s label_selector=\"app=consul\""
  ```

- Consul doit être en mesure d'interagir avec l'API Kubernetes

- On peut lui monter un fichier `kubeconfig`

- Si Consul s'exécute dans un pod, il exploitera le *service account* associé à ce pod

[Cloud Auto-join]: https://www.consul.io/docs/agent/cloud-auto-join.html#kubernetes-k8s-

---

## Configurer le Cloud auto-join

- Nous devons créer un service account pour Consul

- Nous devons créer un role doté des droits `list` et `get` sur les pods

- Nous devons associer (bind) ce role au service account

- Et bien sûr, nous devons nous assurer que les pods Consul utilisent ce service account

---

## Assemblage des composants

- Le fichier `k8s/consul-1.yaml` définit les ressources requises

  (service account, role, role binding, service, stateful set)

- Inspiré de cet [excellent tutoriel](https://github.com/kelseyhightower/consul-on-kubernetes) de Kelsey Hightower

  (de nombreuses fonctionnalités du tutoriel d'origine ont été épurées par souci de simplicité)

---

## Lancer notre cluster Consul

- Nous allons utiliser le fichier YAML fourni

.lab[

- Créer le stateful set et le service associé :
  ```bash
  kubectl apply -f ~/container.training/k8s/consul-1.yaml
  ```

- Suivre les logs à mesure que les pods démarrent séquentiellement :
  ```bash
  stern consul
  ```

<!--
```wait Synced node info```
```key ^C```
-->

- Vérifier l'état de santé du cluster :
  ```bash
  kubectl exec consul-0 -- consul members
  ```

]

---

## Mises en garde et limitations

- Le scheduler peut planifier deux pods Consul sur le même node

  - en cas de défaillance de ce node, deux pods Consul sont perdus en même temps
  - cela entraîne une rupture de service du cluster

- Réduire la taille du cluster (scaling down) va causer sa défaillance

  - lorsqu'un membre Consul quitte le cluster, il doit notifier les autres
  - sinon, le dernier node restant perd le quorum et cesse de fonctionner

- Ce cluster Consul n'utilise pas encore de persistance réelle

  - les données sont stockées sur le système de fichiers éphémère des conteneurs
  - si un pod crash, son remplaçant redémarre à partir d'un état vierge

---

## Optimiser le placement des pods

- Nous devons spécifier au scheduler :

  *ne positionne pas deux de ces pods sur le même node !*

- On configure cela via une section `affinity` telle que :
  ```yaml
    affinity:
      podAntiAffinity:
        requiredDuringSchedulingIgnoredDuringExecution:
          - labelSelector:
              matchLabels:
                app: consul
            topologyKey: kubernetes.io/hostname
  ```

---

## Utiliser un hook de cycle de vie

- Lorsqu'un membre Consul quitte le cluster, il doit impérativement exécuter :
  ```bash
  consul leave
  ```

- Cela s'implémente via une section `lifecycle` comme suit :
  ```yaml
    lifecycle:
      preStop:
        exec:
          command: [ "sh", "-c", "consul leave" ]
  ```

---

## Déployer un cluster Consul résilient

- Ajoutons maintenant la contrainte d'ordonnancement et le hook de cycle de vie

- L'opération peut être effectuée dans le même namespace ou un namespace distinct (selon vos préférences)

- Si nous restons dans le même namespace, un rolling update sera déclenché

  (les pods seront remplacés un par un)

.lab[

- Déployer la version améliorée du cluster Consul :
  ```bash
  kubectl apply -f ~/container.training/k8s/consul-2.yaml
  ```

]

---

## Toujours aucune persistance

- Nous n'exploitons toujours pas de persistance durable

  (aucun `volumeClaimTemplate`, Persistent Volume, etc.)

- Que se passe-t-il si un pod est perdu ?

  - un nouveau pod est reprogrammé par le scheduler (avec un état vide)

  - le nouveau pod tente de contacter les deux autres

  - il sera intégré au cluster (après 1 à 2 minutes d'instabilité)

  - puis il synchronisera les données auprès des autres pods

---

## Modes de défaillance

- Que se passe-t-il si l'on perd deux pods ?

  - une intervention manuelle est requise

  - il faudra forcer le membre restant à fonctionner en mode autonome (solo)

  - avant d'y rattacher de nouveaux pods

- Que se passe-t-il si l'on perd trois pods (la totalité du cluster) ?

  - l'ensemble des données est définitivement perdu

???

:FR:- Lancer des pods ensemble ou séparément
:FR:- Exemple : lancer un cluster Consul