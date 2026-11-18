# Rendu — Atelier Kubernetes

> Complétez ce fichier au fil de l'atelier, puis faites un commit et un push (voir la section « Rendu » du README).

**Hammami / Elyess :**
**Groupe : I3_APP_RS2**

---

## Exercice 1 — Cluster

1. Combien de nœuds affiche `kubectl get nodes` ? Quel est le rôle de chacun ?
   > `kubectl get nodes` affiche 3 nœuds :
   - `k3d-lab-server-0` (rôle `control-plane`) : le control plane. Il héberge l'API Server, le scheduler, le controller-manager et la base d'état du cluster ; c'est lui qui décide où placer les pods.
   - `k3d-lab-agent-0` et `k3d-lab-agent-1` (rôle `<none>`) : les workers, qui exécutent les pods des applications.

2. Que montre `docker ps` ? Qu'est-ce qu'un « nœud » dans k3d, concrètement ?
   > `docker ps` montre 4 conteneurs :
   - `k3d-lab-server-0`, `k3d-lab-agent-0` et `k3d-lab-agent-1`, avec l'image `rancher/k3s:v1.35.5-k3s1` : un conteneur par nœud du cluster ;
   - `k3d-lab-serverlb`, avec l'image `k3d-proxy` : le load balancer de k3d, qui relie le port 8081 du Codespace au cluster.
   Dans k3d, un « nœud » est un conteneur Docker qui fait tourner k3s. On le voit aussi dans les IP internes des nœuds (172.18.0.2 à 172.18.0.4) : ce sont les adresses de ces conteneurs sur le réseau Docker.

## Exercice 2 — Premier déploiement

3. Quels objets Kubernetes ont été créés par `kubectl create deployment` ? (indice : `kubectl get all`)
   > `kubectl create deployment` a créé trois objets : 
   - un Deployment (`deployment.apps/web`)
   - un ReplicaSet (`replicaset.apps/web-xxxxxxxxx`) géré par ce Deployment
   - un Pod (`pod/web-xxxxxxxxx-yyyyy`) créé par le ReplicaSet. 
4. Vous avez supprimé le pod `web-...` : que s'est-il passé, et pourquoi ?
   > Le pod a été supprimé, mais un nouveau pod (avec un nouveau nom) a immédiatement été recréé. Le ReplicaSet doit maintenir le nombre de réplicas demandé (1) : il a constaté un écart entre l'état désiré (1 pod) et l'état réel (0 pod) et a recréé un pod. C'est la boucle de réconciliation de Kubernetes (auto-réparation).

## Exercice 3 — Modifier un conteneur en direct

5. Après suppression du pod, votre message « Bienvenue... » a-t-il survécu ? Pourquoi ?
   > Non, le message n'a pas survécu : on retrouve la page par défaut de nginx. Le système de fichiers d'un conteneur est éphémère : le nouveau pod est recréé à partir de l'image `nginx` d'origine, et la modification faite à la main dans l'ancien conteneur est perdue avec lui.
6. Pourquoi le `port-forward` a-t-il cessé de fonctionner après la suppression du pod ?
   > Un `port-forward` vers un Service ne fait pas de répartition de charge : au lancement, il choisit **un seul pod** derrière le Service et s'y attache. Quand ce pod est supprimé, la connexion n'a plus de cible et le tunnel se termine (erreur visible dans `/tmp/web.log`). Il faut relancer le tunnel pour qu'il s'attache au nouveau pod.

## Exercice 4 — Déclaratif

7. Quelle différence entre `kubectl create` et `kubectl apply` ?
   > `kubectl create` est impératif : on donne un ordre (« crée cet objet »). Il échoue si l'objet existe déjà et ne permet pas de le mettre à jour.
   `kubectl apply` est déclaratif : on décrit l'état désiré dans un fichier YAML, et Kubernetes crée l'objet s'il n'existe pas ou le met à jour pour qu'il corresponde au fichier. On peut le relancer autant de fois qu'on veut.
8. À quoi sert le `selector` d'un Service ? Que se passe-t-il s'il ne correspond à aucun pod ?
   > Le `selector` d'un Service indique quels pods reçoivent le trafic : le Service cible tous les pods dont les labels correspondent (ici `app: site`) et tient à jour sa liste d'endpoints.
   S'il ne correspond à aucun pod, le Service existe mais n'a aucun endpoint (`kubectl get endpoints site` est vide) : le trafic n'est envoyé nulle part, les connexions échouent et un `port-forward` vers ce Service ne trouve aucun pod.

## Exercice 5 — Scaling & mises à jour

9. Sur quels nœuds se trouvent vos 3 réplicas ? Collez la sortie de `kubectl get pods -l app=site -o wide`.
   ```
NAME                    READY   STATUS    RESTARTS   AGE   IP          NODE               NOMINATED NODE   READINESS GATES
site-6694f66dcd-g7m4w   1/1     Running   0          30m   10.42.0.7   k3d-lab-server-0   <none>           <none>
site-6694f66dcd-n2l4n   1/1     Running   0          30m   10.42.1.5   k3d-lab-agent-1    <none>           <none>
site-6694f66dcd-sfp2w   1/1     Running   0          46m   10.42.3.8   k3d-lab-agent-0    <none>           <none>
   ```
10. Avec l'image `nginx:9.99`, quel statut avait le pod ? Quelle commande vous a donné la cause ?
    > Le nouveau pod était en **`ErrImagePull` puis `ImagePullBackOff`** : Kubernetes n'arrivait pas à télécharger l'image car le tag `nginx:9.99` n'existe pas sur Docker Hub. La cause exacte est donnée par `kubectl describe pod -l app=site`, dans la section *Events* (message « Failed to pull image "nginx:9.99" ... not found »).
11. Pendant la panne, le site restait-il accessible ? Pourquoi ?
    > Oui, le site restait accessible. Le rolling update ne supprime les anciens pods qu'une fois les nouveaux prêts : comme le nouveau pod n'a jamais démarré, le déploiement est resté bloqué et les 3 anciens pods (nginx:1.28) ont continué à servir le trafic. 

## Exercice 6 — Portainer & Mario

12. URL publique de votre Mario :
    > https://jubilant-engine-wrrp9rjpq4cp4r-8585.app.github.dev/
13. Retrouvez le Deployment `mario` en ligne de commande. Portainer et `kubectl` parlent-ils au même composant ? Lequel ?
    > Portainer et `kubectl` parlent au même composant : l'API Server (kube-apiserver) du control plane. `kubectl` l'utilise via le fichier kubeconfig ; Portainer, qui tourne dans le cluster, l'utilise via son ServiceAccount, auquel l'installation a donné les droits nécessaires. Portainer n'est qu'une interface graphique au-dessus de l'API Kubernetes.

## Bonus (facultatif)

14. Bonus réalisés et remarques :
    >

---

**Score obtenu avec `./scripts/check.sh` :** 
./scripts/check.sh 

Exercice 1 — Cluster
  ✔ Le cluster k3d 'lab' existe
  ✔ 3 nœuds sont Ready (1 server + 2 agents)
  ✔ Le port 8081 est relié au load balancer

Exercice 2 — Premier déploiement
  ✔ Le Deployment 'web' est disponible
  ✔ Le Service 'web' existe

Exercice 4 — Déclaratif
  ✔ Les TODO des manifestes k8s/site/ sont complétés
  ✔ Le Deployment 'site' est disponible
  ✔ 'site' utilise la ConfigMap 'site-html'
  ✔ Le Service 'site' a des endpoints

Exercice 5 — Scaling & mises à jour
  ✔ 'site' a au moins 3 réplicas prêts
  ✔ Les pods 'site' sont répartis sur ≥ 2 nœuds
  ✔ L'image de 'site' est versionnée (nginx:x.y)
  ✔ Au moins une mise à jour a été déployée

Exercice 6 — Portainer & Mario
  ✔ Portainer est disponible
  ✔ Le Deployment 'mario' est disponible

Bonus
  ✘ A — L'Ingress 'site' répond sur le port 8081
  ✘ B — 'site' a une readinessProbe
  ✘ B — 'site' a des limites de ressources

Score : 15 / 18  (l'exercice 3 n'est pas vérifiable automatiquement)