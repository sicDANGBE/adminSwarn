# Variables & Configuration - Documentation Complète

## 📋 Hiérarchie Ansible

Ansible évalue les variables dans cet ordre (plus haute priorité → petite priorité):

```
1. Extra vars (ansible-playbook --extra-vars)
2. Task vars
3. Block vars
4. Host vars (inventory.yml)
5. Group vars (group_vars/*)
6. Role defaults (defaults/main.yml)
7. Built-in defaults
```

---

## 🌍 Variables Globales (inventory.yml)

Disponibles pour TOUS les rôles et playbooks.

### Contexte: `all` > `vars`

```yaml
all:
  vars:
    # ========== UTILISATEURS & SSH ==========
    sysadmin_user: supadmin
    ansible_user: supadmin
    ansible_port: 49281
    ansible_python_interpreter: /usr/bin/python3
    ansible_become: yes
    local_public_key: "~/.ssh/id_rsa.pub"
    
    # ========== KUBERNETES & INGRESS ==========
    main_domain: dgsynthex.online
    k8s_storage_class: local-path
    
    # ========== AUTHENTIFICATION ANSIBLE ==========
    ansible_ssh_private_key_file: ~/.ssh/id_rsa
    become_method: sudo
    become_user: root
```

### Détail des Variables

#### SSH & Utilisateurs
| Variable | Valeur | Description |
|----------|--------|---|
| `sysadmin_user` | supadmin | Utilisateur admin système |
| `ansible_user` | supadmin | Utilisateur SSH Ansible |
| `ansible_port` | 49281 | SSH port (non-standard) |
| `ansible_python_interpreter` | /usr/bin/python3 | Python pour Ansible |
| `ansible_become` | yes | Activer sudo automatiquement |
| `local_public_key` | ~/.ssh/id_rsa.pub | Chemin clé SSH locale |

#### Kubernetes & Ingress
| Variable | Valeur | Description |
|----------|--------|---|
| `main_domain` | dgsynthex.online | Domaine pour Ingress rules |
| `k8s_storage_class` | local-path | K3s default storage |

---

## 🏠 Variables Hôtes (inventory.yml)

Surcharge variables globales par hôte si nécessaire.

### Control Plane (server group)

```yaml
server:
  hosts:
    ovh.core:
      ansible_host: vps-940a692a.vps.ovh.net
      # Override variables if needed
      # custom_var: value
```

### Workers (agents group)

```yaml
agents:
  hosts:
    ovh.worker.01:
      ansible_host: vps-255dab72.vps.ovh.net
    ovh.worker.02:
      ansible_host: vps-9e3ed523.vps.ovh.net
```

---

## 🔐 Secrets Vault (group_vars/all/secret.yml)

Fichier **chiffré** contenant les credentials sensibles.

### Structure du Fichier

```yaml
---
# Database
postgres_password: "VerySecurePassword123!"
postgres_user: postgres

# n8n
n8n_admin_password: "N8NAdminPass123!"
n8n_admin_email: "admin@dgsynthex.online"
n8n_encryption_key: "{{ lookup('password', '/tmp/n8n_key') }}"

# RabbitMQ
rabbitmq_username: user
rabbitmq_password: "RabbitPass123!"
rabbitmq_erlang_cookie: "ERLANGCOOKIE123456"

# Grafana
grafana_admin_password: "GrafanaPass123!"
grafana_admin_user: admin

# GitHub Integration
github_token: "ghp_xxxxxxxxxxxxxxxxxxxxxxxxxxxxx"

# API Keys
mqtt_username: "mqtt_user"
mqtt_password: "MQTTPass123!"

# Cloud APIs (if needed)
ovh_api_key: "xxxxx"
ovh_api_secret: "xxxxxxxxxxxx"
ovh_consumer_key: "xxxxx"
```

### Gestion du Vault

#### Créer le Fichier Vault
```bash
# Créer new secret file (interactif, vous demandera password)
ansible-vault create group_vars/all/secret.yml

# Ou créer depuis template
cp group_vars/all/secret.yml.example group_vars/all/secret.yml
ansible-vault encrypt group_vars/all/secret.yml
```

#### Éditer le Fichier Vault
```bash
# Éditer (vous demandera password vault)
ansible-vault edit group_vars/all/secret.yml

# View (sans éditer)
ansible-vault view group_vars/all/secret.yml
```

#### Exécuter Playbook avec Vault
```bash
# Demande password vault interactivement
ansible-playbook -i inventory.yml site.yml --ask-vault-pass

# Ou depuis fichier password (moins sécurisé)
ansible-playbook -i inventory.yml site.yml --vault-password-file ~/.vault_pass
```

#### Stocker Password Vault Sécurisé
```bash
# Option 1: Script with secure permissions
echo "MyVaultPassword123" > ~/.vault_pass
chmod 600 ~/.vault_pass

# Option 2: Utiliser pass manager
pass show ansible_vault_pass

# Option 3: Lire depuis 1Password/Bitwarden (CLI)
op read op://vault/Ansible\ Vault/password
```

---

## 📝 Variables Rôle: common

### Emplacement
`roles/common/` (aucun defaults/main.yml généralement, hérite globals)

### Variables Contrôlant le Comportement

| Variable | Défaut | Description |
|----------|--------|---|
| `nftables_enabled` | true | Activer firewall NFTables |
| `nftables_ssh_port` | 49281 | Port SSH pour firewall |
| `fail2ban_enabled` | true | Activer Fail2Ban |
| `fail2ban_bantime` | 3600 | Ban duration (secondes) |

### Templates Utilisées

```
roles/common/templates/
├── nftables_base.j2        # Base firewall rules
└── nftables_logging.conf.j2 # Logging rules
```

Variables accessibles dans templates:
- `nftables_ssh_port`
- `sysadmin_user`
- `main_domain` (pour emails)

---

## 📝 Variables Rôle: k8s

### Emplacement
`roles/k8s/defaults/main.yml`

### Configuration K3s Installée

```yaml
# ===== K3s Version =====
k3s_version: "v1.28.0"              # Pinned K3s version
# Autres versions populaires:
# - "v1.27.10" (stable)
# - "latest" (auto-update)
# - "v1.29.0" (cutting edge, moins testé)

# ===== Networking =====
k3s_flannel_iface: "eth0"           # Interface pour Flannel CNI
# Autres interfaces possibles:
# - "eth0" (default, interface primaire)
# - "ens3" (OVH VPS)
# - "enp0s3" (autre VPS)

# ===== Security Token =====
k3s_token: "{{ lookup('password', '/tmp/k3s_token length=32') }}"
# Auto-generated sécurisé

# ===== Extra Arguments =====
k3s_extra_server_args: ""  # Args additionnels pour K3s server
k3s_extra_agent_args: ""   # Args additionnels pour K3s agents

# Exemples extra args:
# k3s_extra_server_args: "--disable=traefik,servicelb"
# k3s_extra_server_args: "--cluster-cidr=10.42.0.0/16 --service-cidr=10.43.0.0/16"

# ===== Kubeconfig =====
kubeconfig_path: "/etc/rancher/k3s/k3s.yaml"
kubeconfig_mode: "0640"

# ===== API Server =====
k3s_api_server_addr: "0.0.0.0"      # Écoute toutes les interfaces
k3s_api_server_port: 6443           # Port API (standard K8s)

# ===== Scheduler & Controller =====
k3s_feature_gates: ""               # Experimental features
```

### Variables de Conditionnels

```yaml
k8s_cluster_name: "{{ inventory_hostname }}"
k8s_server_ip: "{{ ansible_default_ipv4.address }}"
```

Utilisées pour:
- Déterminer qui est server vs agent
- Configuration Flannel par nœud
- Logs et debugging

### Firewall K8s (NFTables Template)

Variables dans `roles/k8s/templates/nftables_k8s.conf.j2`:
- `k3s_api_server_port` (6443)
- `k8s_node_port_min`, `k8s_node_port_max` (30000-32767)
- Ranges pod CIDR

---

## 📝 Variables Rôle: app_k8s

### Emplacement
`roles/app_k8s/vars/main.yml` (vars, pas defaults!)

⚠️ **IMPORTANT**: `vars/` a plus haute priorité que `defaults/`

### Versions Helm Charts

```yaml
helm_versions:
  # ===== Infrastructure =====
  traefik: "27.0.0"          # Ingress controller (optional, K3s native available)
  
  # ===== Container Management =====
  portainer: "2.21.5"        # K8s management UI
  
  # ===== Databases & Message Brokers =====
  postgres: "15.0.0"         # PostgreSQL Bitnami chart
  rabbitmq: "12.0.0"         # RabbitMQ Bitnami chart
  
  # ===== Automation & Orchestration =====
  n8n: "0.14.0"              # n8n Community chart (ou OCI)
  nodered: "3.1.0"           # Node-RED community chart
  
  # ===== IoT & Messaging =====
  mosquitto: "4.0.0"         # MQTT Broker Bitnami
  
  # ===== LLM & AI =====
  ollama: "0.1.0"            # Ollama community chart
  
  # ===== Observability =====
  monitoring: "45.0.0"       # kube-prometheus-stack
  loki: "6.0.0"              # Loki (log aggregation, optional)
```

### Repositories Helm

```yaml
# Auto-added par app_k8s role
helm_repos:
  - name: traefik
    url: https://traefik.github.io/charts
  - name: portainer
    url: https://portainer.github.io/helm/
  - name: bitnami
    url: https://charts.bitnami.com/bitnami
  - name: prometheus-community
    url: https://prometheus-community.github.io/helm-charts
  - name: n8n
    url: https://n8nio.github.io/helm-charts/
```

### Namespaces Kubernetes

```yaml
k8s_namespaces:
  - name: portainer
    labels:
      type: management
  - name: apps
    labels:
      type: applications
  - name: admin
    labels:
      type: administrative
  - name: monitoring
    labels:
      type: observability
  - name: ollama-system
    labels:
      type: ai-ml
```

### Storage Class Configuration

```yaml
k8s_storage_class: "local-path"    # Par défaut K3s

# Alternative configurations:
# k8s_storage_class: "nfs"          # Si NFS externe
# k8s_storage_class: "ceph"         # Si Ceph cluster

# PVC Sizes par Application
pvc_sizes:
  portainer: "10Gi"
  postgres: "20Gi"
  rabbitmq: "10Gi"
  n8n: "10Gi"
  nodered: "5Gi"
  mosquitto: "5Gi"
  ollama: "50Gi"              # Models storage (can be large)
  prometheus: "20Gi"
  grafana: "5Gi"
```

### Configuration Applications

#### PostgreSQL
```yaml
postgres:
  version: "15"
  user: "{{ postgres_user }}"              # From secret.yml
  password: "{{ postgres_password }}"      # From secret.yml
  database: "applications"
  port: 5432
```

#### n8n
```yaml
n8n:
  admin_user: "{{ n8n_admin_user }}"
  admin_password: "{{ n8n_admin_password }}"
  admin_email: "{{ n8n_admin_email }}"
  db_type: postgres
  db_host: "postgres.apps.svc.cluster.local"
  db_port: 5432
  db_user: "{{ postgres_user }}"
  db_password: "{{ postgres_password }}"
  encryption_key: "{{ n8n_encryption_key }}"
```

#### Grafana
```yaml
grafana:
  admin_user: "{{ grafana_admin_user }}"
  admin_password: "{{ grafana_admin_password }}"
  data_source: "prometheus"
  persistence_enabled: true
  persistence_size: "5Gi"
```

#### Prometheus
```yaml
prometheus:
  retention: "30d"
  scrape_interval: "15s"
  evaluation_interval: "15s"
  persistence_enabled: true
  persistence_size: "20Gi"
```

---

## 📝 Variables Rôle: runner

### Emplacement
`roles/runner/defaults/main.yml`

```yaml
# CI/CD Runner Configuration
runner_type: "gitlab"                    # gitlab, github-actions, custom
runner_endpoint: "https://gitlab.com"   # For GitLab runners
runner_token: "{{ lookup('env','RUNNER_TOKEN') }}"
runner_name: "k8s-runner-{{ inventory_hostname }}"
runner_namespace: "ci-cd"
runner_image: "gitlab/gitlab-runner:latest"

# Resource Limits
runner_cpu_request: "100m"
runner_cpu_limit: "500m"
runner_memory_request: "128Mi"
runner_memory_limit: "512Mi"

# Tags for job selection
runner_tags:
  - k8s
  - docker
  - automation
```

---

## 📝 Variables Rôle: audit_security

### Emplacement
`roles/audit_security/defaults/main.yml`

```yaml
audit_output_dir: "{{ playbook_dir }}/audit_reports"
audit_timestamp: "{{ ansible_date_time.iso8601 }}"

# Checks à effectuer
audit_check_kernel: true
audit_check_users: true
audit_check_firewall: true
audit_check_ssh: true
audit_check_containers: true
```

---

## 🔄 Utilisation Pratique

### Exécuter avec Variables Supplémentaires

```bash
# Surcharger une variable unique
ansible-playbook -i inventory.yml site.yml \
  --extra-vars "main_domain=custom.com"

# Surcharger plusieurs variables
ansible-playbook -i inventory.yml site.yml \
  --extra-vars "k3s_version=v1.27.0 postgres_password=NewPass123"

# Depuis fichier JSON
ansible-playbook -i inventory.yml site.yml \
  --extra-vars "@custom_vars.json"

# Depuis fichier YAML
ansible-playbook -i inventory.yml site.yml \
  --extra-vars "@custom_vars.yaml"
```

### Fichier Variables Externe (custom_vars.yaml)

```yaml
---
main_domain: subdomain.example.com
k3s_version: v1.28.5
postgres_password: "{{ vault_postgres_password }}"
n8n_admin_password: "{{ vault_n8n_password }}"
nftables_ssh_port: 22022
```

Exécuter:
```bash
ansible-playbook -i inventory.yml site.yml \
  --extra-vars "@custom_vars.yaml" \
  --ask-vault-pass
```

### Récupérer Valeur Variable Au Runtime

```bash
# Afficher valeur d'une variable
ansible ovh.core -i inventory.yml -m debug \
  -a "var=main_domain"

# Afficher toutes les variables d'un hôte
ansible ovh.core -i inventory.yml -m debug \
  -a "var=hostvars['ovh.core']"
```

### Tester Syntaxe Variables

```bash
# Dry-run: affiche ce qui serait changé
ansible-playbook -i inventory.yml site.yml --check

# Dry-run verbose avec debug des variables
ansible-playbook -i inventory.yml site.yml --check -vvv
```

---

## 📊 Matrice Accessibilité Variables

| Variable | Global | common | k8s | app_k8s | runner | audit |
|----------|--------|--------|-----|---------|--------|-------|
| `main_domain` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `sysadmin_user` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `ansible_port` | ✅ | ✅ | ✅ | ✅ | ✅ | ✅ |
| `k3s_version` | ❌ | ❌ | ✅ | ✅ | ❌ | ❌ |
| `helm_versions` | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |
| `postgres_password` | ❌ | ❌ | ❌ | ✅ | ❌ | ❌ |
| Custom app vars | ❌ | ❌ | ❌ | ✅ | ✅ | ❌ |

---

Voir aussi: [inventory.yml](inventory.yml), [INSTALLATION.md](INSTALLATION.md), [ROLES.md](ROLES.md)
