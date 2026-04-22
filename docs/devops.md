# DevOps & Infrastructure — Questions & Réponses Entretien

---

## I. Quelle différence entre `git merge` et `git rebase` ?

**Question** : "Quelle est la différence entre `git merge` et `git rebase` ? Quand utilise-t-on l'un plutôt que l'autre en équipe ?"

**Réponse** :

Les deux intègrent les changements d'une branche dans une autre, mais avec des historiques différents.

**`git merge`** : crée un nouveau commit de fusion qui a deux parents — l'historique des deux branches est préservé tel quel. L'historique est non-linéaire mais honnête : il reflète exactement ce qui s'est passé.

```
Avant merge :
  main:    A --- B --- C
                  \
  feature:         D --- E

Après git merge feature → main :
  main:    A --- B --- C --- M   (M = merge commit, 2 parents)
                  \         /
  feature:         D --- E
```

**`git rebase`** : réécrit l'historique de la branche feature en rejouant ses commits au-dessus de la branche cible. L'historique est linéaire et propre, mais les commits sont réécrits (nouveaux SHA).

```
Avant rebase :
  main:    A --- B --- C
                  \
  feature:         D --- E

Après git rebase main (sur feature) :
  main:    A --- B --- C
                        \
  feature:               D' --- E'  (D' et E' sont de nouveaux commits)
```

**Règle d'or** : **ne jamais rebaser une branche partagée** (main, develop). Le rebase réécrit les SHA — si d'autres développeurs ont basé leur travail sur les anciens commits, leur historique diverge et `git push --force` devient nécessaire, ce qui peut écraser le travail des collègues.

**En pratique** :
| Situation | Commande |
|---|---|
| Intégrer une PR dans main | `merge` (préserve l'historique) |
| Nettoyer une branche feature avant PR | `rebase -i` (squash, reword) |
| Mettre à jour sa branche avec main | `rebase main` (historique linéaire) |
| Branche partagée avec l'équipe | `merge` uniquement |

---

## II. Qu'est-ce que le multi-stage build Docker et pourquoi l'utiliser ?

**Question** : "Qu'est-ce qu'un multi-stage build dans un Dockerfile ? Quel est l'avantage en termes de taille d'image et de sécurité ?"

**Réponse** :

Un **multi-stage build** permet d'utiliser plusieurs instructions `FROM` dans un seul Dockerfile. Chaque stage est un environnement indépendant — on peut copier uniquement les artefacts nécessaires du stage précédent, sans embarquer les outils de build dans l'image finale.

**Problème sans multi-stage** : une image Python avec les dépendances de build (gcc, git, pip, fichiers de cache) pèse facilement 1-2 Go. En production, on n'a besoin que du code compilé et des dépendances runtime.

```dockerfile
# Stage 1 : Builder — installe tout, compile
FROM python:3.12 AS builder

WORKDIR /app
COPY requirements.txt .

# Installation dans un répertoire virtuel isolé
RUN pip install --no-cache-dir --prefix=/install -r requirements.txt

# Stage 2 : Image finale — légère, sans les outils de build
FROM python:3.12-slim AS runtime

WORKDIR /app

# On copie UNIQUEMENT les packages installés depuis le builder
COPY --from=builder /install /usr/local
COPY src/ .

# Utilisateur non-root pour la sécurité
RUN useradd -m appuser
USER appuser

CMD ["python", "main.py"]
```

**Gains** :
- **Taille** : image finale 3-5x plus petite (ex : 1.2 Go → 250 Mo)
- **Sécurité** : aucun outil de build (gcc, pip, git) dans l'image de production — surface d'attaque réduite
- **Cache Docker** : les layers du builder sont cachées séparément — un changement de code ne rebuild pas les dépendances

**Pattern courant** : `builder` (install deps + compile) → `tester` (run tests) → `runtime` (image finale). Si les tests échouent, l'image finale n'est pas produite.

---

## III. Qu'est-ce que Helm et comment gère-t-il les déploiements Kubernetes ?

**Question** : "Qu'est-ce que Helm dans l'écosystème Kubernetes ? Pourquoi utilise-t-on des Helm Charts plutôt que des fichiers YAML bruts ?"

**Réponse** :

**Helm** est le gestionnaire de paquets de Kubernetes — il permet de définir, installer, mettre à jour et rollback des applications Kubernetes via des **Charts** (paquets de templates YAML paramétrables).

**Problème sans Helm** : déployer une application sur Kubernetes nécessite plusieurs fichiers YAML (Deployment, Service, Ingress, ConfigMap, Secret, HPA...). Dupliquer ces fichiers pour chaque environnement (dev, staging, prod) crée une explosion de YAML redondant difficile à maintenir.

**Structure d'un Chart Helm** :

```
mon-app/
  Chart.yaml          # métadonnées (nom, version, description)
  values.yaml         # valeurs par défaut paramétrables
  values-prod.yaml    # surcharge pour la production
  templates/
    deployment.yaml   # template avec variables {{ .Values.xxx }}
    service.yaml
    ingress.yaml
    _helpers.tpl      # fonctions réutilisables
```

**Template avec variables** :

```yaml
# templates/deployment.yaml
apiVersion: apps/v1
kind: Deployment
spec:
  replicas: {{ .Values.replicaCount }}
  template:
    spec:
      containers:
        - name: {{ .Chart.Name }}
          image: "{{ .Values.image.repository }}:{{ .Values.image.tag }}"
          resources:
            limits:
              memory: {{ .Values.resources.memory }}
```

```yaml
# values.yaml (défaut)
replicaCount: 1
image:
  repository: mon-app
  tag: latest
resources:
  memory: 256Mi

# values-prod.yaml (surcharge)
replicaCount: 5
resources:
  memory: 1Gi
```

**Commandes clés** :

```bash
helm install mon-app ./mon-app -f values-prod.yaml  # déploiement initial
helm upgrade mon-app ./mon-app -f values-prod.yaml  # mise à jour
helm rollback mon-app 1                             # rollback vers revision 1
helm list                                           # lister les releases
helm history mon-app                                # historique des déploiements
```

**Avantages** : un seul Chart pour tous les environnements (seules les values changent), rollbacks en une commande, partage via Helm repositories (comme npm pour Kubernetes), dry-run avant déploiement (`--dry-run`).

---

## IV. Qu'est-ce qu'ArgoCD et comment s'intègre-t-il dans un pipeline GitOps ?

**Question** : "Qu'est-ce qu'ArgoCD et quelle est la différence avec un pipeline CI/CD classique comme GitHub Actions pour déployer sur Kubernetes ?"

**Réponse** :

**ArgoCD** est un outil de déploiement continu **GitOps** pour Kubernetes. Il surveille en permanence un dépôt Git et s'assure que l'état du cluster correspond à ce qui est déclaré dans Git — si quelqu'un modifie manuellement le cluster, ArgoCD le détecte et peut le corriger automatiquement.

**GitOps** : le dépôt Git est la source de vérité unique pour l'état de l'infrastructure. Tout changement passe par une PR Git — jamais par des commandes manuelles `kubectl`.

**GitHub Actions vs ArgoCD** :

```
GitHub Actions (Push-based CI/CD) :
  Code push → CI build → Tests → kubectl apply (depuis la CI)
  Problème : la CI a besoin d'accès direct au cluster (credentials dans GitHub Secrets)

ArgoCD (Pull-based GitOps) :
  Code push → CI build → Push image → Mettre à jour le tag dans le repo Git
                                              |
                                        ArgoCD surveille le repo Git
                                              |
                                        ArgoCD pull et applique les changements
                                        (depuis l'intérieur du cluster)
```

**Avantages d'ArgoCD** :
- **Sécurité** : le cluster tire ses configs depuis Git (pull), pas de credentials cluster exposés en dehors
- **Réconciliation** : si quelqu'un fait un `kubectl edit` manuel, ArgoCD détecte la dérive et peut resync automatiquement
- **Audit** : tout changement est tracé dans Git (PR, blame, historique)
- **Multi-cluster** : ArgoCD peut gérer plusieurs clusters depuis une interface centrale

**Workflow complet GitOps** :

```
Développeur
    |
    | git push feature-branch
    v
[GitHub Actions — CI]
    | - Tests unitaires
    | - Build image Docker
    | - Push image → registry (ECR, DockerHub)
    | - Met à jour le tag image dans le repo GitOps
    v
[Repo GitOps] (ex: helm-values/prod/values.yaml)
    image.tag: v1.2.3  ← modifié par la CI
    |
    | ArgoCD détecte le changement (polling toutes les 3min ou webhook)
    v
[ArgoCD]
    | - Compare l'état Git vs l'état du cluster
    | - Applique le Helm Chart avec le nouveau tag
    v
[Cluster Kubernetes]  → nouvelle version déployée
```
