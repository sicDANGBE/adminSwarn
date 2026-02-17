# On part de l'image officielle GitHub Runner (Base Ubuntu)
FROM ghcr.io/actions/actions-runner:latest

# On passe root temporairement pour l'installation
USER root

# Désactivation des invites interactives pour éviter les erreurs de build
ENV DEBIAN_FRONTEND=noninteractive

# 1. Mise à jour système et installation dépendances minimales
# On supprime curl/git/etc à la fin si on veut être ultra-strict, 
# mais ici on en a besoin pour le checkout du code et helm.
RUN apt-get update && apt-get upgrade -y && \
    apt-get install -y --no-install-recommends \
    curl \
    ca-certificates \
    git \
    unzip \
    gettext-base \
    perl \
    && apt-get clean \
    && rm -rf /var/lib/apt/lists/*

# 2. Installation de Kubectl (Version Stable v1.28.x pour matcher K3s)
# SECURITÉ : On vérifie le SHA256 du binaire téléchargé
RUN curl -LO "https://dl.k8s.io/release/v1.28.5/bin/linux/amd64/kubectl" && \
    curl -LO "https://dl.k8s.io/release/v1.28.5/bin/linux/amd64/kubectl.sha256" && \
    echo "$(cat kubectl.sha256)  kubectl" | sha256sum --check && \
    chmod +x kubectl && \
    mv kubectl /usr/local/bin/ && \
    rm kubectl.sha256

# 3. Installation de Helm (Script officiel)
RUN curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 4. DURCISSEMENT (HARDENING)
# Suppression de sudo pour empêcher toute élévation de privilèges
# 4. DURCISSEMENT (HARDENING)
# - On s'assure que le runner ne peut PAS utiliser sudo
# - On limite les permissions sur les binaires sensibles
RUN rm -f /usr/bin/sudo && \
    chmod 755 /usr/local/bin/kubectl /usr/local/bin/helm

# On repasse sur l'utilisateur standard 'runner' (UID 1001)
# C'est CRITIQUE : le conteneur ne tournera jamais en root

USER runner

# RUN echo "--- 📂 Vérification du répertoire actuel ---" && \
#     pwd && \
#     ls -la && \
#     echo "--- 🛠️ Vérification des outils installés ---" && \
#     which kubectl && kubectl version --client && \
#     which helm && helm version --template='{{.Version}}' && \
#     which envsubst && envsubst --version | head -n 1 && \
#     echo "--- 🛡️ Vérification de l'utilisateur ---" && \
#     id
# On définit le répertoire de travail pour les actions
# WORKDIR /home/runner/actions-runner