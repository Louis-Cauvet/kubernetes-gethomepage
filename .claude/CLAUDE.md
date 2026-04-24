# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Kubernetes deployment project for [gethomepage.dev](https://gethomepage.dev) — a personal dashboard application. The repo offers two deployment strategies for the same app: raw Kubernetes manifests and a Helm chart.

## Deployment Commands

### Option 1 — Raw Kubernetes (`k8s/`)

```bash
kubectl apply -f k8s/                                   # Full deploy
kubectl delete -f k8s/                                  # Uninstall
kubectl apply -f k8s/<name>-configmap.yaml              # Update a single config
kubectl rollout restart deployment/homepage             # Reload after config change
```

### Option 2 — Helm chart (`homepage-chart/`)

```bash
helm install homepage ./homepage-chart                  # First install
helm upgrade homepage ./homepage-chart                  # Apply changes
helm uninstall homepage                                 # Remove
helm template homepage ./homepage-chart                 # Preview rendered manifests
helm rollback homepage <revision>                       # Roll back
```

After any ConfigMap change, the pod must be restarted:
```bash
kubectl rollout restart deployment/homepage
```

Access the dashboard at `http://<node-ip>:30007`.

## Architecture

There are two deployment artifacts that must stay in sync — they deploy the same application with the same configuration:

| Path | Purpose |
|------|---------|
| `config/` | Source-of-truth for all Homepage configuration (YAML + CSS/JS). Edit these first. |
| `k8s/` | Concrete Kubernetes manifests — one ConfigMap per config file, plus Deployment/Service/RBAC. |
| `homepage-chart/` | Helm chart wrapping the same resources with `values.yaml`-driven parameterization. |

**Config propagation:**
- In the Helm workflow, `values.yaml` embeds the config inline; `helm upgrade` syncs everything.
- In the raw-k8s workflow, changes to `config/` files must be manually reflected in the corresponding `k8s/*-configmap.yaml` before re-applying.

## Key Configuration Files

| File | Content |
|------|---------|
| `config/settings.yaml` | Dashboard title, theme, color scheme |
| `config/bookmarks.yaml` | Browser bookmark links by category |
| `config/services.yaml` | Service cards (icons, URLs, descriptions) |
| `config/widgets.yaml` | Info widgets (weather, clock, search bar) |
| `config/kubernetes.yaml` | Kubernetes cluster integration |
| `config/proxmox.yaml` | Proxmox integration |
| `config/custom.css` | Gradient background override |

## Kubernetes Resources

The deployment creates:
- `Deployment` — 1 replica, image `ghcr.io/gethomepage/homepage:latest`, non-root (UID 1000), NodePort 30007 → container 3000
- `ServiceAccount` + `ClusterRole`/`ClusterRoleBinding` — read-only access to namespaces, pods, nodes, ingresses, metrics
- 10 `ConfigMap` objects — one per configuration file, mounted at `/app/config/`

Resource limits: 300m CPU / 256Mi memory; requests 50m / 64Mi.

## Helm Chart Structure

`homepage-chart/values.yaml` controls replicas, image tag, service type/port, and embeds all config file contents as multiline strings. The templates in `homepage-chart/templates/` mirror the `k8s/` manifests but use Go template variables sourced from `values.yaml`.
