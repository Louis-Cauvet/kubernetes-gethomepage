# Homepage Kubernetes Setup

Ce projet deploie l'application `ghcr.io/gethomepage/homepage:latest` sur Kubernetes avec une configuration fournie par plusieurs `ConfigMap`.

## Structure du projet

```text
homepage/
|-- config/   # fichiers de configuration locaux
|-- k8s/      # manifests Kubernetes appliques au cluster
`-- README.md
```

## Demarrage

Depuis la racine du projet :

```bash
kubectl apply -f k8s/
```

Verifier ensuite que les ressources sont bien creees :

```bash
kubectl get pods
kubectl get svc
kubectl get serviceaccount homepage
```

## Acces a l'application

Le service Kubernetes est de type `NodePort` sur le port `30007`.

```text
http://localhost:30007
```

Le port expose provient de [`k8s/service.yaml`](./k8s/service.yaml).

## Ressources deployees

Le dossier `k8s/` contient notamment :

- un `Deployment` nomme `homepage`
- un `Service` `NodePort` nomme `homepage`
- un `ServiceAccount` nomme `homepage`
- un `ClusterRole` et un `ClusterRoleBinding` pour permettre a Homepage de lire des ressources Kubernetes
- plusieurs `ConfigMap` montees dans `/app/config`

## Mettre a jour la configuration

La source appliquee au cluster est actuellement dans les manifests du dossier `k8s/`.
Autrement dit, modifier uniquement un fichier dans `config/` ne met pas a jour le cluster tant que le `ConfigMap` correspondant dans `k8s/` n'est pas reapplique.

Exemple pour les favoris :

1. Modifier [`k8s/bookmarks-configmap.yaml`](./k8s/bookmarks-configmap.yaml)
2. Reappliquer la configuration :

```bash
kubectl apply -f k8s/bookmarks-configmap.yaml
```

3. Redemarrer le deployment :

```bash
kubectl rollout restart deployment homepage
```

Tu peux faire la meme chose avec les autres fichiers :

- `k8s/services-configmap.yaml`
- `k8s/settings-configmap.yaml`
- `k8s/widgets-configmap.yaml`
- `k8s/docker-configmap.yaml`
- `k8s/kubernetes-configmap.yaml`
- `k8s/proxmox-configmap.yaml`
- `k8s/custom-css-configmap.yaml`
- `k8s/custom-js-configmap.yaml`

## Pourquoi un redemarrage est necessaire

Les fichiers de configuration sont montes avec `subPath` dans le conteneur via le `Deployment`.
Dans ce mode, les mises a jour de `ConfigMap` ne sont pas rechargees automatiquement par le pod existant.

## Commandes utiles

Voir l'etat du deploiement :

```bash
kubectl get all
```

Voir les logs :

```bash
kubectl logs deployment/homepage
```

Redemarrer l'application :

```bash
kubectl rollout restart deployment homepage
```

Suivre le rollout :

```bash
kubectl rollout status deployment homepage
```

Decrire un pod :

```bash
kubectl describe pod <nom-du-pod>
```

Lister les `ConfigMap` :

```bash
kubectl get configmaps
```

Afficher une `ConfigMap` :

```bash
kubectl describe configmap homepage-bookmarks
```

Voir l'utilisation CPU/memoire si `metrics-server` est installe :

```bash
kubectl top pods
```

## Suppression

Pour supprimer l'application :

```bash
kubectl delete -f k8s/
```

## Notes importantes

- Les manifests utilisent le namespace `default`.
- Le `ServiceAccount` et le `ClusterRoleBinding` pointent aussi vers `default`.
- La variable `HOMEPAGE_ALLOWED_HOSTS` autorise `localhost:30007` et l'IP du pod.
- Si `http://localhost:30007` ne repond pas, verifie que ton cluster expose bien les `NodePort` sur ta machine locale.
