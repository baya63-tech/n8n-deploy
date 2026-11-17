n8n-deploy

Déploiement automatisé de n8n (https://n8n.io) conteneurisé avec GitHub Actions et Kubernetes.

Un git push sur main déclenche automatiquement le build de l'image Docker, son envoi dans un registre et son déploiement dans un cluster Kubernetes.

ARCHITECTURE

À chaque git push sur main, GitHub Actions exécute les étapes suivantes :

1. Build de l'image Docker à partir du Dockerfile
2. Push de l'image dans le registre
3. Déploiement du manifeste Kubernetes

Deux variantes du pipeline existent.

Workflow deploy-test.yml :
- Registre : GitHub Container Registry (ghcr.io)
- Cluster : kind (cluster temporaire)
- Déclenchement : automatique, à chaque push sur main
- État : validé

Workflow deploy-azure.yml :
- Registre : Azure Container Registry (bayachacr)
- Cluster : Azure Kubernetes Service (myAKSCluster)
- Déclenchement : manuel
- État : écrit, non utilisable sans subscription active

STRUCTURE DU DÉPÔT

- Dockerfile : image n8n, version fixée (2.37.10)
- kubernetes/n8n-deployment.yml : ConfigMap Caddy, PVC, Deployment, Service
- .github/workflows/deploy-test.yml : pipeline ghcr.io et kind
- .github/workflows/deploy-azure.yml : pipeline ACR et AKS (OIDC)

CHOIX TECHNIQUES

- Version de n8n fixée (n8nio/n8n:2.37.10) plutôt que latest, pour éviter une migration involontaire de la base de données n8n lors d'un rebuild.
- Caddy en conteneur annexe pour le HTTPS (domaine nip.io).
- Stratégie Recreate : un seul pod n8n à la fois, car le volume est en ReadWriteOnce.
- Authentification GitHub vers Azure par OIDC (Federated Credential) : aucun mot de passe stocké dans GitHub.

PIPELINE DE TEST (deploy-test.yml)

Étapes : build de l'image, login à ghcr.io avec le GITHUB_TOKEN, push de l'image taguée avec le SHA du commit, création d'un cluster kind, chargement de l'image, kubectl apply du manifeste, attente de la disponibilité du déploiement.

Le cluster kind est détruit à la fin du job : ce pipeline sert à valider le build et le déploiement, pas à héberger n8n.

PIPELINE AZURE (deploy-azure.yml)

Étapes : login Azure (OIDC), az acr build, récupération des identifiants AKS, kubectl apply et kubectl set image.

Prérequis :

- Une subscription Azure active avec l'ACR et l'AKS (créés par Terraform).
- Une App Registration github-n8n-deploy avec un Federated Credential dont le subject est repo:baya63-tech@279504364/n8n-deploy@1375884484:ref:refs/heads/main (GitHub inclut ici les ID numériques du propriétaire et du dépôt).
- Les rôles suivants pour cette identité : AcrPush sur l'ACR, Azure Kubernetes Service Cluster User Role sur l'AKS, Azure Kubernetes Service RBAC Writer sur l'AKS.
- Les secrets GitHub : AZURE_CLIENT_ID, AZURE_TENANT_ID, AZURE_SUBSCRIPTION_ID.

Lancement : onglet Actions, workflow Deploy n8n to AKS, bouton Run workflow.

INFRASTRUCTURE (TERRAFORM)

L'ACR, l'AKS et le rôle AcrPull (AKS vers ACR) sont décrits dans un projet Terraform séparé (main.tf, provider azurerm version 4.x).

ÉTAT ACTUEL ET LIMITES

- La subscription Azure sponsorisée a expiré : le resource group, l'AKS et l'ACR ne sont plus accessibles. Le pipeline Azure n'a donc pas pu être exécuté jusqu'au bout.
- Avant l'expiration, n8n tournait sur l'AKS (pod 2/2 Running, exposé via un LoadBalancer).
- Dans l'environnement de test (kind), le LoadBalancer reste sans IP externe et le HTTPS Caddy pour le domaine nip.io n'est pas fonctionnel.

REPRENDRE SUR UNE NOUVELLE SUBSCRIPTION

1. az login puis az account set --subscription ID_DE_LA_SUBSCRIPTION
2. terraform apply pour recréer l'ACR, l'AKS et le rôle AcrPull
3. Mettre à jour le secret AZURE_SUBSCRIPTION_ID
4. Remplacer l'IP nip.io dans kubernetes/n8n-deployment.yml par la nouvelle IP du LoadBalancer
5. Faire attribuer les 3 rôles à github-n8n-deploy
6. Retirer allow-no-subscriptions: true de deploy-azure.yml, puis lancer le workflow manuellement