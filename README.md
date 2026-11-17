



Déploiement automatisé de n8n (https://n8n.io) conteneurisé avec GitHub Actions et Kubernetes



Un git push sur main déclenche automatiquement le build de l'image Docker, son envoi dans un registre et son déploiement dans un cluster Kubernetes



git push (main)

&#x20;  │

&#x20;  ▼

GitHub Actions

&#x20;  │

&#x20;  ├─ 1. Build de l'image Docker (Dockerfile)

&#x20;  ├─ 2. Push de l'image dans le registre

&#x20;  └─ 3. Déploiement du manifeste Kubernetes





Deux variantes du pipeline existent :



| Workflow | Registre | Cluster | Déclenchement | État |

| deploy-test.yml | GitHub Container Registry (ghcr.io) | kind (cluster temporaire) | Automatique, à chaque push sur main | Validé |

| deploy-azure.yml | Azure Container Registry (bayachacr) | Azure Kubernetes Service (myAKSCluster) | Manuel | Écrit, non utilisable sans subscription active |



n8n-deploy/

├── Dockerfile                          # Image n8n, version fixée (2.37.10)

├── kubernetes/

│   └── n8n-deployment.yml              # ConfigMap Caddy, PVC, Deployment, Service

└── .github/workflows/

&#x20;   ├── deploy-test.yml                 # Pipeline ghcr.io + kind

&#x20;   └── deploy-azure.yml                # Pipeline ACR + AKS (OIDC)



\- Version de n8n fixée: (n8nio/n8n:2.37.10) plutôt que latest, pour éviter une migration involontaire de la base de données n8n lors d'un rebuild.

\- Caddy:  en conteneur annexe pour le HTTPS (domaine nip.io).

\- Stratégie Recreate : un seul pod n8n à la fois, car le volume est en ReadWriteOnce

\- Authentification GitHub → Azure par OIDC (Federated Credential) : aucun mot de passe stocké dans GitHub.





Étapes : build de l'image, login à ghcr.io avec le GITHUB\_TOKEN, push de l'image taguée avec le SHA du commit, création d'un cluster kind, chargement de l'image, kubectl apply du manifeste, attente de la disponibilité du déploiement.



Le cluster kind est détruit à la fin du job : ce pipeline sert à valider le build et le déploiement, pas à héberger n8n.



Étapes : login Azure (OIDC), az acr build, récupération des identifiants AKS, kubectl apply et kubectl set image



\- Une subscription Azure active avec l'ACR et l'AKS (créés par Terraform).

\- Une App Registration github-n8n-deploy` avec un Federated Credential dont le subject est :

&#x20; `repo:baya63-tech@279504364/n8n-deploy@1375884484:ref:refs/heads/main`

&#x20; (GitHub inclut ici les ID numériques du propriétaire et du dépôt).

\- Les rôles suivants pour cette identité :

&#x20; - `AcrPush` sur l'ACR

&#x20; - `Azure Kubernetes Service Cluster User Role` sur l'AKS

&#x20; - `Azure Kubernetes Service RBAC Writer` sur l'AKS

\- Les secrets GitHub : `AZURE\_CLIENT\_ID`, `AZURE\_TENANT\_ID`, `AZURE\_SUBSCRIPTION\_ID`.



\### Lancement



Onglet Actions → Deploy n8n to AKS → Run workflow



L'ACR, l'AKS et le rôle AcrPull (AKS → ACR) sont décrits dans un projet Terraform séparé (main.tf, provider azurerm -> 4.0).





\- La subscription Azure sponsorisée a expiré : le resource group, l'AKS et l'ACR ne sont plus accessibles. Le pipeline Azure n'a donc pas pu être exécuté jusqu'au bout.

\- Avant l'expiration, n8n tournait sur l'AKS (pod 2/2 Running, exposé via un LoadBalancer).

\- Dans l'environnement de test (kind), le `LoadBalancer` reste sans IP externe et le HTTPS Caddy pour le domaine nip.io n'est pas fonctionnel.



1\. az login puis az account set --subscription <ID>

2\. terraform apply pour recréer l'ACR, l'AKS et le role AcrPull

3\. Mettre à jour le secret AZURE\_SUBSCRIPTION\_ID

4\. Remplacer l'IP nip.io dans kubernetes/n8n-deployment.yml par la nouvelle IP du LoadBalancer

5\. Faire attribuer les 3 rôles à github-n8n-deploy

6\. Retirer allow-no-subscriptions: true de deploy-azure.yml, puis lancer le workflow manuellement

