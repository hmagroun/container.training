# Modèle réseau de Kubernetes

- En résumé (TL,DR) :

  *Notre cluster (nodes et pods) forme un unique grand réseau IP plat (flat IP network).*

--

- Dans le détail :

  - tous les nodes peuvent communiquer directement entre eux (sans NAT)

  - tous les pods peuvent communiquer directement entre eux (sans NAT)

  - les pods et les nodes peuvent communiquer directement entre eux (sans NAT)

  - chaque pod a connaissance de sa propre adresse IP (toujours sans NAT)

- La majorité des clusters Kubernetes s'appuient sur la CNI pour configurer le réseau des pods

  (allocation des adresses IP, création et configuration des interfaces réseau, routage...)

---

## Modèle réseau de Kubernetes : les points forts

- Tout peut joindre tout

- Pas de translation d'adresses (NAT)

- Pas de translation de ports (PAT)

- Aucun protocole propriétaire spécifique

- Les adresses IP sont allouées par la stack réseau, pas par les utilisateurs

  (cela évite les contraintes complexes liées à la portabilité des adresses)

- La CNI est hautement flexible et supporte de multiples architectures

  (switching, routage, tunneling... pratiquement tout est possible !)

- Exemple : on peut allouer un sous-réseau par node et exploiter une topologie routée simple

---

## Modèle réseau de Kubernetes : les contraintes

- Tout peut joindre tout

  - si nous souhaitons de l'isolation réseau, nous devons implémenter des network policies

  - certains clusters managés (comme AWS EKS) n'intègrent pas de contrôleur de network policy par défaut

- Il existe littéralement des dizaines d'implémentations réseau différentes pour Kubernetes

  (le dépôt https://github.com/containernetworking/cni/ référence plus de 25 plugins)

- Les pods bénéficient d'une connectivité de niveau 3 (IP), mais les *services* opèrent au niveau 4 (TCP ou UDP)

  (les Services mappent vers un port TCP ou UDP précis ; pas de plages de ports arbitraires ni de paquets IP bruts)

- Le proxy de service Kubernetes par défaut, `kube-proxy`, montre des limites en termes de montée en charge

  (bien que cela ait été considérablement optimisé dans les [versions récentes de kube-proxy][tables-have-turned])

[tables-have-turned]: https://www.youtube.com/watch?v=yOGHb2HjslY

---

## Modèle réseau de Kubernetes : en pratique

- Nous n'avons pas à nous soucier de la configuration réseau sur des clusters de développement local

  (tout est configuré automatiquement et il n'est presque jamais nécessaire d'y toucher)

- Nous n'avons généralement pas non plus à nous en préoccuper sur les clusters managés

  (sauf si nous souhaitons reconfigurer ou remplacer la stack installée par défaut)

- Nous *devons* impérativement choisir une stack réseau dans les autres contextes :

  - déploiement de Kubernetes sur bare metal ou sur des machines virtuelles brutes

  - lorsque nous administrons nous-mêmes le control plane

---

## Quelle stack réseau choisir ?

*Cela dépend de votre contexte !*

- [Weave] = extrêmement simple à déployer, sans configuration requise, empreinte légère...
  
  *mais hélas, il n'est plus maintenu !*

- [Cilium] = extrêmement puissant et flexible, considéré par beaucoup comme la référence...

  *mais il repose sur eBPF, ce qui peut rendre le troubleshooting plus complexe !*

- D'autres alternatives solides incluent [Calico], [Flannel], [kube-router]

- Et bien sûr, certains fournisseurs cloud ou équipementiers réseau proposent leurs propres solutions

  (qui peuvent être plus ou moins adaptées à vos cas d'usage !)

- Vos priorités vont-elles vers la performance brute ? La fiabilité ? La sécurité ? L'observabilité ?

[Weave]: https://github.com/weaveworks/weave
[Cilium]: https://cilium.io/
[Calico]: https://docs.tigera.io/calico/latest/about/
[Flannel]: https://github.com/flannel-io/flannel
[kube-router]: https://www.kube-router.io/

---

## Une architecture multicouche

- Le "réseau pod-to-pod" (ou "pod network", ou "CNI") :

  - assure la communication entre les pods et les nodes

  - est généralement pris en charge par des plugins CNI

- Le "réseau pod-to-service" (ou "Kubernetes service proxy") :

  - assure la communication interne et la répartition de charge

  - est implémenté par défaut avec `kube-proxy`

- Les Network Policies :

  - assurent le filtrage (pare-feu) et l'isolation réseau

  - peuvent être intégrées directement au "pod network" ou fournies par un composant tiers dédié

---

class: pic

![Vue d'ensemble des trois couches réseau de Kubernetes](images/k8s-net-0-overview.svg)

---

class: pic

![Réseau pod-to-pod](images/k8s-net-1-pod-to-pod.svg)

---

class: pic

![Réseau pod-to-service](images/k8s-net-2-pod-to-svc.svg)

---

class: pic

![Network policies](images/k8s-net-3-netpol.svg)

---

class: pic

![Vue d'ensemble complète des couches réseau](images/k8s-net-4-overview.svg)

---

class: extra-details

## Éléments d'architecture complémentaires

- Le trafic entrant (inbound traffic) peut être géré par différents composants :

  - un composant comme `kube-proxy` ou `kube-router` (pour les services de type NodePort)

  - des load balancers (idéalement connectés directement au réseau des pods)

- Il est tout à fait possible d'exploiter plusieurs réseaux de pods en parallèle

  (via des "méta-plugins" tels que CNI-Genie ou Multus)

- Certaines solutions couvrent plusieurs rôles simultanément

  (par ex. `kube-router` peut assurer le réseau de pods, appliquer les network policies et remplacer `kube-proxy`)

???

:FR:- Le modèle réseau de Kubernetes