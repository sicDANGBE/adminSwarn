# Rôles Ansible - Documentation Détaillée

## 📑 Index

1. [common](#role-common---hardening-systeme)
2. [k8s](#role-k8s---kubernetes-k3s-cluster)
3. [app_k8s](#role-app_k8s---application-deployment)
4. [runner](#role-runner---ci-cd-runners)
5. [audit_security](#role-audit_security---security-audit)
6. [clean](#role-clean---destructive-cleanup)

---

## Role: common - Hardening Système

### Objectif
Préparer les nœuds pour la production en appliquant les bonnes pratiques de sécurité et configuration système.

**Exécution**: All nodes (k8s_cluster)  
**Tags**: `common`, `system`  
**Handlers**: Restart SSH, Restart Rsyslog, Reload NFTables

### Tâches Principales

#### 1. Mise à Jour Système
```yaml
- name: OS Update
  apt:
    update_cache: yes
    upgrade: dist
```
- Met à jour tous les packages Debian
- Applique les security patches
- Requiert: connexion Internet stable

#### 2. Gestion Utilisateurs
```yaml
- Création de {{ sysadmin_user }} (supadmin)
- SSH public key deployment ({{ local_public_key }})
- Passwordless sudo configuration
- Suppression/verrouillage 'debian' user
```
**Variables**:
- `sysadmin_user`: Nom d'utilisateur admin (défaut: `supadmin`)
- `local_public_key`: Chemin local SSH public key (défaut: `~/.ssh/id_rsa.pub`)

#### 3. SSH Hardening

**Configuration** (`/etc/ssh/sshd_config`):
```ini
Port 49281                      # Non-standard port
PermitRootLogin no             # Root login disabled
PasswordAuthentication no       # Keys only
PubkeyAuthentication yes       # SSH keys enabled
StrictModes yes
X11Forwarding no               # No X11 forwarding
PermitEmptyPasswords no        # No empty passwords
ForkAfterAuthentication no     # Modern SSH
AllowUsers supadmin            # Only specific user
MaxAuthTries 3                 # Rate limiting
```

**Impact**: Service SSH restarts, sessions SSH actuelles restent actives

#### 4. Firewall NFTables

**Structure**:
```
/etc/nftables.d/
├── 10-base.nft          # SSH, DNS, NTP
├── 50-k8s.nft           # Kubernetes specific (added later by k8s role)
└── 99-logging.nft       # Logging rules
```

**Base Rules (nftables_base.j2)**:
```nftables
# Default policy: DROP incoming
chain input {
  type filter hook input priority filter; policy drop;
  
  # Allow loopback
  iif lo accept
  
  # Allow SSH (port 49281)
  tcp dport 49281 accept
  
  # Allow DNS (53)
  udp dport 53 accept
  tcp dport 53 accept
  
  # Allow NTP (123)
  udp dport 123 accept
  
  # Related/established
  ct state { related, established } accept
}
```

**Variables NFTables**:
- `nftables_ssh_port`: SSH port (défaut: 49281)
- `nftables_enabled`: Enable firewall (défaut: true)

**Logging** (nftables_logging.conf.j2):
```nftables
# Log rejected packets to syslog
chain input {
  type filter hook input priority filter; policy drop;
  ...
  # Drop and log
  counter drop log prefix "nftables-drop: "
}
```

#### 5. Fail2Ban (Intrusion Prevention)

**Installation**: `fail2ban`, `fail2ban-systemd`

**Jails configurés**:
- `sshd`: Protège SSH (port 49281)
- `nftables-blacklist`: Utilise NFTables

**Paramètres**:
```ini
[DEFAULT]
bantime = 3600          # 1 heure
findtime = 600          # 10 minutes
maxretry = 5            # 5 tentatives
destemail = admin@...

[sshd]
enabled = true
logpath = /var/log/auth.log
```

#### 6. Logging & Rsyslog Configuration

**Fichiers configurés**:
- `/etc/rsyslog.conf`: Config principale
- `/var/log/auth.log`: SSH attempts
- `/var/log/NFTables.log`: Firewall rejections
- `/var/log/fail2ban.log`: Intrusion attempts

#### 7. Système Modules

**Modules chargés permanently**:
```bash
overlay       # Container runtime support
br_netfilter  # Kubernetes networking
nf_conntrack  # Connection tracking
```

### Dépendances

- **Avant**: Aucun prérequis
- **Après**: Applicable avant `k8s` role

### Sorties/État Modifié

- ✅ SSH hardened et accessible sur port 49281
- ✅ Firewall actif (NFTables)
- ✅ User `supadmin` créé avec accès passwordless
- ✅ Fail2Ban en cours d'exécution
- ✅ System logging opérationnel
- ✅ OS packages à jour

---

## Role: k8s - Kubernetes (K3s) Cluster

### Objectif
Installer et configurer un cluster Kubernetes K3s (lightweight distribution) composé d'un control plane et de workers.

**Exécution**: All nodes (k8s_cluster)  
**Tags**: `k8s`, `kubernetes`  
**Handlers**: Reload NFTables  
**Mode**: Ansible automatically detects server vs agents

### Tâches Principales

#### 1. Préparation Système

**Disable Swap**:
```bash
# Kubernetes requirement
swapoff -a
# Persist in /etc/fstab
sed -i '/ swap / s/^/#/' /etc/fstab
```

**Kernel Modules** (reuse common role modules):
```bash
modprobe overlay
modprobe br_netfilter
```

**Kernel Parameters**:
```sysctl
vm.overcommit_memory = 1
kernel.panic = 10
kernel.panic_on_oops = 1
```

#### 2. Installation K3s Server (Control Plane)

**Hosts**: `server` group (ovh.core)  
**URL**: https://get.k3s.io

**Variables de Configuration**:
| Variable | Default | Description |
|----------|---------|---|
| `k3s_version` | (latest) | K3s version tag |
| `k3s_flannel_iface` | eth0 | Flannel interface |
| `k3s_extra_server_args` | "" | Extra K3s args |

**Script Installation**:
```bash
curl -sfL https://get.k3s.io | \
  INSTALL_K3S_VERSION={{ k3s_version }} \
  K3S_FLANNEL_IFACE={{ k3s_flannel_iface }} \
  K3S_TOKEN={{ k3s_token }} \
  sh -
```

**Composants Installés Automatiquement**:
- API Server (6443)
- Scheduler
- Controller Manager
- etcd (embedded)
- CoreDNS
- **Traefik** (Ingress Controller)
- **Local-path Provisioner** (Storage)
- Metrics Server

#### 3. Installation K3s Agent (Workers)

**Hosts**: `agents` group (ovh.worker.01, ovh.worker.02)

**Configuration**:
```bash
K3S_URL=https://{{ server_ip }}:6443
K3S_TOKEN={{ k3s_token }}
curl -sfL https://get.k3s.io | sh -
```

**Rôle Worker**:
- Kubelet (node agent)
- Kube-proxy (networking)
- Container runtime (containerd)

#### 4. Kubectl Access Configuration

**Fichier**: `/etc/rancher/k3s/k3s.yaml`
- Autorisations pour localhost access
- Token authentification
- API Server URL: `https://127.0.0.1:6443`

**Export pour Ansible Use**:
```yaml
- name: Copy kubeconfig
  fetch:
    src: /etc/rancher/k3s/k3s.yaml
    dest: /tmp/k3s.yaml
    flat: yes
```

#### 5. Traefik Configuration (ACME/TLS)

**Fichier Template**: `traefik-config.yaml.j2`

**ACME Configuration**:
```yaml
providers:
  kubernetesIngress:
    ingressClass: traefik

certificatesResolvers:
  letsencrypt:
    acme:
      email: admin@{{ main_domain }}
      storage: /data/acme.json
      httpChallenge:
        entryPoint: web
      tlsChallenge: {}
```

**Storage**: `/var/lib/rancher/k3s/server/manifests/traefik-config.yaml`

#### 6. CNI Configuration (Flannel)

**Interface**: Configurée via `k3s_flannel_iface` (default: eth0)  
**Pod CIDR**: 10.42.0.0/16  
**Service CIDR**: 10.43.0.0/16

**Troubleshooting**: Si pods ne peuvent pas communiquer, vérifier l'interface correcte

#### 7. NFTables K8s Rules

**Fichier Template**: `nftables_k8s.conf.j2`

**Ports Ouvertes**:
```nftables
# API Server (control plane only)
tcp dport 6443 accept

# Kubelet (tous les nœuds)
tcp dport 10250 accept

# NodePort services (30000-32767)
tcp dport 30000-32767 accept
udp dport 30000-32767 accept

# Pod CIDR (cluster-internal)
ip saddr 10.42.0.0/16 accept
ip daddr 10.42.0.0/16 accept
```

### Dépendances

- **Avant**: `common` role (pour SSH, firewall, users)
- **Après**: Kubernetes API accessible, `app_k8s` peut déployer

### Variables Clés (defaults/main.yml)

```yaml
k3s_version: "v1.28.0"              # K3s version
k3s_flannel_iface: "eth0"           # CNI interface
k3s_token: "{{ lookup('password', '/tmp/k3s_token') }}"  # Secure token
k3s_extra_server_args: ""           # Extra K3s args
kubeconfig_path: "/etc/rancher/k3s/k3s.yaml"
```

### Sorties/État Modifié

- ✅ K3s server installé (control plane)
- ✅ K3s agents installés (workers)
- ✅ Cluster opérationnel (pods en cours d'exécution)
- ✅ Traefik configuré avec ACME
- ✅ kubectl accessible sur tous les nœuds
- ✅ NFTables firewall mise à jour pour K8s

### Commandes Utiles Après Déploiement

```bash
# Vérifier l'état du cluster
kubectl get nodes
kubectl get pods -A

# Voir les services
kubectl get svc -A

# Vérifier Traefik
kubectl get ingress -A

# Voir les logs
kubectl logs -n kube-system -l app=traefik

# Accéder au cluster depuis externe
scp supadmin@ovh.core:/etc/rancher/k3s/k3s.yaml ~/.kube/config-k3s
export KUBECONFIG=~/.kube/config-k3s
```

---

## Role: app_k8s - Application Deployment

### Objectif
Déployer et configurer 9 applications Kubernetes via Helm charts dans un cluster K3s opérationnel.

**Exécution**: `server` only (control plane)  
**Tags**: `app_k8s`  
**Dépendance**: K3s cluster must be running

### Architecture Déploiement

```
app_k8s/
├── tasks/
│   ├── main.yml           # Orches principal
│   ├── monitoring.yml     # Prometheus + Grafana
│   ├── mqtt.yml          # MQTT Broker
│   ├── n8n.yml           # N8N Workflow
│   ├── nodered.yml       # Node-RED
│   ├── ollama.yml        # Ollama LLM
│   ├── portainer.yml     # Portainer UI
│   ├── postgres.yml      # PostgreSQL
│   ├── rabbitmq.yml      # RabbitMQ
│   └── traefik.yml       # Traefik Config (optionnel)
├── vars/
│   └── main.yml          # Versions Helm, versions
└── templates/
    └── (If needed for values generation)
```

### Tâches Principales

#### 0. DNS Configuration Fix

**Problème**: Après K3s install, `/etc/resolv.conf` peut être overwritten  
**Solution**:
```yaml
- name: Fix DNS
  copy:
    dest: /etc/resolv.conf
    content: |
      nameserver 1.1.1.1      # Cloudflare
      nameserver 8.8.8.8      # Google
      nameserver 213.186.33.99 # OVH
```

#### 1. Helm Installation

**Source**: Official Helm script  
**URL**: `https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3`  
**Version**: v3.x (v4 doesn't exist yet)

```bash
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

#### 2. Python Dependencies

**Modules Requis** (pour Ansible K8s):
```bash
apt install python3-kubernetes python3-jsonpatch python3-yaml
```

Permet à Ansible d'interagir avec l'API K8s:
- `kubernetes.core.helm`: Deploy Helm charts
- `kubernetes.core.k8s`: Create resources (Namespaces, Ingress)

#### 3. Namespace Creation

**Namespaces**:
```yaml
- portainer     # Portainer CE
- apps          # Main applications
- admin         # Administrative tools
- monitoring    # Prometheus + Grafana
- ollama-system # Ollama LLM
```

```yaml
- name: Create Namespaces
  kubernetes.core.k8s:
    kubeconfig: /etc/rancher/k3s/k3s.yaml
    name: "{{ item }}"
    api_version: v1
    kind: Namespace
    state: present
  loop:
    - portainer
    - apps
    - admin
    - monitoring
    - ollama-system
```

#### 4. Helm Chart Deployments

**Module Defaults**:
```yaml
module_defaults:
  group/kubernetes.core.helm:
    kubeconfig: /etc/rancher/k3s/k3s.yaml
    wait: true
    wait_timeout: 5m
```

**Chaque Chart Importé**:
```yaml
- import_tasks: portainer.yml
- import_tasks: postgres.yml
- import_tasks: rabbitmq.yml
- import_tasks: n8n.yml
- import_tasks: nodered.yml
- import_tasks: mqtt.yml
- import_tasks: monitoring.yml
- import_tasks: ollama.yml
```

### Applications Déployées

#### Portainer CE
```yaml
Chart: portainer/portainer
Namespace: portainer
Version: 2.21.5
Ingress: portainer.dgsynthex.online
Storage: 10Gi PVC
Features:
  - Web UI pour gérer K8s
  - Stack management
  - Registry management
```

#### PostgreSQL
```yaml
Chart: Bitnami/postgresql
Namespace: apps
Version: Latest Bitnami
Storage: 20Gi PVC
Credentials: Vault-stored password
Access: Internal K8s DNS (postgresql.apps.svc.cluster.local)
```

#### RabbitMQ
```yaml
Chart: Bitnami/rabbitmq
Namespace: apps
Version: 12.0.0
Port: 5672 (AMQP), 15672 (Management UI)
Ingress: rabbitmq.dgsynthex.online
Storage: 10Gi PVC
Features:
  - Message broker
  - Management dashboard
  - HA clustering (optional)
```

#### N8N (Workflow Automation)
```yaml
Chart: n8n-io/n8n ou community chart
Namespace: apps
Version: 0.14.0
Ingress: n8n.dgsynthex.online
Storage: 10Gi PVC (/home/node/.n8n)
Features:
  - Workflow visual editor
  - 400+ integrations
  - Database backend (PostgreSQL)
```

#### Node-RED
```yaml
Chart: Node-RED community chart
Namespace: apps
Ingress: nodered.dgsynthex.online
Storage: 5Gi PVC (/data)
Features:
  - Visual function orchestration
  - MQTT/IoT ready
  - Flow editing UI
```

#### MQTT Broker (Mosquitto)
```yaml
Chart: Bitnami/mosquitto
Namespace: apps
Version: 4.0.0
Port: 1883 (MQTT), 8883 (MQTT+TLS), 9001 (WebSocket)
Ingress: mqtt.dgsynthex.online
Features:
  - IoT messaging protocol
  - Topic-based pub/sub
  - TLS support
```

#### Ollama (LLM Inference)
```yaml
Chart: Custom/Ollama ou community
Namespace: ollama-system
Ingress: ollama.dgsynthex.online
Storage: 50Gi PVC (/root/.ollama - models)
Features:
  - Local LLM inference
  - REST API
  - Model caching
Note: Requires GPU if available
```

#### Prometheus + Grafana (kube-prometheus-stack)
```yaml
Chart: prometheus-community/kube-prometheus-stack
Namespace: monitoring
Version: 45.0.0
Components:
  - Prometheus (metrics scraping)
  - AlertManager (alerting)
  - Grafana (visualization)
  - Node Exporter (host metrics)
Ingress: grafana.dgsynthex.online
Storage:
  - Prometheus: 20Gi PVC
  - Grafana: 5Gi PVC
Features:
  - Cluster metrics
  - Custom dashboards
  - Alert rules
  - ServiceMonitor discovery
```

#### Traefik Configuration (Native K3s)
```yaml
Already deployed by K3s
Ingress Controller: enabled
ACME: Configured in k8s role
CertificateResolvers: letsencrypt
Middleware: path stripping, compression
```

### Variables de Configuration (vars/main.yml)

```yaml
helm_versions:
  portainer: "2.21.5"
  rabbitmq: "12.0.0"      # Bitnami
  n8n: "0.14.0"           # OCI ou Community
  mosquitto: "4.0.0"      # Bitnami
  monitoring: "45.0.0"    # kube-prometheus-stack

storage_class: "local-path"  # K3s default

# Application-specific config
k8s_namespace_apps: "apps"
k8s_namespace_portainer: "portainer"
k8s_namespace_monitoring: "monitoring"
k8s_namespace_ollama: "ollama-system"

# Helm repos (auto-added)
helm_repos:
  - name: portainer
    url: https://portainer.github.io/helm/
  - name: bitnami
    url: https://charts.bitnami.com/bitnami
  - name: prometheus-community
    url: https://prometheus-community.github.io/helm-charts
```

### Dépendances

- **Avant**: K3s cluster running with Traefik
- **Après**: All apps accessible via *.dgsynthex.online

### Sorties/État Modifié

- ✅ Helm installé
- ✅ Namespaces créés
- ✅ 9 applications déployées
- ✅ Ingress rules créées
- ✅ TLS certificates issued (Let's Encrypt)
- ✅ PVCs provisionnés
- ✅ Services pour inter-app communication

---

## Role: runner - CI/CD Runners

### Objectif
Déployer des agents CI/CD et runners d'automatisation pour pipelines Kubernetes.

**Exécution**: `server` (control plane)  
**Tags**: `k8s_runner`

### Tâches Principales

#### 1. Template Processing

**Fichier Template**: `deployment.yaml.j2`  
**Destination**: Generated deployment manifest

#### 2. Runner Registration

- Enregistrement avec système CI/CD (GitLab Runners, GitHub Actions, etc.)
- Configuration token/credentials
- Labels pour job selection

#### 3. Deployment Creation

- Création pod runner sur K8s
- Mount volumes si nécessaire
- Resource requests/limits

### Voir INSTALLATION.md pour détails runners

---

## Role: audit_security - Security Audit

### Objectif
Générer un rapport de conformité sécurité post-déploiement.

**Exécution**: All nodes (k8s_cluster)  
**Tags**: `never`, `audit` (never automatic)  
**Gather Facts**: true

### Tâches de Rapport

1. **Kernel Parameters Report**
   - vm.swappiness
   - panic settings
   - Memory overcommit
   - File descriptor limits

2. **User Accounts Report**
   - Active users in `/etc/passwd`
   - sudo privileges
   - SSH keys
   - Locked accounts

3. **Firewall Rules Report**
   - NFTables rules (base, k8s, logging)
   - Open ports
   - Connection states

4. **SSH Configuration Report**
   - `/etc/ssh/sshd_config` settings
   - Authentication methods
   - Port configuration

5. **Docker/Container Configuration**
   - containerd config
   - Runtime settings
   - Storage driver

6. **Compliance Report**
   - Generated locally in `audit_reports/`
   - Timestamp included
   - Summary of findings

### Exécution

```bash
# Run audit
ansible-playbook -i inventory.yml site.yml --tags audit

# View reports
ls -la audit_reports/
cat audit_reports/security_audit_*.json
```

---

## Role: clean - Destructive Cleanup

### ⚠️ WARNING: IRREVERSIBLE

### Objectif
Reset complet du cluster (utiliser UNIQUEMENT pour dev/testing).

**Exécution**: All nodes  
**Tags**: `never`, `clean` (never automatic)

### Tâches

1. **Stop K3s Services**
   - `/usr/local/bin/k3s-killall.sh` (servers)
   - `/usr/local/bin/k3s-agent-killall.sh` (agents)

2. **Remove K3s Directories**
   - `/etc/rancher/k3s/`
   - `/var/lib/rancher/`
   - `/var/lib/kubelet/`
   - `/opt/k3s/`

3. **Remove Container Images**
   - Force delete all K3s images
   - Clean containerd storage

4. **Unmount Volumes**
   - Force unmount all bound volumes
   - Remove local-path storage

5. **Remove systemd Services**
   - k3s.service
   - k3s-agent.service

### Exécution

```bash
# Confirm you really want this!
ansible-playbook -i inventory.yml site.yml --tags clean --check  # Dry-run

# Actually execute
ansible-playbook -i inventory.yml site.yml --tags clean
```

---

## Variables Héritées (Group Vars)

Tous les rôles peuvent accéder aux variables globales dans `group_vars/all/`:

### Obligatoires
- `sysadmin_user`: Utilisateur admin (défaut: supadmin)
- `main_domain`: Domaine des Ingress (défaut: dgsynthex.online)

### Secrets (secret.yml - Vault)
- `github_token`: GitHub API token
- `postgres_password`: PostgreSQL root password
- `grafana_admin_password`: Grafana admin password
- `rabbitmq_password`: RabbitMQ guest password
- `n8n_admin_password`: N8N admin password

### Optionnelles
- `ansible_port`: SSH port (défaut: 49281)
- `k3s_version`: K3s version (défaut: latest)
- `ansible_become`: Use sudo (défaut: yes)

---

## Matrice Dépendances Rôles

```
common
  ↓
k8s (depends on common for users, firewall)
  ↓
app_k8s (depends on k8s for running cluster)
  ↓
runner (depends on app_k8s for infrastructure)

audit_security (can run on any state)
clean (destructive, independent)
```

---

Voir aussi: [INSTALLATION.md](INSTALLATION.md), [APPLICATION_DEPLOYMENT.md](APPLICATION_DEPLOYMENT.md), [VARIABLES.md](VARIABLES.md)
