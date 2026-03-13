# 🚀 adminSwarn - Kubernetes Infrastructure Automation

**Production-Grade IaC for K3s Cluster with 9+ Microservices**

> Automated deployment and management of Kubernetes (K3s) infrastructure on OVH VPS with integrated applications for automation, monitoring, and administration.

---

## 📖 Documentation Index

Commencez par le document approprié selon votre besoin:

### 🆕 **Nouveau au Projet?**
- **[ARCHITECTURE.md](ARCHITECTURE.md)** — Vue d'ensemble complète
  - Architecture infrastructure (3-node cluster)
  - Diagrammes réseau & Kubernetes
  - Hiérarchie des rôles Ansible
  - Flux de déploiement

### 🛠️ **Prêt à Déployer?**
- **[INSTALLATION.md](INSTALLATION.md)** — Guide étape par étape
  - Prérequis infrastructure & locaux
  - Configuration initiale (Vault secrets, inventory)
  - Déploiement par phases (common → k8s → apps)
  - Erreurs courantes & solutions
  - Mises à jour & maintenance

### 📚 **Comprendre les Rôles**
- **[ROLES.md](ROLES.md)** — Documentation détaillée de chaque rôle
  - `common` — Hardening système & sécurité
  - `k8s` — Installation K3s cluster
  - `app_k8s` — Déploiement 9 applications Helm
  - `runner` — CI/CD runners
  - `audit_security` — Compliance reporting
  - `clean` — Destructive reset

### 🔧 **Configuration & Variables**
- **[VARIABLES.md](VARIABLES.md)** — Guide complet des variables
  - Variables globales (inventory.yml)
  - Ansible Vault (secrets management)
  - Variables par rôle
  - Hiérarchie des variables Ansible

### 🔐 **Architecture Sécurité**
- **[SECURITY.md](SECURITY.md)** — Posture sécurité & bonnes pratiques
  - SSH hardening (port 49281, key-only auth)
  - NFTables firewall (modular rules)
  - Fail2Ban intrusion prevention
  - Kubernetes security (RBAC, NetworkPolicies)
  - Audit logging & incident response

### �️ **Sécurité Avancée N8N & Node-RED**
- **[N8N_NODERED_SECURITY.md](N8N_NODERED_SECURITY.md)** — Améliorations sécurité détaillées
  - Rate limiting (Traefik Middleware)
  - Kubernetes RBAC & ServiceAccounts
  - Network policies (egress control)
  - Pod Disruption Budgets (HA)
  - Resource quotas & security contexts
  - Audit logging & webhooks control
  - Best practices & troubleshooting
### 🤖 **MCP (Model Context Protocol) pour N8N**
- **[MCP_N8N_INSTALLATION.md](MCP_N8N_INSTALLATION.md)** — Intégration MCP serveur
  - 3 options d'installation (hébergé, Docker, Kubernetes)
  - Architecture K3s auto-hébergée (recommandée)
  - Configuration Claude/ChatGPT/Cursor
  - Sécurité & best practices
  - Monitoring avec Prometheus/Grafana
  - Processus de déploiement & rollback
### 📦 **Applications Déployées**
- **[APPLICATION_DEPLOYMENT.md](APPLICATION_DEPLOYMENT.md)** — Guide per-application
  - Traefik (Ingress Controller + TLS)
  - PostgreSQL (Database backend)
  - RabbitMQ (Message broker)
  - N8N (Workflow automation) — ✅ Sécurité avancée implémentée
  - Node-RED (Visual orchestration) — ✅ Sécurité avancée implémentée
  - MQTT Broker (IoT messaging)
  - Ollama (Local LLM inference) — ✅ MCP server pour intégration IA
  - Portainer (K8s management UI)
  - Prometheus + Grafana (Monitoring stack)

### 🆘 **Dépannage & Diagnostic**
- **[TROUBLESHOOTING.md](TROUBLESHOOTING.md)** — Erreurs courantes & solutions
  - SSH connection issues
  - K3s service failures
  - Pod deployment errors
  - Database connection problems
  - Firewall/networking issues
  - Performance diagnostics

---

## ⚡ Quick Start

### Déploiement Complet (Zéro à Production)

```bash
# 1. Clone repo et configurer
git clone <repo>
cd ansible

# 2. Créer Ansible Vault
ansible-vault create group_vars/all/secret.yml
# Ajouter: postgres_password, n8n_admin_password, etc.

# 3. Vérifier Inventory
vim inventory.yml
# Vérifier: ansible_host, main_domain, sysadmin_user

# 4. Tester connectivité
ansible -i inventory.yml k8s_cluster -m ping

# 5. FULL DEPLOYMENT (90-150 minutes)
ansible-playbook -i inventory.yml site.yml \
  --tags common,k8s,app_k8s \
  --ask-vault-pass

# 6. Vérifier cluster opérationnel
ssh -p 49281 supadmin@ovh.core
sudo kubectl get nodes
sudo kubectl get pods -A
```

### Accès Applications

```
https://traefik.dgsynthex.online          Dashboard de routage
https://portainer.dgsynthex.online        Gestion K8s UI
https://n8n.dgsynthex.online              Automation workflows
https://nodered.dgsynthex.online          Visual orchestration
https://grafana.dgsynthex.online          Monitoring & metrics
https://rabbitmq.dgsynthex.online         Message broker
https://mqtt.dgsynthex.online             IoT messaging
https://ollama.dgsynthex.online           LLM inference API
```

---

## 🏗️ Structure Projet

```
ansible/
├── site.yml                    # Main orchestrator playbook
├── inventory.yml              # 3-node cluster definition
│
├── README.md                  # Ce fichier
├── ARCHITECTURE.md            # Vue d'ensemble technique
├── INSTALLATION.md            # Guide déploiement étape-par-étape
├── ROLES.md                   # Documentation détaillée rôles
├── VARIABLES.md               # Guide configuration
├── SECURITY.md                # Posture sécurité
├── APPLICATION_DEPLOYMENT.md  # Details applications
└── TROUBLESHOOTING.md         # Debug & solutions
│
├── group_vars/all/
│   ├── secret.yml             # 🔐 Chiffré Vault (credentials)
│   └── secret.yml.example     # Template pour secrets
│
└── roles/
    ├── common/                # OS hardening & base setup
    ├── k8s/                   # Kubernetes (K3s) installation
    ├── app_k8s/               # Application deployment (Helm)
    ├── runner/                # CI/CD runners
    ├── audit_security/        # Security compliance audit
    └── clean/                 # Destructive cluster reset
```

---

## 🎯 Exécution par Cas d'Usage

### 🆕 Déploiement Initial

```bash
# Full deployment (tout en une fois, advisé pour stabilité réseau)
ansible-playbook -i inventory.yml site.yml \
  --tags common,k8s,app_k8s \
  --ask-vault-pass

# OU par étapes distinctes
ansible-playbook -i inventory.yml site.yml --tags common --ask-vault-pass
ansible-playbook -i inventory.yml site.yml --tags k8s --ask-vault-pass
ansible-playbook -i inventory.yml site.yml --tags app_k8s --ask-vault-pass
```

### 🔄 Mise à Jour Partielle

```bash
# Upgrader K3s version
vi roles/k8s/defaults/main.yml  # Change k3s_version
ansible-playbook -i inventory.yml site.yml --tags k8s --ask-vault-pass

# Appliquer hardening system
ansible-playbook -i inventory.yml site.yml --tags common --ask-vault-pass

# Redéployer une application
ansible-playbook -i inventory.yml site.yml --tags app_k8s --ask-vault-pass
```

### 🔍 Audit & Diagnostic

```bash
# Tester sans faire de changements
ansible-playbook -i inventory.yml site.yml --check --ask-vault-pass

# Verbose output (debug)
ansible-playbook -i inventory.yml site.yml -vvv --ask-vault-pass

# Audit sécurité (rapport généré localement)
ansible-playbook -i inventory.yml site.yml --tags audit --ask-vault-pass
```

### 🧹 Reset Destructif

```bash
# ⚠️ IRREVERSIBLE - Resets tout le cluster!
ansible-playbook -i inventory.yml site.yml --tags clean --ask-vault-pass

# Puis redéployer:
ansible-playbook -i inventory.yml site.yml \
  --tags common,k8s,app_k8s \
  --ask-vault-pass
```

---

## 🔐 Variables Clés

### Configuration Essentielles

| Variable | Fichier | Description |
|----------|---------|---|
| `main_domain` | inventory.yml | Domaine pour Ingress (ex: dgsynthex.online) |
| `sysadmin_user` | inventory.yml | Utilisateur admin SSH (ex: supadmin) |
| `ansible_port` | inventory.yml | SSH port (ex: 49281) |
| `k3s_version` | roles/k8s/defaults/main.yml | K3s version (ex: v1.28.0) |
| `storage_class` | roles/app_k8s/vars/main.yml | K3s storage provisioner |
| `postgres_password` | group_vars/all/secret.yml 🔐 | Database password |
| `n8n_admin_password` | group_vars/all/secret.yml 🔐 | N8N admin credential |
| `grafana_admin_password` | group_vars/all/secret.yml 🔐 | Grafana admin credential |

**🔐 Vault-Protected**:  Stockés chiffrés en `group_vars/all/secret.yml`

> Pour plus de détails: [VARIABLES.md](VARIABLES.md)

---

## 💾 Déploiement Recommandé

### Phase 1: Infrastructure Système (30-45 min)
```bash
ansible-playbook -i inventory.yml site.yml --tags common --ask-vault-pass
```
- ✅ SSH hardening, firewall, users, OS updates

### Phase 2: Kubernetes Cluster (20-40 min)
```bash
ansible-playbook -i inventory.yml site.yml --tags k8s --ask-vault-pass
```
- ✅ K3s server (control plane) + agents (workers)
- ✅ Traefik + Flannel networking

### Phase 3: Applications (25-60 min)
```bash
ansible-playbook -i inventory.yml site.yml --tags app_k8s --ask-vault-pass
```
- ✅ Helm installation
- ✅ 9 services: Portainer, PostgreSQL, RabbitMQ, N8N, Node-RED, MQTT, Ollama, Prometheus, Grafana

### Total: ~90-150 minutes

---

## 📱 Prérequis

### Infrastructure
- 3 OVH VPS (ou équivalent)
  - Control Plane: 2vCPU, 4GB RAM, 50GB disk
  - Workers (x2): 2vCPU, 4GB RAM, 50GB disk
- Domaine DNS pointant sur Traefik Ingress

### Local (Ansible Controller)
```bash
# Terraform
ansible >= 2.10
python3 >= 3.8
python3-kubernetes

# Test
ssh -p 49281 supadmin@ovh.core "echo OK"  # SSH must work first
```

---

## 🛡️ Sécurité Intégrée

- ✅ **SSH Hardening**: Key-only auth, custom port (49281), root disabled
- ✅ **Firewall**: NFTables (modular, stateful)
- ✅ **Intrusion Prevention**: Fail2Ban active
- ✅ **Secrets Management**: Ansible Vault (AES256)
- ✅ **HTTPS/TLS**: Let's Encrypt automatique via Traefik
- ✅ **Kubernetes RBAC**: Role-based access control
- ✅ **Audit Logging**: Kernel events, firewall logs, system events

> Détails complets: [SECURITY.md](SECURITY.md)

---

## 📋 État du Projet

### ✅ Complété
- Infrastructure Kubernetes (K3s 3-node cluster)
- 9 applications en production
- SSH/Firewall hardening
- Monitoring (Prometheus + Grafana)
- Documentation complète

### 🔄 En Cours
- Amélioration sécurité N8N
- Amélioration sécurité Node-RED
- Integration MCP (Model Context Protocol) pour N8N

### 🚀 Futur
- Support GPU (NVIDIA)
- High-Availability (etcd clustering)
- Backup automation (etcd snapshots)
- SIEM integration (logs centralisés)
- WAF integration (Web Application Firewall)

---

## 🤝 Contribution

Pour rapporter bugs ou suggérer améliorations:
1. Voir [TROUBLESHOOTING.md](TROUBLESHOOTING.md) pour issues connues
2. Consulter les logs: `sudo kubectl logs -A`
3. Créer issue ou PR

---

## 📞 Support

- **Documentation**: Voir liens ci-dessus (ARCHITECTURE, INSTALLATION, ROLES, etc.)
- **Logs**: `sudo journalctl -u k3s --no-pager` et `sudo kubectl logs -A`
- **Health Check**: `sudo kubectl get nodes` et `sudo kubectl get pods -A`

---

## 📄 License

Projet personnalisé. Disponible pour utilisation interne uniquement.

---

**Dernière mise à jour**: Décembre 2025  
**Maintenu par**: DevOps Team  
**Kubernetes Version**: K3s 1.28+
