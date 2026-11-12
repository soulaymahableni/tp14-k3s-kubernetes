# TP14 : réponses aux questions

## Partie 1 : architecture et cluster

| Composant | Rôle | Dans ce cluster |
|---|---|---|
| API server | Point d'entrée unique : `kubectl`, kubelet et controllers passent tous par lui (HTTPS 6443) | Processus `k3s server` sur `k3s-server` |
| Datastore | Stocke l'état désiré et l'état observé | SQLite (`/var/lib/rancher/k3s/server/db/`) ; etcd embarqué en haute disponibilité |
| Scheduler | Choisit un nœud pour chaque Pod en fonction des requests et des contraintes | Dans `k3s server` |
| Controllers | Boucles de réconciliation (Deployment → ReplicaSet → Pods, endpoints…) | Dans `k3s server` |
| kubelet | Agent de chaque nœud : lance les conteneurs via containerd, exécute les probes, remonte l'état | Sur `k3s-server` et `k3s-agent1` |

**Points clés du playbook / rôle**
* `creates: /usr/local/bin/k3s` rend l'installation idempotente : un second `vagrant provision` ne réinstalle rien.
* `--node-ip` et `--flannel-iface` forcent l'IP du réseau privé (192.168.56.x). Sous VirtualBox, `eth0` (NAT) porte la même IP `10.0.2.15` sur toutes les VM : sans ces options, les nœuds s'annoncent avec la même IP et le réseau des Pods entre nœuds ne fonctionne pas.
* `--tls-san 192.168.56.10` ajoute l'IP au certificat de l'API server, ce qui permet d'utiliser `kubectl` depuis la VM de lab.
* `--write-kubeconfig-mode 644` rend `k3s.yaml` lisible sans `sudo`.
* Le jeton est lu avec `slurp`, partagé via `hostvars`, et masqué (`no_log`) dans la tâche de l'agent.
* `INSTALL_K3S_VERSION` est épinglé dans `defaults/main.yml` : chaque reconstruction donne exactement le même cluster.

**Temps de reconstruction** (Plan B : `k3s-uninstall.sh` puis réinstallation) : **4 min 17 s** (`real 4m17,165s`).
L'intérêt : le cluster est **jetable et reproductible**. Toute la configuration est dans le code (Vagrantfile + rôle), rien n'est fait à la main.

## Partie 2 : Pods et Deployments

**Pod lancé avec `kubectl run` puis supprimé : il n'est pas recréé.** C'est un Pod « nu » : aucun controller ne le surveille, donc personne ne réconcilie l'écart.

**Pod d'un Deployment supprimé : il est recréé immédiatement.** Le Deployment gère un ReplicaSet dont l'état désiré est `replicas: 3`. Le controller de ReplicaSet constate 2 Pods au lieu de 3 et en crée un nouveau (nouveau nom, nouvelle IP).

**Répartition** : le scheduler répartit les Pods sur les deux nœuds (colonne `NODE` de `-o wide`). Il tient compte des requests et étale les Pods d'un même ReplicaSet. Le server K3s accepte aussi des Pods, car il n'a pas de taint par défaut.

**`kubectl exec -it … -- sh` échoue** (`exec: "sh": executable file not found in $PATH`) : l'image `whoami` est construite `FROM scratch` et ne contient que le binaire Go. Cela réduit la surface d'attaque. Pour déboguer, on utilise un conteneur éphémère : `kubectl debug -it <pod> --image=busybox:1.37 --target=whoami`.

**`kubectl explain deployment.spec.strategy`** : deux types possibles. `RollingUpdate` (par défaut) remplace progressivement les Pods selon `maxUnavailable` et `maxSurge`. `Recreate` supprime tous les Pods avant de créer les nouveaux, avec une coupure ; c'est celui utilisé pour PostgreSQL à cause du volume RWO.

## Partie 3 : Services et Ingress

* Le **selector** du Service (`app: whoami`) doit correspondre aux labels des Pods : il alimente les **EndpointSlices** (IP:port des Pods *prêts*). Avec un selector faux, les endpoints sont vides et le Service ne répond pas.
* `port-forward svc/whoami` choisit **un seul Pod** au démarrage : `Hostname` reste fixe tant que le port-forward tourne, et change quand on le relance. L'Ingress, lui, répartit chaque requête entre les Pods.
* **DNS interne** : CoreDNS résout `whoami.tp14.svc.cluster.local` (`<service>.<namespace>.svc.cluster.local`) vers la ClusterIP. Depuis le même namespace, `http://whoami` suffit.
* **Ingress** : Traefik (inclus dans K3s) écoute sur les ports 80/443 des nœuds via ServiceLB, et route selon l'hôte `whoami.lab.local` et le chemin : `/api` vers `nginx-api`, `/` vers `whoami`. Le chemin n'est pas réécrit : nginx répond lui-même sur `/api`.

## Partie 4 : ConfigMap et Secret

* `envFrom.configMapRef` injecte toutes les clés de la ConfigMap. `env.valueFrom.secretKeyRef` injecte une seule clé du Secret (`API_KEY`).
* **base64 n'est pas du chiffrement** : c'est un simple encodage, réversible par tout le monde (`base64 -d`). La protection d'un Secret repose sur le RBAC (qui a le droit de faire `get secret`), sur le chiffrement au repos du datastore (`k3s server --secrets-encryption`) et sur le fait de ne jamais le versionner.
* **Modifier la ConfigMap ne change pas les Pods existants** : les variables d'environnement sont lues une seule fois, au démarrage du conteneur. Il faut recréer les Pods : `kubectl rollout restart deploy/whoami`, qui fait un rolling update sans coupure. Une ConfigMap montée en **volume**, elle, est mise à jour dans le Pod après un délai (environ une minute), mais l'application doit relire le fichier.

## Partie 5 : santé, ressources, déploiements progressifs

* **readinessProbe** en échec : le Pod passe `0/1` et sort des endpoints, donc il ne reçoit plus de trafic. Il n'est pas redémarré.
* **livenessProbe** en échec (3 fois de suite) : le kubelet tue le conteneur et le relance (`RESTARTS` +1, événements `Liveness probe failed` puis `Killing`). Après le redémarrage, `/health` revient à 200, car l'état est en mémoire.
* **requests** : réservation utilisée par le scheduler et base de calcul du HPA. **limits** : plafond. Au-delà de la limite CPU, le conteneur est ralenti (throttling) ; au-delà de la limite mémoire, il est tué (**OOMKilled**, code 137). C'est ce qu'on observe avec `limits.memory: 4Mi`.

**Pourquoi aucune interruption pendant les mises à jour ?**
* `maxUnavailable: 0` : Kubernetes ne supprime jamais un ancien Pod avant qu'un nouveau soit **prêt**.
* `maxSurge: 1` : il crée un Pod supplémentaire à la fois (4 Pods au maximum pendant la mise à jour).
* `readinessProbe` : un nouveau Pod ne reçoit du trafic qu'une fois `/health` à 200.
* Avec l'image `v9.9.9` inexistante, le nouveau Pod reste en `ImagePullBackOff` et n'est jamais prêt. Le rollout est donc bloqué, et les 3 anciens Pods continuent de servir. `rollout undo` revient à la révision précédente (visible dans `rollout history`, limitée par `revisionHistoryLimit`).

## Partie 6 : stockage persistant

* Le PVC `postgres-data` (StorageClass `local-path`) est lié à un PV créé automatiquement. Supprimer le Pod ne supprime ni le PVC ni le PV : le nouveau Pod remonte le même volume et retrouve la table.
* **Emplacement physique** : un dossier sur le disque du nœud où le Pod a démarré la première fois, `/var/lib/rancher/k3s/storage/pvc-<uid>_tp14_postgres-data` (visible avec `kubectl get pv -o yaml`, champ `spec.local.path`).
* **Limite** : le PV porte une `nodeAffinity` vers ce nœud. Le Pod ne peut donc **plus être planifié ailleurs**. Si le nœud tombe ou est drainé, PostgreSQL reste `Pending` et les données sont inaccessibles (et perdues si le disque l'est). Il n'y a ni réplication ni haute disponibilité. En production, on utilise un stockage réseau ou répliqué (Longhorn, Ceph/Rook, NFS, disques cloud via CSI), ou une base managée.
* `strategy: Recreate`, parce qu'un volume `ReadWriteOnce` ne peut pas être monté par deux Pods sur des nœuds différents pendant un rolling update.

## Partie 7 : HorizontalPodAutoscaler

* metrics-server (inclus dans K3s) fournit `kubectl top` et les métriques CPU/mémoire au HPA.
* Le HPA vise 50 % des **requests** CPU (20m, donc 10m par Pod). Calcul : `replicas = ceil(replicas_actuels × utilisation_actuelle / 50 %)`, borné entre 2 et 8.
* Sous charge, `TARGETS` dépasse 50 % et le nombre de replicas augmente. À l'arrêt de la charge, il redescend à 2 après la fenêtre de stabilisation (60 s ici, 300 s par défaut), pour éviter les oscillations.
* Sans `requests.cpu` sur les Pods, le HPA affiche `<unknown>` et ne peut rien calculer.

## Défis

* **Troisième nœud** : il suffit d'ajouter une entrée dans `NODES`. Les groupes Ansible et `node_ip` sont générés à partir du tableau, et le rôle rejoint le cluster automatiquement.
* **`kubectl drain`** : marque le nœud `SchedulingDisabled` (cordon) et évince ses Pods. Les ReplicaSets les recréent sur les autres nœuds. `uncordon` le rend de nouveau planifiable.
* **PodDisruptionBudget `minAvailable: 1`** : avec un seul Pod whoami, l'éviction ferait passer le nombre de Pods disponibles à 0. Le drain est donc refusé (`Cannot evict pod as it would violate the pod's disruption budget`) et réessaie jusqu'au timeout. Avec 2 Pods ou plus répartis, le drain passe.

## Remarques sur l'environnement (Plan B)
* Pas de virtualisation imbriquée et 5 Go de RAM : K3s installé directement sur la VM de lab (un seul nœud). Le Vagrantfile et le rôle Ansible sont livrés pour la version deux nœuds.
* Incident PostgreSQL : le Pod restait `0/1`. `gosu` était bloqué, puis `pg_ctl: could not send stop signal: Permission denied`, et la base restait en `recovery mode`. `dmesg` montrait `apparmor="DENIED" operation="signal" profile="cri-containerd.apparmor.d"`. Correction dans `06-postgres.yaml` : `runAsUser/runAsGroup/fsGroup: 70` (plus de `gosu`, pas de root) et `appArmorProfile: Unconfined` pour ce seul Pod, puis suppression du PVC pour repartir d'un répertoire de données sain.
* `helm-install-traefik-crd` en erreur : les CRD Gateway API appartenaient déjà à une autre release Helm (`gateway-api-crd`) installée lors d'un TP précédent. Sans impact sur l'Ingress (`networking.k8s.io/v1`).
