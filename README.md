# 🚀 Projet GitOps avec ArgoCD et Kubernetes

Déploiement automatique d'une application web sur un cluster Kubernetes local, déclenché par un simple `git push`, **sans intervention manuelle** sur le cluster.

---

## 🎯 Objectif du projet

Mettre en place un flux **GitOps** : le dépôt Git devient la « source de vérité » du déploiement. Dès qu'une modification est poussée sur Git (changement de page, de version ou du nombre de réplicas), l'application se met à jour automatiquement dans le cluster via ArgoCD.

```
git push  →  GitHub  →  ArgoCD détecte le changement  →  cluster Kubernetes mis à jour ✅
```

---

## 📚 Les concepts clés

### 🐳 Docker — mettre l'application dans une boîte

Un **conteneur** Docker est une boîte qui embarque une application avec tout ce qu'il lui faut pour tourner (code, dépendances, serveur web). L'avantage : ça fonctionne de façon identique partout.

Dans ce projet, on construit une **image Docker** basée sur nginx contenant notre page HTML.

### ☸️ Kubernetes — orchestrer les conteneurs

Docker met une application dans une boîte ; **Kubernetes gère un parc de boîtes** à grande échelle. Il :

- redémarre automatiquement un conteneur qui plante ;
- lance plusieurs copies (réplicas) pour encaisser la charge ;
- répartit le trafic entre ces copies ;
- maintient en permanence l'état désiré (si on veut 3 copies et qu'il n'en reste que 2, il en recrée une).

### 🟦 Minikube — un Kubernetes local

**Minikube** est un mini-cluster Kubernetes qui tourne sur la machine de développement. Il permet de s'entraîner et de tester en local, sans payer de serveurs cloud. C'est lui qui héberge toute l'application : une fois démarré, il travaille en arrière-plan.

> Alternative possible : **K3s**, une version allégée de Kubernetes. L'énoncé propose « Minikube/K3s » : les deux conviennent. Minikube a été choisi ici pour son intégration native avec Docker et WSL2.

### 🐙 ArgoCD — le moteur GitOps

**ArgoCD** est installé dans le cluster et surveille le dépôt Git en boucle. Son métier :

```
1. Lire l'état désiré dans Git (les fichiers YAML)
2. Comparer avec l'état réel du cluster
3. Si écart  →  aligner le cluster sur Git automatiquement
```

ArgoCD fournit une interface web qui affiche l'état des applications :
- 🟢 **Synced** : cluster = Git
- 🟡 **OutOfSync** : un écart existe, synchronisation nécessaire
- ❤️ **Healthy** : les pods tournent correctement

### 🔄 GitOps — Pull plutôt que Push

Il existe deux façons d'automatiser un déploiement :

| | Modèle **Push** (CI/CD classique) | Modèle **Pull** (GitOps) |
|---|---|---|
| Qui agit | GitHub Actions pousse vers le serveur | ArgoCD tire depuis Git |
| Le cluster | reçoit passivement | va chercher activement |
| Sécurité | le cluster expose des accès | le cluster n'expose rien |

Dans ce projet on utilise le modèle **Pull** : Git ne fait que stocker et historiser les fichiers ; c'est ArgoCD, installé dans le cluster, qui tire les changements et les applique. L'historique Git devient ainsi l'historique des déploiements (rollback facile via `git revert`).

---

## 🏗️ Architecture

```
┌─────────────┐   git push   ┌──────────┐   surveille   ┌─────────┐   déploie   ┌──────────┐
│  Mon poste  │ ───────────► │  GitHub  │ ◄──────────── │ ArgoCD  │ ──────────► │ Minikube │
│  (YAML)     │              │ (vérité) │               │ (robot) │             │ (cluster)│
└─────────────┘              └──────────┘               └─────────┘             └──────────┘
                                                              ▲
                                                              │ tire l'image
                                                        ┌───────────┐
                                                        │ Docker Hub│
                                                        │  (image)  │
                                                        └───────────┘
```

---

## 📁 Structure des dépôts

Le projet repose sur **deux parties distinctes** (bonne pratique GitOps : séparer le code de l'application de sa configuration de déploiement).

### Dépôt de l'application (le site)

Contient la page web et sa recette de construction Docker.

```
mon-site/
├── index.html        # la page web
└── Dockerfile        # recette : nginx + la page
```

### Dépôt GitOps (la configuration) — surveillé par ArgoCD

```
gitops-demo/
├── manifests/
│   ├── deployment.yaml    # définit l'app : image + nombre de réplicas
│   └── services.yaml      # expose l'app dans le cluster
├── application.yaml       # configuration ArgoCD (appliquée en local)
└── README.md
```

Le lien entre les deux : la ligne `image:` du `deployment.yaml` pointe vers l'image construite et publiée sur Docker Hub.

---

## 🔧 Le rôle de chaque fichier

| Fichier | S'adresse à | Rôle |
|---|---|---|
| `Dockerfile` | Docker | Construire l'image (nginx + page HTML) |
| `deployment.yaml` | Kubernetes | Lancer l'app : quelle image, combien de réplicas |
| `services.yaml` | Kubernetes | Rendre l'app joignable dans le cluster |
| `application.yaml` | ArgoCD | Indiquer quel dépôt Git surveiller |

---

## ⚙️ Mise en place — étape par étape

### 1. Démarrer le cluster et installer ArgoCD

```bash
# démarrer Minikube
minikube start --driver=docker

# installer ArgoCD
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
kubectl wait --for=condition=available --timeout=300s deployment/argocd-server -n argocd
```

### 2. Accéder à l'interface ArgoCD

```bash
# récupérer le mot de passe admin
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d; echo

# ouvrir l'interface (laisser le terminal ouvert)
kubectl port-forward svc/argocd-server -n argocd 8080:443
```

➡️ Interface sur **https://localhost:8080** — login `admin` + le mot de passe ci-dessus.

### 3. Construire et publier l'image Docker

Dans le dossier de l'application :

```bash
docker login
docker build -t TON_PSEUDO_DOCKER/gitops-site:1.0 .
docker push TON_PSEUDO_DOCKER/gitops-site:1.0
```

> ⚠️ L'image doit être en **public** sur Docker Hub pour que le cluster puisse la télécharger.

### 4. Connecter ArgoCD au dépôt GitOps

Via l'interface (bouton **NEW APP**) ou via le fichier `application.yaml`, avec ces paramètres :

| Champ | Valeur                                       |
|---|----------------------------------------------|
| Repository URL | `https://github.com/UIshfaq/gitops-demo.git` |
| Revision | `HEAD`                                       |
| Path | `manifests`                                  |
| Cluster | `https://kubernetes.default.svc`             |
| Namespace | `default`                                    |

ArgoCD lit alors le dossier `manifests/`, applique les fichiers, et l'application apparaît dans l'interface.

### 5. Voir l'application

```bash
minikube service demo-app -n default --url
```

➡️ Ouvrir l'URL affichée (`http://127.0.0.1:XXXXX`) dans le navigateur. La page s'affiche, servie depuis l'image Docker.

> Le service est de type `ClusterIP` (accès interne au cluster). `minikube service` crée un tunnel temporaire pour y accéder depuis le navigateur en local. **Laisser le terminal ouvert** tant qu'on consulte la page.

---

## 🔌 Relancer le projet après extinction du PC

Au redémarrage de la machine, rien ne tourne. Relancer dans cet ordre :

| Étape | Action / commande | Terminal à garder ouvert |
|---|---|---|
| 1 | Lancer **Docker Desktop** et attendre qu'il soit prêt | — |
| 2 | Ouvrir **Ubuntu** (WSL2) | — |
| 3 | `minikube start --driver=docker` | non |
| 4 | `minikube status` (doit afficher *Running*) | non |
| 5 | `kubectl port-forward svc/argocd-server -n argocd 8080:443` | ✅ oui |
| 6 | `minikube service demo-app -n default --url` (dans un 2e terminal) | ✅ oui |

Les pods et ArgoCD reviennent automatiquement après `minikube start` : aucune réinstallation n'est nécessaire.

> 💡 Le jour de la soutenance, effectuer ces étapes **en avance** : le démarrage de Minikube prend 1 à 2 minutes. Prévoir **deux terminaux** ouverts (interface ArgoCD + accès à la page).

**Mettre en pause proprement à la fin :**

```bash
minikube stop      # libère les ressources, l'état est conservé
```

---

## 🔄 Comment on travaille au quotidien (sans workflow CI/CD)

Le déploiement de l'image est géré **manuellement**, avec des **tags versionnés** (1.0, 2.0, 3.0…). C'est volontaire : chaque version est traçable, et ArgoCD redéploie automatiquement dès que le tag change dans le `deployment.yaml`.

> Le tag `latest` a été écarté : il ne change jamais de nom, donc le cluster ne détecte pas qu'une nouvelle image existe et ne redéploie pas. Versionner (1.0, 2.0…) garantit un déploiement automatique et traçable.

### Modifier la page web

```bash
# 1. construire la nouvelle version (incrémenter le numéro)
docker build -t TON_PSEUDO_DOCKER/gitops-site:2.0 .
docker push TON_PSEUDO_DOCKER/gitops-site:2.0

# 2. mettre à jour le tag dans gitops-demo/manifests/deployment.yaml
#    image: TON_PSEUDO_DOCKER/gitops-site:2.0

# 3. pousser la modification
git commit -am "page version 2.0"
git push
```

➡️ ArgoCD détecte le nouveau tag et redéploie automatiquement.

### Modifier le nombre de réplicas

Aucun Docker nécessaire, uniquement Git :

```bash
sed -i 's/replicas: 2/replicas: 3/' manifests/deployment.yaml
git commit -am "scale to 3 replicas"
git push
```

➡️ ArgoCD aligne le cluster : 3 pods apparaissent automatiquement.

### Forcer la synchronisation (sans attendre)

ArgoCD vérifie Git toutes les ~3 minutes. Pour aller plus vite : dans l'interface, **REFRESH** → **SYNC** → **SYNCHRONIZE**.

---

## ✅ Tests de validation du PoC

### Test principal — Mise à jour automatique de la page web

C'est la démonstration centrale du projet : une modification de la page sur Git se répercute automatiquement dans le navigateur, sans aucune action sur le cluster.

```bash
# 1. modifier le contenu de index.html (ex: "Version 1" → "Version 2")

# 2. construire et publier la nouvelle version
docker build -t TON_PSEUDO_DOCKER/gitops-site:2.0 .
docker push TON_PSEUDO_DOCKER/gitops-site:2.0

# 3. mettre à jour le tag dans deployment.yaml puis pousser
git commit -am "page version 2.0"
git push
```

➡️ Après synchronisation (automatique ou via REFRESH), recharger la page dans le navigateur (**Ctrl+Shift+R**) : la nouvelle version s'affiche. Le déploiement s'est fait uniquement via Git, sans toucher au cluster.

> Pour visualiser : `minikube service demo-app -n default --url`

### Test secondaire — Scaling automatique

Modifier le nombre de réplicas dans Git (ex : de 2 à 4) :

```bash
sed -i 's/replicas: 2/replicas: 4/' manifests/deployment.yaml
git commit -am "scale to 4 replicas"
git push
kubectl get pods -n default
```

➡️ 4 pods apparaissent automatiquement, sans intervention directe sur le cluster.

### Test bonus — Self-heal (auto-réparation)

Supprimer un pod à la main :

```bash
kubectl delete pod <nom-du-pod> -n default
```

➡️ ArgoCD le recrée automatiquement : le cluster reste toujours aligné sur Git.

---

## 🎤 Synthèse pour la soutenance

Ce projet démontre une chaîne de déploiement continu en mode GitOps :

- **Kubernetes (Minikube)** héberge l'application en local.
- **Docker / Docker Hub** conteneurise et stocke l'image versionnée.
- **Git (GitHub)** est la source de vérité et l'historique des déploiements.
- **ArgoCD** surveille Git et synchronise le cluster automatiquement.

> *« Contrairement à une CI/CD classique où GitHub pousse vers le serveur, ici ArgoCD tire depuis Git. Le cluster n'expose aucun accès vers l'extérieur, et l'historique Git devient l'historique des déploiements. Pour passer en production, il suffirait de remplacer Minikube par un cluster managé (EKS, Scaleway Kapsule…) et d'exposer l'application via un Ingress : toute la logique GitOps reste inchangée. »*

---

## 🧹 Nettoyage

```bash
kubectl delete -f application.yaml
minikube stop      # met en pause
minikube delete    # supprime le cluster
```