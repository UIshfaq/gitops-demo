# 🚀 GitOps avec ArgoCD et Kubernetes (Minikube)

PoC : déployer automatiquement une application dès qu'un développeur pousse du code sur Git, **sans intervention manuelle** sur le cluster.

## 🎯 Le principe

```
git push (modif YAML)  →  GitHub  →  ArgoCD détecte le diff  →  sync auto  →  Minikube à jour ✅
```

ArgoCD compare en boucle l'état déclaré dans Git (la « source de vérité ») avec l'état réel du cluster. Dès qu'il détecte un écart, il aligne le cluster sur Git automatiquement.

## 🧱 Les 3 briques

| Brique | Rôle |
|---|---|
| 🐳 Docker | Met l'application dans un conteneur |
| ☸️ Kubernetes (Minikube) | Orchestre les conteneurs en local |
| 🐙 ArgoCD | Surveille le repo Git et synchronise le cluster |