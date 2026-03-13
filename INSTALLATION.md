# Installation & Déploiement - Guide Complet

## 📋 Prérequis

### Infrastructure

- **3 Nœuds VPS OVH** (ou équivalent cloud)
  - Control Plane: 2 vCPU, 4GB RAM, 50GB Disk minimum
  - Workers (x2): 2 vCPU, 4GB RAM, 50GB Disk minimum
- **Domaine DNS**: `dgsynthex.online` pointant sur Traefik Ingress

### Localement (Ansible Controller)

```bash
# Debian/Ubuntu
sudo apt install ansible python3 python3-pip

# ou macOS
brew install ansible

# Vérifier version
ansible --version  # Minimum 2.10

# Dépendances Python
pip3 install requests pyyaml kubernetes jinja2
```

### SSH Access Configuration

```bash
# Générer clés SSH si manquantes
ssh-keygen -t rsa -b 4096 -f ~/.ssh/id_rsa -N ""

# Tester access avant déploiement
ssh -p 49281 -i ~/.ssh/id_rsa supadmin@vps-940a692a.vps.ovh.net "echo 'SSH OK'"
```

---

## 🔐 Configuration Initiale

### Étape 1: Préparer le Vault Secrets

```bash
# Créer directory
mkdir -p group_vars/all

# Créer fichier Vault (vous demande password)
ansible-vault create group_vars/all/secret.yml

# Ajouter les variables:
```

Contenu `secret.yml`:
```yaml
---
# Database
postgres_user: postgres
postgres_password: "ChooseStrongPassword123!"

# n8n
n8n_admin_email: "admin@dgsynthex.online"
n8n_admin_password: "N8NAdminPass123!"
n8n_encryption_key: "e8Xj4kL9mN2pQ7rT4vW6"

# RabbitMQ
rabbitmq_username: guest
rabbitmq_password: "RabbitPass123!"
rabbitmq_erlang_cookie: "ERLANGCOOKIE123456"

# Grafana
grafana_admin_user: admin
grafana_admin_password: "GrafanaPass123!"

# GitHub (optionnel)
github_token: "ghp_xxxxxxxxxxxxx"
```

### Étape 2: Configuration Inventory

Vérifier `inventory.yml` avec vos détails OVH:

```yaml
all:
  vars:
    sysadmin_user: supadmin
    ansible_user: supadmin
    ansible_port: 49281
    ansible_python_interpreter: /usr/bin/python3
    ansible_become: yes
    main_domain: dgsynthex.online
    local_public_key: "~/.ssh/id_rsa.pub"

  children:
    k8s_cluster:
      children:
        server:
          hosts:
            ovh.core:
              ansible_host: vps-940a692a.vps.ovh.net
        agents:
          hosts:
            ovh.worker.01:
              ansible_host: vps-255dab72.vps.ovh.net
            ovh.worker.02:
              ansible_host: vps-9e3ed523.vps.ovh.net
```

Les IP/FQDN doivent être exactes pour SSH.

### Étape 3: Tester Connectivité

```bash
# Tester tous les hôtes
ansible -i inventory.yml k8s_cluster -m ping

# Résultat attendu:
# ovh.core | SUCCESS => { "ping": "pong" }
# ovh.worker.01 | SUCCESS => { "ping": "pong" }
# ovh.worker.02 | SUCCESS => { "ping": "pong" }

# Si ça échoue, vérifier:
# - SSH keys déployées?
# - Firewall OVH permettant connexions?
# - ansible_host = IP/FQDN correct?
```

---

## 🚀 Déploiement Par Étapes

### **Phase 1: Infrastructure Système** (30-45 minutes)

**Objectif**: Durcir le système, setup SSH, firewall, utilisateurs

```bash
# 1. Vérifier syntaxe
ansible-playbook -i inventory.yml site.yml \
  --tags common --syntax-check

# 2. Dry-run (montre ce qui serait changé)
ansible-playbook -i inventory.yml site.yml \
  --tags common --check

# 3. Exécuter réellement
ansible-playbook -i inventory.yml site.yml \
  --tags common --ask-vault-pass
```

**Qu'il se passe**:
- Update OS packages
- Configure SSH hardening (port 49281)
- Setup user `supadmin` avec SSH keys
- Deploy NFTables firewall
- Enable Fail2Ban
- Configure syslog/rsyslog

**Vérifier après**:
```bash
# SSH doit fonctionner avec clés uniquement
ssh -p 49281 -i ~/.ssh/id_rsa supadmin@ovh.core "uname -a"

# Firewall actif
ssh -p 49281 supadmin@ovh.core "sudo nftables -v"

# Utilisateurs configurés
ssh -p 49281 supadmin@ovh.core "id"
```

⚠️ **CRITIQUE**: Si vous êtes SSH actuellement, NE FERMEZ PAS la session! Utilisez deux terminaux.

---

### **Phase 2: Kubernetes Cluster** (20-40 minutes)

**Objectif**: Installer K3s (control plane + workers)

```bash
# 1. Syntaxe check
ansible-playbook -i inventory.yml site.yml \
  --tags k8s --syntax-check

# 2. Exécuter
ansible-playbook -i inventory.yml site.yml \
  --tags k8s --ask-vault-pass
```

**Qu'il se passe**:
- Disable swap
- Load kernel modules
- Install K3s server (control plane)
- Install K3s agents (workers)
- Configure Traefik ACME
- Setup NFTables Kubernetes rules

**Vérifier après**:
```bash
# SSH to control plane
ssh -p 49281 supadmin@ovh.core

# Vérifier K3s running
sudo systemctl status k3s

# Vérifier kubeconfig
sudo ls -la /etc/rancher/k3s/k3s.yaml

# Vérifier kubectl
sudo kubectl get nodes
# Résultat:
# NAME          STATUS   ROLES                  AGE
# ovh.core      Ready    control-plane,master   2m
# ovh.worker.01 Ready    <none>                 1m
# ovh.worker.02 Ready    <none>                 1m

# Vérifier cluster health
sudo kubectl get pods -A
```

**Attendre**: Tous les pods CoreDNS, metrics-server doivent être en `Running`

---

### **Phase 3: Applications Kubernetes** (25-60 minutes)

**Objectif**: Déployer 9 applications via Helm

```bash
# 1. Syntaxe check
ansible-playbook -i inventory.yml site.yml \
  --tags app_k8s --syntax-check

# 2. Exécuter
ansible-playbook -i inventory.yml site.yml \
  --tags app_k8s --ask-vault-pass
```

**Qu'il se passe**:
- Install Helm
- Create namespaces
- Add Helm repositories
- Deploy 9 Helm charts
- Create Ingress rules
- Let's Encrypt certificates auto-issued

**Vérifier après**:
```bash
# Listing tous les pods
sudo kubectl get pods -A

# Vérifier les ingress
sudo kubectl get ingress -A

# Vérifier les services
sudo kubectl get svc -A

# Attendre TLS certificates
sudo kubectl get ingress apps-traefik -A
# Certificat "acme-tls-123" doit exister

# Vérifier Portainer est up
sudo kubectl logs -n portainer -l app=portainer | head -20
```

**Ingress Endpoints Accessibles**:
- https://portainer.dgsynthex.online
- https://traefik.dgsynthex.online (dashboard)
- https://n8n.dgsynthex.online
- https://nodered.dgsynthex.online
- https://rabbitmq.dgsynthex.online (15672)
- https://mqtt.dgsynthex.online
- https://ollama.dgsynthex.online
- https://grafana.dgsynthex.online

---

### **Phase 4: CI/CD Runners** (Optionnel, 10-15 minutes)

```bash
ansible-playbook -i inventory.yml site.yml \
  --tags k8s_runner --ask-vault-pass
```

Déploie des runners pour GitLab/GitHub actions.

---

## 🎯 Déploiement Complet en Une Fois

Pour déploiement initial de zéro:

```bash
# Full deployment (common + k8s + app_k8s)
ansible-playbook -i inventory.yml site.yml \
  --tags common,k8s,app_k8s \
  --ask-vault-pass
```

**Durée totale**: ~90-150 minutes  
**Réseau requis**: Stable (téléchargements 5-10GB ISO K3s, images Docker)

---

## ⚠️ Erreurs Courantes & Solutions

### Erreur: "Permission denied (publickey)"

```
fatal: [ovh.core]: FAILED! => {
  "msg": "Authentication or permission failure."
}
```

**Solution**:
```bash
# Vérifier SSH key
ls -la ~/.ssh/id_rsa

# Assurez-vous key est dans ssh-agent
ssh-add ~/.ssh/id_rsa

# Test direct
ssh -vvv -p 49281 -i ~/.ssh/id_rsa supadmin@vps-940a692a.vps.ovh.net
```

### Erreur: "NFTables: port already in use"

```
ERROR: nftables rule conflict on port 49281
```

**Solution**: K3s service était déjà running
```bash
ssh -p 49281 supadmin@ovh.core "sudo systemctl stop k3s"
# Puis réessayer
```

### Erreur: "Swap is enabled"

K3s refuse démarrer si swap actif.

**Solution**:
```bash
ssh supadmin@ovh.core
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab
```

### Erreur: "helm not found"

```
bin/helm: No such file or directory
```

**Solution**: Réinstaller Helm manuellement
```bash
ssh supadmin@ovh.core
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash
helm version
```

### Pods ne démarrent pas (ImagePullBackOff)

K8s essaie de pull images mais échoue.

**Solution**: Vérifier DNS
```bash
# Sur ovh.core
sudo kubectl exec -it $(sudo kubectl get pod -n kube-system | grep coredns | head -1 | awk '{print $1}') \
  -n kube-system -- dig google.com

# Si échoue, vérifier resolv.conf
sudo cat /etc/resolv.conf

# Rebuild par app_k8s role
```

---

## 🔄 Mises à Jour & Maintenance

### Upgrader K3s

Changer version dans `roles/k8s/defaults/main.yml`:

```yaml
k3s_version: "v1.29.0"  # Au lieu de actuelle
```

Puis réexécuter:
```bash
ansible-playbook -i inventory.yml site.yml \
  --tags k8s --ask-vault-pass
```

K3s upgrade sans downtime (rolling update sur workers).

### Upgrader une Application

Changer version dans `roles/app_k8s/vars/main.yml`:

```yaml
helm_versions:
  n8n: "0.15.0"  # Upgrade
```

Puis:
```bash
ansible-playbook -i inventory.yml site.yml \
  --tags app_k8s --ask-vault-pass
```

Helm sera upgraded si version change.

### Backup Cluster

Sauvegarder etcd (database cluster):

```bash
ssh supadmin@ovh.core
sudo k3s etcd-snapshot save --name backup-$(date +%Y%m%d)
sudo ls -la /var/lib/rancher/k3s/server/db/snapshots/
```

Copier snapshots localement:
```bash
scp -P 49281 -r supadmin@ovh.core:/var/lib/rancher/k3s/server/db/snapshots/ \
  ~/k8s_backups/
```

### Reset Cluster (DESTRUCTIVE)

⚠️ **Ceci delete tout!**

```bash
ansible-playbook -i inventory.yml site.yml \
  --tags clean --ask-vault-pass
```

Puis redéployer:
```bash
ansible-playbook -i inventory.yml site.yml \
  --tags common,k8s,app_k8s --ask-vault-pass
```

---

## 📊 Monitoring Déploiement

### Vérifier Logs en Direct

```bash
# Terminal 1: Watch progress
watch -n 5 'ssh supadmin@ovh.core "sudo kubectl get pods -A"'

# Terminal 2: Stream logs
ssh supadmin@ovh.core "sudo kubectl logs -n apps -l app=n8n -f"
```

### Diagnostic Complet

```bash
# Nodes
sudo kubectl get nodes -o wide

# All pods
sudo kubectl get pods -A --field-selector=status.phase!=Running

# Pod details
sudo kubectl describe pod <pod-name> -n <namespace>

# Events
sudo kubectl get events -A --sort-by=.metadata.creationTimestamp

# Storage
sudo kubectl get pvc -A

# Ingress status
sudo kubectl get ingress -A -o wide
```

---

## 🔐 Post-Déploiement: Sécurisation

### 1. Changer Domaine Let's Encrypt

Éditer `group_vars/all/secret.yml`:
```yaml
letsencrypt_email: "your-email@dgsynthex.online"
```

### 2. Activer NetworkPolicy (Optionnel)

Limiter traffic inter-pods:

```bash
# Créer une restrictive network policy
cat <<EOF | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all
  namespace: apps
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  - Egress
EOF
```

### 3. Configurer RBAC

Créer service accounts avec permissions minimales:

```bash
kubectl create serviceaccount n8n-sa -n apps
kubectl create rolebinding n8n-binding \
  --clusterrole=edit \
  --serviceaccount=apps:n8n-sa
```

### 4. Configurer PodSecurityPolicy (K3s 1.25+)

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  runAsUser:
    rule: 'MustRunAsNonRoot'
  fsGroup:
    rule: 'RunAsAny'
```

---

Voir aussi: [ARCHITECTURE.md](ARCHITECTURE.md), [ROLES.md](ROLES.md), [TROUBLESHOOTING.md](TROUBLESHOOTING.md)
