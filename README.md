# Déploiement de Homepage

Ce projet permet de déployer [`gethomepage.dev`](https://gethomepage.dev/) sur Kubernetes avec deux approches :

- `k8s/` : manifests Kubernetes classiques, méthode par défaut
- `homepage-chart/` : chart Helm, méthode optimisée

## Prérequis

Avant de commencer, il faut disposer de :

- `kubectl` configuré sur un cluster Kubernetes accessible
- `helm` installé (si vous optez pour la méthode Helm)

Commandes de vérification :

```bash
kubectl config current-context
kubectl get nodes
```
Ces commandes permettent de vérifier que l'on est connecté au bon cluster Kubernetes.
Si vous optez pour l'option via Helm, exécutez également :

```bash
helm version
```

afin de vérifier que Helm est bien disponible sur la machine.

## 1. Installer Homepage avec les deux méthodes

### Méthode 1 : installation avec les manifests `k8s/`

Cette méthode applique directement les fichiers YAML présents dans le dossier `k8s/`.

Depuis la racine du projet :

```bash
kubectl apply -f k8s/
```

Cette commande crée l'ensemble des ressources Kubernetes décrites dans le dossier `k8s/`, comme le `Deployment`, le `Service`, le `ServiceAccount`, les droits RBAC et les `ConfigMap`.

### Méthode 2 : installation avec Helm (`homepage-chart/`)

Cette méthode utilise le chart Helm du projet pour générer et installer les ressources Kubernetes.

Depuis la racine du projet :

```bash
helm install homepage ./homepage-chart
```

Cette commande installe une release Helm nommée `homepage` à partir du chart local contenu dans `homepage-chart/`.

### Vérification du bon déploiement du service

Afin de vérifier que le service est bien déployé, vous pouvez exécuter :

```bash
kubectl get pods
kubectl get svc
kubectl rollout status deployment/homepage
```

Rôle de ces commandes :

- `kubectl get pods` affiche les pods créés pour vérifier qu'ils sont bien démarrés
- `kubectl get svc` affiche les services du cluster, notamment le service `homepage`
- `kubectl rollout status deployment/homepage` suit l'avancement du déploiement et confirme qu'il s'est terminé correctement

Dans les 2 méthodes, le service est exposé en `NodePort` sur le port `30007` :

```text
http://localhost:30007
```

Voici également quelques commandes utiles de supervision :

```bash
kubectl logs deployment/homepage
kubectl describe deployment homepage
kubectl describe pod <nom-du-pod>
```

Rôle de ces commandes :

- `kubectl logs deployment/homepage` affiche les journaux de l'application pour repérer une erreur de démarrage ou de configuration
- `kubectl describe deployment homepage` donne le détail du déploiement et de son état courant
- `kubectl describe pod <nom-du-pod>` permet d'inspecter un pod précis, utile pour comprendre pourquoi il ne démarre pas ou redémarre en boucle

et pour la méthode avec Helm, vous pouvez également exécuter : 

```bash
helm get values homepage
helm history homepage
```

Rôle de ces commandes :

- `helm get values homepage` affiche les valeurs utilisées par la release, ce qui permet de vérifier la configuration réellement appliquée
- `helm history homepage` liste les différentes révisions de la release, utile pour suivre les mises à jour et préparer un éventuel retour arrière

### Suppression du déploiement

Pour supprimer le déploiement instancié avec les manifests `k8s/`, il faut exécuter :

```bash
kubectl delete -f k8s/
```

Cette commande supprime toutes les ressources créées à partir des manifests du dossier `k8s/`.
   

Si vous avez instancié le service avec Helm, la commande de désinstallation est :

```bash
helm uninstall homepage
```

qui supprime la release Helm et les ressources Kubernetes associées.


## 2. Mise en place de l'auto-scaling horizontal (HPA)

Le projet inclut un `HorizontalPodAutoscaler` qui ajuste automatiquement le nombre de pods en fonction de la charge CPU. Le nombre de replicas varie entre **2** (minimum) et **4** (maximum), avec un seuil de déclenchement à **70% d'utilisation CPU**.

### Prérequis : Metrics Server

Le HPA nécessite que le **Metrics Server** soit installé dans le cluster. Sans lui, le HPA ne peut pas lire les métriques et reste inactif.

Vérifier s'il est présent :

```bash
kubectl get deployment metrics-server -n kube-system
```

Si la colonne `READY` affiche `0/1`, le Metrics Server est installé mais non fonctionnel. Sur les clusters locaux (kind, k3s, Minikube), il faut désactiver la vérification TLS :

```bash
# Télécharger le manifest officiel et ajouter --kubelet-insecure-tls
Invoke-WebRequest "https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml" -OutFile "$env:TEMP\metrics-server.yaml"
$content = Get-Content "$env:TEMP\metrics-server.yaml" -Raw
$content = $content -replace "- --metric-resolution=15s", "- --metric-resolution=15s`r`n        - --kubelet-insecure-tls"
$content | Out-File "$env:TEMP\metrics-server-fixed.yaml" -Encoding utf8
kubectl apply -f "$env:TEMP\metrics-server-fixed.yaml"
```

Vérifier que le Metrics Server est opérationnel :

```bash
kubectl top nodes
```

### Déploiement du HPA

**Avec Helm** : 

Le HPA est activé par défaut dans `homepage-chart/values.yaml` :

```bash
helm install homepage ./homepage-chart
```

**Avec les manifests `k8s/`** :

```bash
kubectl apply -f k8s/hpa.yaml
```

### Vérification

```bash
kubectl get hpa homepage
kubectl describe hpa homepage
```

La colonne `TARGETS` doit afficher une valeur réelle, par exemple `4%/70%`. Si elle affiche `<unknown>/70%`, attendre 1 à 2 minutes le temps que les métriques se stabilisent après le démarrage des pods.

### Configuration

Les paramètres du HPA sont centralisés dans `homepage-chart/values.yaml` :

```yaml
autoscaling:
  enabled: true
  minReplicas: 2
  maxReplicas: 4
  targetCPUUtilizationPercentage: 70
  behavior:
    scaleUp:
      stabilizationWindowSeconds: 0
      policies:
        - type: Percent
          value: 100
          periodSeconds: 60
    scaleDown:
      stabilizationWindowSeconds: 300
      policies:
        - type: Percent
          value: 50
          periodSeconds: 60
```

La fenêtre de stabilisation de 300 secondes en scale down évite les oscillations incessantes : le HPA attend 5 minutes avant de supprimer des pods.

## 3. Monitoring et observabilité

Le projet inclut un chart Helm dédié à l'observabilité, qui déploie une stack complète dans son propre namespace `monitoring`. Cette stack couvre les deux piliers utiles pour ce projet : **les métriques** (Prometheus + Grafana) et **les logs** (Loki + Promtail).

### Composants déployés

Le chart `monitoring-chart/` agrège trois dépendances Helm :

- **`kube-prometheus-stack`** — Prometheus, Grafana, AlertManager, kube-state-metrics et node-exporter, le tout préconfiguré avec des dashboards Kubernetes natifs
- **`loki`** — agrégation et stockage des logs en mode SingleBinary (suffisant pour un cluster mono-nœud)
- **`promtail`** — DaemonSet qui scrape les logs de tous les pods et les pousse vers Loki

Grafana est exposé en `NodePort` sur le port `30080`, et Loki est ajouté automatiquement comme datasource au démarrage de Grafana.

### Installation

Depuis la racine du projet :

```bash
# 1. Ajouter les dépôts Helm nécessaires
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update

# 2. Télécharger les sous-charts
helm dependency update ./monitoring-chart

# 3. Installer la stack dans le namespace monitoring
helm install monitoring ./monitoring-chart --namespace monitoring --create-namespace
```

Le namespace dédié permet d'isoler la stack d'observabilité du cycle de vie de l'application : un `helm uninstall homepage` ne touche pas à Grafana, et inversement.

### Accès à Grafana

```text
http://localhost:30080
```

Identifiants par défaut : `admin` / `admin` (configurable dans `monitoring-chart/values.yaml`).

Vérifier que les composants tournent :

```bash
kubectl get pods -n monitoring
```

Les pods attendus : `prometheus-monitoring-...`, `monitoring-grafana-...`, `alertmanager-...`, `loki-0`, `promtail-...`.

### Vérification de Loki

Dans Grafana, aller dans **Explore** → datasource **Loki**, puis lancer la requête suivante pour voir les logs de Homepage :

```logql
{namespace="default", pod=~"my-homepage.*"}
```

### Suppression

```bash
helm uninstall monitoring -n monitoring
```

## 4. Sauvegarde avec Velero et Garage



## 5. Analyse comparative : `k8s/` vs Helm

Pour avoir mis en place les 2 méthodes, on constate que l'approche `k8s/` est pratique pour comprendre Kubernetes et chacun de ces manifests de configuration, mais qu'elle devient vite lourde à maintenir dès qu'il faut faire évoluer cette configuration ou rejouer proprement des mises à jour.

L'approche Helm apporte une couche d'abstraction, grâce aux variables centralisées dans `homepage-chart/values.yaml`. Cela rend le déploiement plus réutilisable, plus configurable et plus propre à faire évoluer. Helm facilite aussi l'exploitation grâce à des commandes natives comme `helm upgrade`, `helm history`, `helm rollback` ou `helm uninstall`.

En pratique, le passage de `k8s/` à Helm apporte surtout trois bénéfices :

- une configuration centralisée et plus facile à personnaliser
- des mises à jour plus propres avec versionnement des releases
- une meilleure maintenabilité si le service doit évoluer ou être redéployé plusieurs fois

En résumé, la méthode `homepage-chart/` est préférable pour un déploiement plus industrialisé et plus simple à administrer dans le temps.

## 6. Schéma de l'infrastructure Kubernetes : 

![alt text](images/kubernetes-schema.png)