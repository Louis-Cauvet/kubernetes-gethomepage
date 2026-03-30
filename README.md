# 🚀 Homepage Kubernetes Setup

Ce projet déploie **Homepage** sur Kubernetes avec une configuration modulaire via ConfigMaps.

---

## 📦 📁 Structure du projet

```
project/
├── config/     # fichiers YAML locaux (source)
├── k8s/        # manifests Kubernetes
```

---

## ▶️ Lancer l'application

### 1. Appliquer toute la configuration

```bash
kubectl apply -f k8s/
```

---

### 2. Vérifier que tout est lancé

```bash
kubectl get pods
kubectl get svc
```

---

### 3. Accéder à l'application

👉 Ouvrir dans le navigateur :

```
http://localhost:30007
```

---

## 🔄 Mettre à jour la configuration

### 1. Modifier un fichier (ex: `config/bookmarks.yaml`)

---

### 2. Mettre à jour le ConfigMap correspondant

```bash
kubectl apply -f k8s/bookmarks-configmap.yaml
```

👉 Faire pareil pour :

* `services-configmap.yaml`
* `settings-configmap.yaml`
* etc.

---

### 3. Redémarrer le Deployment (obligatoire)

```bash
kubectl rollout restart deployment homepage
```

---

## 🔁 Redémarrer l'application

```bash
kubectl rollout restart deployment homepage
```

---

## 🧪 Debug

### Voir les logs

```bash
kubectl logs deployment/homepage
```

---

### Décrire le pod

```bash
kubectl describe pod <nom-du-pod>
```

---

### Voir les ConfigMaps

```bash
kubectl get configmaps
kubectl describe configmap homepage-bookmarks
```

---

## 🛑 Supprimer l'application

```bash
kubectl delete -f k8s/
```

---

## 💡 Commandes utiles

### Voir les pods

```bash
kubectl get pods
```

### Voir les services

```bash
kubectl get svc
```

### Voir les ressources utilisées

```bash
kubectl top pods
```

---

## ⚠️ Notes importantes

* Toute modification de config nécessite :

  1. `kubectl apply`
  2. `kubectl rollout restart`

* Les fichiers sont montés avec `subPath` → pas de reload automatique

---

## 🎯 Accès rapide

| Élément  | URL                    |
| -------- | ---------------------- |
| Homepage | http://localhost:30007 |


