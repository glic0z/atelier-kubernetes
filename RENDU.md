# Rendu — Atelier Kubernetes

> Complétez ce fichier au fil de l'atelier, puis faites un commit et un push (voir la section « Rendu » du README).

**Nom / Prénom :** Nasser (GitHub : glic0z)
**Groupe :**

---

## Exercice 1 — Cluster

1. Combien de nœuds affiche `kubectl get nodes` ? Quel est le rôle de chacun ?
   > 3 nœuds : `k3d-lab-server-0` (control-plane : API Server, scheduler, etcd) et `k3d-lab-agent-0` / `k3d-lab-agent-1` (workers, qui exécutent les pods).
2. Que montre `docker ps` ? Qu'est-ce qu'un « nœud » dans k3d, concrètement ?
   > `docker ps` montre un conteneur par nœud (3 `rancher/k3s`) + le load balancer `k3d-lab-serverlb` + `k3d-lab-tools`. Un nœud k3d est un conteneur Docker qui fait tourner k3s. `kubectl get pods -A` montre déjà dans `kube-system` : coredns, traefik, local-path-provisioner, metrics-server.

## Exercice 2 — Premier déploiement

3. Quels objets Kubernetes ont été créés par `kubectl create deployment` ? (indice : `kubectl get all`)
   > Un Deployment `web`, un ReplicaSet `web-<hash>` et un Pod `web-<hash>-<id>` ; `kubectl expose` a ajouté un Service `web` (ClusterIP).
4. Vous avez supprimé le pod `web-...` : que s'est-il passé, et pourquoi ?
   > Le ReplicaSet a recréé immédiatement un nouveau pod (nom différent : `web-68d995574f-rc4ql` -> `web-68d995574f-hrbkf`) : la boucle de réconciliation ramène l'état réel (1 pod) à l'état désiré (replicas=1).

## Exercice 3 — Modifier un conteneur en direct

5. Après suppression du pod, votre message « Bienvenue... » a-t-il survécu ? Pourquoi ?
   > Non : le système de fichiers d'un conteneur est éphémère, le nouveau pod repart de l'image nginx d'origine (page par défaut revenue, vérifié avec `exec cat`).
6. Pourquoi le `port-forward` a-t-il cessé de fonctionner après la suppression du pod ?
   > Un `port-forward` vers un Service s'attache à un seul pod ; quand ce pod est supprimé, le tunnel est coupé.

## Exercice 4 — Déclaratif

7. Quelle différence entre `kubectl create` et `kubectl apply` ?
   > `create` est impératif et échoue si la ressource existe ; `apply` est déclaratif : il crée ou met à jour la ressource pour qu'elle corresponde au fichier (idempotent, versionnable dans Git).
8. À quoi sert le `selector` d'un Service ? Que se passe-t-il s'il ne correspond à aucun pod ?
   > Le `selector` désigne les pods (par label) qui reçoivent le trafic. S'il ne correspond à aucun pod, le Service n'a pas d'endpoints et les requêtes échouent.

## Exercice 5 — Scaling & mises à jour

9. Sur quels nœuds se trouvent vos 3 réplicas ? Collez la sortie de `kubectl get pods -l app=site -o wide`.
   ```
NAME                   READY   STATUS    NODE
site-75775484b-7ph4m   1/1     Running   k3d-lab-agent-0
site-75775484b-frmgb   1/1     Running   k3d-lab-server-0
site-75775484b-n5gd6   1/1     Running   k3d-lab-agent-1
   ```
10. Avec l'image `nginx:9.99`, quel statut avait le pod ? Quelle commande vous a donné la cause ?
    > `ErrImagePull` puis `ImagePullBackOff`. `kubectl describe pod` (section Events) : `docker.io/library/nginx:9.99: not found`.
11. Pendant la panne, le site restait-il accessible ? Pourquoi ?
    > Oui : le rolling update garde les anciens pods tant que le nouveau n'est pas Ready (maxUnavailable 25 %), donc les 3 anciens pods continuaient à servir le trafic.

## Exercice 6 — Portainer & Mario

12. URL publique de votre Mario :
    > https://silver-dollop-7xr55655469hr44p-8585.app.github.dev/
13. Retrouvez le Deployment `mario` en ligne de commande. Portainer et `kubectl` parlent-ils au même composant ? Lequel ?
    > Oui, `kubectl get deploy mario -o yaml` le montre : c'est un Deployment standard. Portainer (qui tourne dans le cluster avec un ServiceAccount) et kubectl appellent le même composant, l'API Server de Kubernetes. La seule différence : Portainer ajoute des labels `io.portainer.kubernetes.application.*` (owner: admin, stackid: 1) et n'ajoute pas l'annotation `last-applied-configuration` de `kubectl apply`. Mario a été déployé via l'éditeur web de Portainer (compte admin créé avec le setup token des logs).

## Bonus (facultatif)

14. Bonus réalisés et remarques :
    > A (Ingress Traefik sur le port 8081, HTTP 200 avec la page), B (readinessProbe + requests/limits), C (drain/uncordon de `k3d-lab-agent-0` : les pods sont replanifiés sur les autres nœuds et ne reviennent pas tout seuls après `uncordon`, le scheduler ne déplace pas des pods déjà placés).

---

**Score obtenu avec `./scripts/check.sh` :** 18 / 18
