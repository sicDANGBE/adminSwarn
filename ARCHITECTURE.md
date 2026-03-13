# Architecture - adminSwarn Kubernetes Infrastructure

## 📐 Vue d'Ensemble Globale

adminSwarn est une solution **Infrastructure-as-Code** complète pour déployer et maintenir un cluster Kubernetes (K3s) de production avec une suite complète de microservices.

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                   │
│         Kubernetes Cluster (K3s) - dgsynthex.online             │
│                                                                   │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                   Control Plane                            │ │
│  │                  ovh.core (vps-940a692a)                  │ │
│  │                                                             │ │
│  │  - K3s Server (API, etcd, scheduler, controller-mgr)      │ │
│  │  - Traefik Ingress Controller                             │ │
│  │  - Local-path Storage Provisioner                         │ │
│  │                                                             │ │
│  └────────────────────────────────────────────────────────────┘ │
│   ↑                              ↑                              ↑ │
│   │ K8s API, CNI (Flannel)      │ kubelet      │ kubelet       │ │
│   │                              │              │              │ │
│  ┌─────────────────┐    ┌─────────────────┐   ┌──────────────┐ │
│  │  Worker Node 1  │    │  Worker Node 2  │   │  Observ.    │ │
│  │ ovh.worker.01   │    │ ovh.worker.02   │   │  (local)    │ │
│  │ (vps-255dab72)  │    │ (vps-9e3ed523)  │   │             │ │
│  │                 │    │                 │   │             │ │
│  │ K3s Agent       │    │ K3s Agent       │   │ Monitoring  │ │
│  └─────────────────┘    └─────────────────┘   └──────────────┘ │
│                                                                   │
└─────────────────────────────────────────────────────────────────┘
                              ↓ HTTPS
                      ┌───────────────────┐
                      │  Traefik Ingress  │
                      │ (Load Balancer)   │
                      │  - Auto SSL/TLS   │
                      │  - Let's Encrypt  │
                      └───────────────────┘
                              ↓
                      ┌───────────────────┐
                      │  External Traffic │
                      └───────────────────┘
```

---

## 🌐 Topologie Réseau

### Nœuds OVH VPS

| Hostname | Rôle | FQDN | IP Locale | Utilisateur SSH |
|----------|------|------|-----------|---|
| `ovh.core` | Control Plane | vps-940a692a.vps.ovh.net | - | supadmin:49281 |
| `ovh.worker.01` | Worker | vps-255dab72.vps.ovh.net | - | supadmin:49281 |
| `ovh.worker.02` | Worker | vps-9e3ed523.vps.ovh.net | - | supadmin:49281 |

**Réseau K3s Interne**:
- **Pod CIDR**: 10.42.0.0/16 (default K3s)
- **Service CIDR**: 10.43.0.0/16 (default K3s)
- **CNI**: Flannel (interface configurée via `k3s_flannel_iface`)
- **DNS Cluster**: 10.43.0.10 (CoreDNS)

### Ingress & Routage

```
Internet (port 443/80)
    ↓
Traefik Ingress Controller (kube-system)
    ↓
Services K8s (via ClusterIP)
    ↓
Pods (différents namespaces)
```

**Domaine Racine**: `dgsynthex.online`  
**Certificats**: Let's Encrypt (via Traefik ACME)  
**Endpoints**:
- `traefik.dgsynthex.online` - Dashboard Traefik
- `portainer.dgsynthex.online` - Gestion K8s
- `n8n.dgsynthex.online` - Workflow automation
- `nodered.dgsynthex.online` - Visual orchestration
- `mqtt.dgsynthex.online` - MQTT Broker
- `rabbitmq.dgsynthex.online` - RabbitMQ Management
- `grafana.dgsynthex.online` - Monitoring Dashboard
- `ollama.dgsynthex.online` - LLM Inference API

---

## 📦 Namespaces Kubernetes

```
K3s Cluster
├── kube-system
│   ├── Traefik (Ingress Controller)
│   ├── CoreDNS (DNS)
│   ├── Metrics Server
│   └── Local Path Provisioner (Storage)
├── kube-node-lease
│   └── Node Heartbeats
├── default
│   └── (Unused - workloads in custom namespaces)
├── portainer (Namespace isolated)
│   ├── Portainer CE Pod
│   ├── Persistent Volume
│   └── Service + Ingress
├── apps (Main workloads)
│   ├── PostgreSQL
│   ├── RabbitMQ
│   ├── N8N
│   ├── Node-RED
│   └── MQTT Broker (Mosquitto)
├── admin (Administrative tools)
│   └── (Reserved for admin tasks)
├── monitoring (Observability Stack)
│   ├── Prometheus
│   ├── Grafana
│   ├── AlertManager
│   └── Loki (Log Aggregation)
└── ollama-system (LLM Services)
    ├── Ollama API Server
    ├── Model Cache
    └── Persistent Storage
```

---

## 🔄 Flux de Déploiement Ansible

```
site.yml (Orchestrator Principal)
│
├─ Play 1: Infrastructure Système
│  │  Hosts: k8s_cluster (ALL NODES: control + workers)
│  │  Handlers: Restart SSH, Reload NFTables, Restart Rsyslog
│  │
│  ├─ Role: clean (Tags: never, clean)
│  │  └─> Nettoyage destructif COMPLET (utilisé pour reset)
│  │
│  └─ Role: common (Tags: common, system)
│     ├─> OS Updates
│     ├─> SSH Hardening (Key-only, port 49281)
│     ├─> Firewall NFTables (Modular)
│     ├─> Fail2Ban setup
│     ├─> User Management + Privilege Separation
│     ├─> System Configuration
│     └─> Logging Setup
│
├─ Play 2: Kubernetes Cluster
│  │  Hosts: k8s_cluster (ALL NODES)
│  │  Tags: k8s, kubernetes
│  │
│  └─ Role: k8s
│     ├─> Disable Swap
│     ├─> Load Kernel Modules (overlay, br_netfilter)
│     ├─> K3s Server Installation (Control Plane) 
│     ├─> K3s Agent Installation (Workers)
│     ├─> kubectl Access Config
│     └─> Traefik ACME Configuration
│
├─ Play 3: Applications Kubernetes
│  │  Hosts: server (Control Plane ONLY)
│  │  Tags: app_k8s
│  │
│  └─ Role: app_k8s
│     ├─> DNS Configuration Fix
│     ├─> Helm Installation
│     ├─> Namespace Creation (portainer, apps, admin, monitoring, ollama-system)
│     ├─> StorageClass Verification
│     ├─> Helm Chart Deployments
│     │  ├─> Traefik (already deployed via K3s)
│     │  ├─> Portainer CE
│     │  ├─> PostgreSQL
│     │  ├─> RabbitMQ
│     │  ├─> N8N
│     │  ├─> Node-RED
│     │  ├─> MQTT Broker
│     │  ├─> Ollama
│     │  └─> Prometheus + Grafana (kube-prometheus-stack)
│     └─> Ingress Rules Creation
│
├─ Play 4: Runners Kubernetes
│  │  Hosts: server (Control Plane ONLY)
│  │  Tags: k8s_runner
│  │
│  └─ Role: runner
│     └─> Deploy CI/CD Runners/Automation Agents
│
└─ Play 5: Audit Sécurité
   │  Hosts: k8s_cluster (ALL NODES)
   │  Tags: never, audit (never executed by default)
   │
   └─ Role: audit_security
      ├─> Kernel Parameters Report
      ├─> User Accounts Report
      ├─> Firewall Rules Report
      ├─> SSH Configuration Report
      ├─> Docker/Container Config Report
      └─> Générate Compliance Report (locally)
```

---

## 🗂️ Arborescence Ansible

```
ansible/
├── site.yml                               # Main Orchestrator
├── inventory.yml                          # Inventory + Global Vars
├── README.md                              # Documentation
├── ARCHITECTURE.md                        # This file
├── project_export.md                      # Project export info
├── secret_ovh.yaml                        # OVH-specific secrets (optional)
│
├── group_vars/
│   └── all/
│       ├── secret.yml                     # Vault: Passwords, tokens, API keys
│       └── secret.yml.example             # Template for secrets
│
└── roles/
    ├── common/                            # Base system hardening
    │   ├── handlers/main.yml
    │   ├── tasks/main.yml
    │   └── templates/
    │       ├── nftables_base.j2
    │       └── nftables_logging.conf.j2
    │
    ├── k8s/                               # Kubernetes (K3s) setup
    │   ├── defaults/main.yml
    │   ├── tasks/main.yml
    │   └── templates/
    │       ├── nftables_k8s.conf.j2
    │       └── traefik-config.yaml.j2
    │
    ├── app_k8s/                           # Application deployment
    │   ├── tasks/main.yml
    │   ├── vars/main.yml
    │   ├── templates/ (if needed)
    │   └── monitoring.yml, mqtt.yml, ... (subtasks)
    │
    ├── runner/                            # CI/CD Runners
    │   ├── tasks/main.yml
    │   └── templates/deployment.yaml.j2
    │
    ├── audit_security/                    # Security Audit
    │   ├── defaults/main.yml
    │   └── tasks/main.yml
    │
    └── clean/                             # Destructive cleanup
        └── tasks/main.yml
```

---

## 📊 Données Persistantes

### StorageClass K3s

**Nom**: `local-path`  
**Type**: Local directory provisioning  
**Path**: `/var/lib/rancher/k3s/storage` (per node)  
**Reclaim Policy**: Delete  
**Volume Binding**: Per node affinity

### Applications avec Persistent Volumes

| App | PVC Name | Size | Namespace | Mount Path |
|-----|----------|------|-----------|------------|
| PostgreSQL | `postgres-pvc` | 20Gi | apps | `/var/lib/postgresql` |
| RabbitMQ | `rabbitmq-pvc` | 10Gi | apps | `/var/lib/rabbitmq` |
| N8N | `n8n-pvc` | 10Gi | apps | `/home/node/.n8n` |
| Node-RED | `nodered-pvc` | 5Gi | apps | `/data` |
| Portainer | `portainer-pvc` | 10Gi | portainer | `/data` |
| Ollama | `ollama-pvc` | 50Gi | ollama-system | `/root/.ollama` |
| Prometheus | `prometheus-pvc` | 20Gi | monitoring | `/prometheus` |
| Grafana | `grafana-pvc` | 5Gi | monitoring | `/var/lib/grafana` |

---

## 🔐 Sécurité - Architecture

### Couches de Sécurité

```
Internet
    ↓ (Port 80, 443)
Firewall Host (NFTables)
    ↓
Traefik Ingress + TLS Termination
    ↓
Network Policy (K8s, si configurée)
    ↓
Service Mesh (future improvement)
    ↓
RBAC + Pod Security Policies
    ↓
Application Runtime
```

### Contrôle d'Accès SSH

- **Port**: 49281 (non-standard)
- **Authentification**: Clés SSH uniquement (PasswordAuthentication: no)
- **Root Login**: Désactivé
- **Utilisateur Système**: `supadmin` (avec sudo sans mot de passe)

### Firewall NFTables

```
nftables configuration (modular)
├── Base rules (base.j2)
│   ├── SSH (port 49281)
│   ├── DNS (53)
│   └── NTP (123)
├── Kubernetes rules (k8s.conf.j2)
│   ├── API Server (6443)
│   ├── Kubelet (10250)
│   ├── Service traffic (30000-32767)
│   └── Pod CIDR (10.42.0.0/16)
└── Logging rules (logging.conf.j2)
    └── Rejected packets logging
```

---

## 🚀 Séquences de Déploiement Recommandées

### Déploiement Initial Complet

```bash
# 1. Infrastructure + Hardening
ansible-playbook -i inventory.yml site.yml --tags common

# 2. Kubernetes Cluster
ansible-playbook -i inventory.yml site.yml --tags k8s

# 3. Applications
ansible-playbook -i inventory.yml site.yml --tags app_k8s

# 4. CI/CD Runners (optionnel)
ansible-playbook -i inventory.yml site.yml --tags k8s_runner

# Ou tout en une fois (conseillé en réseau stable)
ansible-playbook -i inventory.yml site.yml --tags common,k8s,app_k8s,k8s_runner
```

### Mise à Jour Partielle

```bash
# Appliquer les hardening à jour
ansible-playbook -i inventory.yml site.yml --tags common --limit ovh.core

# Upgrader K3s seulement
ansible-playbook -i inventory.yml site.yml --tags k8s

# Redéployer une app spécifique
ansible-playbook -i inventory.yml site.yml --tags app_k8s --extra-vars "app=n8n"
```

### Audit & Diagnostic

```bash
# Générer rapport de conformité sécurité
ansible-playbook -i inventory.yml site.yml --tags audit

# Cleanup destructif (ATTENTION)
ansible-playbook -i inventory.yml site.yml --tags clean
```

---

## 📈 Évolutivité

### Passage à Plusieurs Clusters

```yaml
# Ajouter un nouveau cluster dans inventory.yml
all:
  children:
    k8s_cluster_prod:
      children:
        server:
          hosts: prod.server
        agents:
          hosts: prod.worker.01, prod.worker.02
    k8s_cluster_staging:
      children:
        server:
          hosts: staging.server
        agents:
          hosts: staging.worker.01
```

### Ajout de Nœuds Workers

1. Ajouter le nœud à `inventory.yml` (section `agents`)
2. Configurer SSH key et network access
3. Exécuter: `ansible-playbook -i inventory.yml site.yml --tags common,k8s --limit new.worker`

### Ajout de Nouvelles Applications

1. Créer un fichier task dans `roles/app_k8s/tasks/app_name.yml`
2. Importer dans `roles/app_k8s/tasks/main.yml`
3. Définir les variables dans `roles/app_k8s/vars/main.yml`
4. Créer Ingress rule si nécessaire
5. Déployer: `ansible-playbook -i inventory.yml site.yml --tags app_k8s`

---

## 🔄 Variables Globales & Configuration

### Hiérarchie Ansible

```
Priority (Highest → Lowest)
│
├─ Extra vars (--extra-vars)
├─ Task vars
├─ Block vars
├─ Host vars (inventory.yml)
├─ Group vars (group_vars/*)
├─ Role defaults (defaults/main.yml)
└─ Built-in defaults
```

### Fichiers de Configuration Clés

- **inventory.yml**: Hotes, groupes, vars globales  
- **group_vars/all/secret.yml**: Vault avec secrets  
- **roles/k8s/defaults/main.yml**: K3s configuration  
- **roles/app_k8s/vars/main.yml**: Helm chart versions  

---

## 📝 Notes Arquitecturales

1. **K3s vs Full Kubernetes**: K3s est plus léger, donc adapté aux VPS petits/moyens
2. **Local-path Storage**: OK pour petite charge, préférer NFS/Ceph pour évoluer
3. **Single Load Balancer**: Traefik K3s natif, préférer metallb/ingress-nginx pour multi-cluster
4. **Secrets**: Stockés en base64 dans etcd K3s, chiffrer le backup etcd en production
5. **Monitoring**: Stack Prometheus + Grafana standard, ajouter AlertManager en prod
6. **Network**: Flannel OK, préférer Cilium pour network policies avancées

---

Voir aussi: [INSTALLATION.md](INSTALLATION.md), [ROLES.md](ROLES.md), [SECURITY.md](SECURITY.md)
