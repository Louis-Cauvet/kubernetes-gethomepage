# Homepage sur Kubernetes et Helm

Ce projet deploie [`gethomepage.dev`](https://gethomepage.dev/) avec deux approches :

- `k8s/` : manifests Kubernetes "bruts" (`kubectl apply`)
- `homepage-chart/` : chart Helm parametrable


## Structure du projet

```text
homepage/
|-- config/           # configuration source de Homepage
|-- k8s/              # manifests Kubernetes prets a appliquer
|-- homepage-chart/   # chart Helm
```

## Prerequis

Avant de demarrer, il faut avoir :

- un cluster Kubernetes accessible avec `kubectl`
- `kubectl` configuré sur le bon contexte
- `helm` installé si vous utilisez le chart Helm

Verification rapide :

```bash
kubectl config current-context
kubectl get nodes
helm version
```

## Option 1 : Demarrer Homepage avec Kubernetes (`k8s/`)

Cette methode applique directement les fichiers YAML presents dans `k8s/`.

### Ce que fait l'installation

La commande :

```bash
kubectl apply -f k8s/
```

crée les ressources suivantes :

- un `Deployment` nommé `homepage`
- un `Service` `NodePort` nommé `homepage`
- un `ServiceAccount` nommé `homepage`
- un `ClusterRole` et un `ClusterRoleBinding` pour permettre a Homepage de lire certaines ressources du cluster
- plusieurs `ConfigMap` montées dans `/app/config`

Les fichiers de configuration Homepage sont injectés depuis des `ConfigMap` :

- `bookmarks.yaml`
- `services.yaml`
- `settings.yaml`
- `widgets.yaml`
- `docker.yaml`
- `kubernetes.yaml`
- `proxmox.yaml`
- `custom.css`
- `custom.js`

### Installation

Depuis la racine du projet :

```bash
kubectl apply -f k8s/
```

Verifier ensuite :

```bash
kubectl get pods
kubectl get svc
kubectl get configmaps
kubectl get serviceaccount homepage
kubectl get clusterrole homepage
kubectl get clusterrolebinding homepage
```

Suivre le démarrage :

```bash
kubectl rollout status deployment/homepage
```

### Acces a l'application

Le service expose Homepage en `NodePort` :

- port applicatif : `3000`
- port expose sur le cluster : `30007`

Acces local :

```text
http://localhost:30007
```

Le port provient de [`k8s/service.yaml`]

### Commandes utiles avec `kubectl`

Voir les logs :

```bash
kubectl logs deployment/homepage
```

Décrire le déploiement :

```bash
kubectl describe deployment homepage
```

Décrire un pod :

```bash
kubectl describe pod <nom-du-pod>
```

Lister les `ConfigMap` :

```bash
kubectl get configmaps
```

Redémarrer l'application :

```bash
kubectl rollout restart deployment/homepage
```

### Mettre a jour la configuration en mode Kubernetes

En mode `k8s/`, la source de vérité pour le cluster est le contenu du dossier `k8s/`.
Modifier uniquement un fichier dans `config/` ne met rien a jour tant que le `ConfigMap` correspondant n'est pas réappliqué.

Exemple pour les favoris :

1. Modifier [`k8s/bookmarks-configmap.yaml`]
2. Réappliquer le manifest :

```bash
kubectl apply -f k8s/bookmarks-configmap.yaml
```

3. Redémarrer Homepage :

```bash
kubectl rollout restart deployment/homepage
kubectl rollout status deployment/homepage
```

Le même principe s'applique à :

- `k8s/services-configmap.yaml`
- `k8s/settings-configmap.yaml`
- `k8s/widgets-configmap.yaml`
- `k8s/docker-configmap.yaml`
- `k8s/kubernetes-configmap.yaml`
- `k8s/proxmox-configmap.yaml`
- `k8s/custom-css-configmap.yaml`
- `k8s/custom-js-configmap.yaml`


### Suppression du service

```bash
kubectl delete -f k8s/
```

## Option 2 : Demarrer Homepage avec Helm (`homepage-chart/`)

Cette méthode utilise le chart Helm du projet pour génerer puis installer les ressources Kubernetes.

### Ce que fait Helm ici

Helm :

- lit `homepage-chart/Chart.yaml`
- charge les valeurs par défaut depuis `homepage-chart/values.yaml`
- rend les templates du dossier `homepage-chart/templates/`
- installe les objets dans Kubernetes sous forme de release Helm


### Installation Helm

Depuis la racine du projet :

```bash
helm install homepage ./homepage-chart
```

Verifier l'installation :

```bash
helm list
kubectl get pods
kubectl get svc
kubectl get configmaps
```

Suivre le déploiement :

```bash
kubectl rollout status deployment/homepage
```

### Accès a l'application

Par defaut, le chart expose Homepage en `NodePort` sur `30007` :

```text
http://localhost:30007
```

### Personnalisation via `values.yaml`

Le fichier principal de configuration du chart est :

- [`homepage-chart/values.yaml`]

On peut y modifier notamment :

- l'image Docker
- le nombre de replicas
- le type de service
- le `nodePort`
- les ressources CPU / memoire
- tout le contenu des fichiers Homepage (`bookmarks`, `services`, `settings`, `widgets`, etc.)

### Mettre a jour une installation Helm

Si vous modifiez `homepage-chart/values.yaml` ou un template du chart :

```bash
helm upgrade homepage ./homepage-chart
```

Verifier ensuite :

```bash
helm status homepage
kubectl rollout status deployment/homepage
```

### Commandes Helm utiles

Voir le manifeste genéré sans installer :

```bash
helm template homepage ./homepage-chart
```

Installer une release :

```bash
helm install homepage ./homepage-chart
```

Afficher les valeurs utilisees :

```bash
helm get values homepage
```

Afficher l'état de la release :

```bash
helm status homepage
```

Lister les releases :

```bash
helm list
```

Voir l'historique des révisions :

```bash
helm history homepage
```

Revenir a une révision précedente :

```bash
helm rollback homepage <revision>
```

Supprimer la release :

```bash
helm uninstall homepage
```

### Ce qu'il faut savoir sur le chart actuel

Le chart utilise le nom de release Helm pour nommer la plupart des ressources.
Avec cette commande :

```bash
helm install homepage ./homepage-chart
```

vous obtiendrez notamment :

- un `Deployment` `homepage`
- un `Service` `homepage`
- des `ConfigMap` comme `homepage-bookmarks`, `homepage-services`, `homepage-settings`


## Quelle methode choisir ?

Utiliser `k8s/` si vous voulez :

- appliquer des manifests simples et explicites
- voir directement les ressources YAML finales
- travailler sans Helm

Utiliser `homepage-chart/` si vous voulez :

- parametrer facilement le deploiement
- gerer les mises a jour avec `helm upgrade`
- beneficier de `helm history`, `helm rollback` et `helm template`

