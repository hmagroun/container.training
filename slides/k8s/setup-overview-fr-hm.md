# Installer Kubernetes

- Kubernetes est composé de nombreux composants nécessitant une configuration rigoureuse

- Une exploitation sécurisée requiert généralement des certificats TLS et une CA locale

  (autorité de certification)

- Tout déployer manuellement est faisable, mais rarement pratiqué

  (sauf dans un cadre d'apprentissage)

- Faisons un tour d'horizon rapide des options disponibles !

---

## Développement local

- Vous écrivez du code destiné à s'exécuter à terme sur Kubernetes ?

- Disposer d'un cluster de développement dédié est alors une excellente pratique !

- Plutôt que de simplement distribuer des images de conteneurs, nous pouvons les valider directement sur Kubernetes

- Particulièrement précieux lors de l'écriture ou du test d'objets spécifiques à Kubernetes

  (ConfigMaps, Secrets, StatefulSets, Jobs, RBAC, etc.)

- Idéal pour vérifier rapidement la structure d'une ressource

  (par ex. quels sont les champs disponibles dans la spec d'un Deployment ?)

---

## Clusters à nœud unique (Single-node)

- Travailler avec un cluster ne comportant qu'un seul node est parfaitement adapté

- Cela simplifie énormément d'aspects :

  - le réseau des pods ne nécessite ni plugins CNI, ni overlay networks, etc.

  - ces clusters peuvent être entièrement encapsulés dans une VM ou une image de conteneur prête à l'emploi

  - certains volets de sécurité sont simplifiés (modèle de menace différent)

  - les images peuvent être buildées directement sur le node (évitant l'usage d'un registry intermédiaire)

- Exemples : Docker Desktop, k3d, KinD, MicroK8s, Minikube

  (certains d'entre eux supportent également des topologies multi-nodes)

---

## Clusters managés ("Solutions clés en main")

- De nombreux fournisseurs cloud et hébergeurs proposent des offres de "Kubernetes managé"

- Le déploiement et la maintenance du *control plane* sont intégralement pris en charge par le fournisseur

  (idéalement, les clusters peuvent être instanciés automatiquement via une API, une CLI ou une interface web)

- Compte tenu de la complexité intrinsèque de Kubernetes, cette approche est *fortement recommandée*

  (au moins pour vos premiers clusters de production)

- Après une période d'exploitation sur Kubernetes, vous serez mieux armés pour décider :

  - s'il convient d'opérer vous-mêmes votre infrastructure ou de recourir à une solution managée

  - quelle distribution ou offre répond le plus précisément à vos contraintes

---

## Managé ≠ hébergé avec infogérance

- Kubernetes managé ≠ hébergement infogéré

- L'infogérance implique généralement que l'hébergeur prenne en charge :

  - l'installation, les montées de version, les correctifs de sécurité critiques, les sauvegardes

  - la collecte des logs et des métriques

  - la mise en place de la supervision, du système d'alerting et des astreintes (on-call)

- Un service de Kubernetes managé signifie généralement que le provider prend en charge :

  - l'installation initiale

  - éventuellement les upgrades (dans une certaine mesure : leur déclenchement et coordination restent sous votre responsabilité)

  - et rien de plus !

---

## Kubernetes "managé"

- Un "Kubernetes managé" fournit l'équivalent d'une machine virtuelle brute

- Il reste nécessaire d'intégrer de nombreuses briques pour atteindre un niveau de préparation à la production (production-ready)

  (upgrades, logging, supervision...)

- Il faut également ajouter des composants quasi indispensables non fournis par défaut :

  - ingress controller

  - network policy controller

  - storage class...

📽️ [How to make Kubernetes ryhme with production readiness](https://www.youtube.com/watch?v=6G4v-ZE6OHI)

---

## Observabilité

- Logs, métriques, traces...

- Sélectionner une solution (auto-hébergée ou as-a-service ?)

- Configurer le control plane, les nodes et les divers composants

- Mettre en place des dashboards, superviser les métriques critiques

  (par ex. sur AWS, tracer les flux inter-AZ et le trafic sortant par application pour éviter les mauvaises surprises de facturation)

- Mettre en place la supervision, les notifications d'incident et la rotation des astreintes

---

## Sauvegardes (Backups)

- Sauvegarde complète au niveau système des nodes ?

  (peu pertinent et inefficace)

- Sauvegarde des données du control plane ?

  (essentiel ; bien qu'il ne soit pas toujours possible d'extraire des sauvegardes etcd sur du managé)

- Sauvegarde des volumes persistants ?

  (bonne pratique ; mais pas toujours suffisante)

- Sauvegardes applicatives, par ex. dumps de bases de données, log-shipping ?

  (plus efficaces et fiables ; demandent un effort d'ingénierie selon l'application et le moteur de base de données)

---

## Montées de version (Upgrades)

- Control plane

  *généralement automatisées par le fournisseur ; mais peuvent provoquer des ruptures de compatibilité*

- Nodes

  *scénario idéal : mise à jour in-place ; sinon : implique le reprovisioning de nouveaux nodes*

- Composants additionnels (ingress controller, operators, etc.)

  *très variable selon la nature des composants !*

---

## Ne partez pas seuls !

N'hésitez pas à vous faire accompagner par des experts avant de basculer votre première application en production sur K8S !

---

## Gestion des nodes

- La plupart des "solutions clés en main" intègrent des control planes entièrement infogérés

  (incluant les upgrades du control plane, parfois exécutées automatiquement)

- Cependant, chez la majorité des providers, la gestion des *nodes* reste à votre charge

  (provisioning, mise à niveau, dimensionnement des nodes)

- Exemple avec les ["managed node groups" d'Amazon EKS](https://docs.aws.amazon.com/eks/latest/userguide/managed-node-groups.html) :

  *...when bugs or issues are reported [...] you're responsible for deploying these patched AMI versions to your managed node groups.*

---

## Différences entre offres managées

- La plupart des fournisseurs permettent de sélectionner la version cible de Kubernetes

  - certains mettent à disposition des versions très récentes

  - d'autres accusent un retard important (parfois de 2 à 3 versions mineures)

- Certains providers proposent de multiples options de networking ou de stockage

- D'autres n'en supportent qu'une seule, étroitement liée à leur infrastructure propriétaire

  (en changer est théoriquement possible, mais peut s'avérer complexe ou non supporté)

- Certains fournisseurs permettent de configurer ou d'ajuster le control plane

  (généralement via les "feature gates" de Kubernetes)

---

## Choisir un fournisseur

- Les modèles de tarification varient sensiblement d'un provider à un autre

  - les nodes sont généralement facturés à leur tarif nominal standard

  - le control plane peut être gratuit ou soumis à une tarification horaire fixe

- Au-delà du coût, les écarts fonctionnels entre providers sont *majeurs*

- Les acteurs majeurs ("hyperscalers") ne sont pas systématiquement les plus adaptés !

- Consultez [cette page](https://kubernetes.io/docs/setup/production-environment/turnkey-solutions/) pour une vue d'ensemble des fournisseurs certifiés

---

## Distributions et installeurs Kubernetes

- Si vous préférez opérer Kubernetes par vous-mêmes, de nombreuses options existent

  (gratuites, commerciales, propriétaires, open source...)

- Certaines constituent de simples outils d'installation, d'autres de véritables plateformes applicatives

- Certaines s'appuient sur des outils de déploiement éprouvés

  (comme Puppet, Terraform...)

- Les options sont trop nombreuses pour être toutes énumérées

  (consultez [cette page](https://kubernetes.io/partners/#iframe-landscape-conformance) pour un panorama complet !)

---

## kubeadm

- kubeadm est un outil officiel intégré à Kubernetes conçu pour simplifier l'initialisation de clusters

- De nombreux autres installeurs et distributions s'appuient dessus (bien que ce ne soit pas systématique)

- Il s'utilise également de manière autonome

- Point d'entrée de référence pour déployer Kubernetes sur vos propres machines

  (environnements virtuels ou physiques indifféremment)

- Supporte la haute disponibilité sur le control plane ("multi-master")

  (cela introduit néanmoins une complexité supplémentaire liée au load balancer devant l'API)

---

## Déploiement manuel

- Les ressources suivantes sont avant tout à vocation pédagogique !

- [Kubernetes The Hard Way](https://github.com/kelseyhightower/kubernetes-the-hard-way) par Kelsey Hightower

  *guide étape par étape pour déployer Kubernetes sur GCP, avec certificats, HA...*

- [Deep Dive into Kubernetes Internals for Builders and Operators](https://www.youtube.com/watch?v=3KtEAa7_duA)

  *conférence détaillant l'installation d'un cluster Kubernetes simplifié, sans couche de sécurité ni HA*

- 🇫🇷 [Démystifions les composants internes de Kubernetes](https://www.youtube.com/watch?v=OCMNA0dSAzc)

  *version enrichie de la précédente, avec gestion des certificats et versions récentes de k8s*

---

## À propos de nos clusters de formation

- Comment ont été initialisés les clusters Kubernetes que nous utilisons ?

--

- Nous avons utilisé `kubeadm` sur des instances de VM fraîchement déployées sous Ubuntu LTS

    1. Installation de Docker

    2. Installation des paquets Kubernetes

    3. Exécution de `kubeadm init` sur le premier node (déploie le control plane sur ce node)

    4. Déploiement de Cilium (le réseau overlay) via une unique commande `helm install`

    5. Exécution de `kubeadm join` sur les autres nodes (avec le token généré par `kubeadm init`)

    6. Copie du fichier de configuration généré par `kubeadm init`

- Consultez le [README prepare VMs](https://@@GITREPO@@/blob/master/prepare-vms/README.md) pour plus de précisions

---

## "Limites" de `kubeadm`

- N'installe pas Docker ni aucun autre container engine

  (choix de conception délibéré pour laisser une totale liberté de runtime)

- Ne déploie pas le réseau overlay

  (également par conception, pour les mêmes raisons)

- Un control plane en HA nécessite [des étapes de configuration complémentaires](https://kubernetes.io/docs/setup/independent/high-availability/)

- Notez qu'un control plane HA impose également de configurer un load balancer dédié devant l'API

  (ce qui sort du périmètre d'action de kubeadm)

???

:FR:- Survol des techniques d'installation de Kubernetes