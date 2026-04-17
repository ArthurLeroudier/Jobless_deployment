# Jobless — Dépôt GitOps

> Dépôt de déploiement continu du projet [JobLess](https://github.com/anhlinhpiketty/Mise_en_production_DS) sur SSP Cloud via ArgoCD.

---

## Principe

Ce dépôt est la **source de vérité** pour l'état désiré de l'infrastructure Kubernetes de JobLess. ArgoCD surveille ce dépôt en continu : tout changement sur la branche `main` est automatiquement synchronisé sur le cluster SSP Cloud.

```
GitHub Actions (repo principal)          Ce dépôt (GitOps)
        │                                       │
  Build & push                           Source of truth
  image Docker            ──────►        manifestes K8s
        │                                       │
        ▼                                       ▼
   Docker Hub                           ArgoCD (SSP Cloud)
                                               │
                                    selfHeal: true
                                               │
                                        Cluster K8s
                                    ┌──────────────────┐
                                    │  Backend (API)   │  :8000
                                    │  Frontend (UI)   │  :8501
                                    └──────────────────┘
```

---

## Structure

```
Jobless_deployment/
├── application_backend.yaml     # Application ArgoCD — backend
├── application_frontend.yaml    # Application ArgoCD — frontend
│
├── deployment/                  # Manifestes Kubernetes — Backend (FastAPI)
│   ├── deployment.yaml          # Deployment : image arthurleroudier/jobless
│   ├── service.yaml             # Service ClusterIP (port 80 → 8000)
│   └── ingress.yaml             # Ingress (à compléter)
│
└── deployment_front/            # Manifestes Kubernetes — Frontend (Streamlit)
    ├── deployment.yaml          # Deployment : image arthurleroudier/jobless_front
    ├── service.yaml             # Service LoadBalancer (port 8 → 8501)
    └── ingress.yaml             # Ingress NGINX avec TLS et CORS
```

---

## Applications ArgoCD

Deux objets `Application` ArgoCD pilotent le déploiement :

| Application | Chemin surveillé | Namespace cible |
|---|---|---|
| `jobless-backend` | `deployment/` | `user-aleroudier` |
| `jobless-frontend` | `deployment_front/` | `user-aleroudier` |

Les deux applications ont `selfHeal: true` : si l'état du cluster dérive des manifestes, ArgoCD corrige automatiquement.

Pour appliquer une application ArgoCD :

```bash
kubectl apply -f application_backend.yaml
kubectl apply -f application_frontend.yaml
```

---

## Déploiements Kubernetes

### Backend (FastAPI)

- **Image** : `arthurleroudier/jobless:v1.0`
- **Port** : `8000`
- **Replicas** : 1
- **Secret requis** : `api-jeton` (clé `API_KEY`)
- **Exposition** : Service ClusterIP (port 80 → 8000)

### Frontend (Streamlit)

- **Image** : `arthurleroudier/jobless_front:v1.0` — `imagePullPolicy: Always`
- **Port** : `8501`
- **Replicas** : 1
- **Secret requis** : `front-jeton` (clé `FRONT_KEY`)
- **Exposition** : Service LoadBalancer + Ingress NGINX
- **URL publique** : `https://jobless-website.lab.sspcloud.fr`

---

## Secrets Kubernetes requis

Avant tout déploiement, créer les secrets dans le namespace cible :

```bash
# Clé API LLM (backend)
kubectl create secret generic api-jeton \
  --from-literal=API_KEY='votre_clé_api_llm' \
  -n user-aleroudier

# Clé frontend (si applicable)
kubectl create secret generic front-jeton \
  --from-literal=FRONT_KEY='votre_clé_front' \
  -n user-aleroudier
```

---

## Ingress frontend

L'Ingress frontend est configuré avec :

- **Classe** : `nginx`
- **TLS** activé sur `jobless-website.lab.sspcloud.fr`
- **CORS** ouvert (`*`) avec les méthodes `GET, POST, PUT, DELETE, OPTIONS`

---

## Flux de mise à jour

1. Un développeur pousse du code sur le [dépôt principal](https://github.com/anhlinhpiketty/Mise_en_production_DS)
2. GitHub Actions build et pousse la nouvelle image Docker sur Docker Hub (tag `v*` ou `main`)
3. Le manifeste `deployment.yaml` est mis à jour manuellement (ou par automation) avec le nouveau tag d'image
4. Le push sur `main` de ce dépôt déclenche la synchronisation ArgoCD
5. ArgoCD applique les changements sur le cluster SSP Cloud

---

## Liens

- [Dépôt principal — code source](https://github.com/anhlinhpiketty/Mise_en_production_DS)
- [Application en ligne](https://jobless-website.lab.sspcloud.fr/)
- [Documentation du projet](https://anhlinhpiketty.github.io/Mise_en_production_DS/)
