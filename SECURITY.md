# Sécurité - Architecture & Bonnes Pratiques

## 🔐 Principes Généraux

### Defense in Depth (Défense en Profondeur)

Plusieurs couches de sécurité pour minimiser risque single point of failure:

```
Layer 1: OS-level Security
├─ SSH hardening (key-only, port 49281)
├─ NFTables firewall (stateful)
├─ Fail2Ban (intrusion prevention)
└─ OS updates & patching

Layer 2: Kubernetes Security
├─ Network Policies (pod isolation)
├─ RBAC (role-based access control)
├─ Pod Security Policies
└─ Secret encryption in etcd

Layer 3: Application Security
├─ TLS/HTTPS (Let's Encrypt)
├─ API authentication/authorization
├─ Input validation
└─ Secrets management
```

---

## 🛡️ SSH Hardening

### Configuration Déployée

**Fichier**: `/etc/ssh/sshd_config`

```ini
Port 49281
AddressFamily any
ListenAddress 0.0.0.0
ListenAddress ::

# ========== AUTHENTICATION ==========
PermitRootLogin no                    # Disable root login
PubkeyAuthentication yes              # Use SSH keys
PasswordAuthentication no             # NO password auth
PermitEmptyPasswords no              # No blank passwords
KbdInteractiveAuthentication no      # No keyboard-interactive

# ========== SECURITY POLICY ==========
StrictModes yes                       # Check permissions on ~/.ssh
AllowUsers supadmin                  # ONLY this user can SSH
MaxAuthTries 3                       # Fail after 3 attempts
MaxSessions 5                        # Max concurrent sessions
ClientAliveInterval 300              # Idle timeout 5 min
ClientAliveCountMax 2                # Kill after 2 idle intervals

# ========== NO X11/PORT FORWARDING ==========
X11Forwarding no
AllowTcpForwarding no
AllowAgentForwarding no
PermitTunnel no

# ========== MODERN FEATURES ==========
ForkAfterAuthentication no           # Modern SSH server behavior
```

### Implémentation Ansible

```yaml
# roles/common/tasks/main.yml
- name: SSH Hardening
  lineinfile:
    path: /etc/ssh/sshd_config
    regexp: "^#?{{ item.key }}"
    line: "{{ item.key }} {{ item.value }}"
  loop:
    - key: "Port"
      value: "49281"
    - key: "PermitRootLogin"
      value: "no"
    - key: "PasswordAuthentication"
      value: "no"
    - key: "PubkeyAuthentication"
      value: "yes"
    - key: "AllowUsers"
      value: "supadmin"
  notify: Restart SSH
```

### Bonnes Pratiques

✅ **À FAIRE**:
- Sauvegarder SESSION SSH avant redémarrer SSH
- Tester connexion dans second terminal avant fermer premier
- Garder backup de clés SSH privées (fichier sécurisé)
- Utiliser SSH keys avec passphrases fortes

❌ **À ÉVITER**:
- Permettre SSH password login
- Permettre SSH root login
- Utiliser port standard 22 (target évident pour bots)
- Partager SSH keys entre équipe (utiliser bastion/jump host)

---

## 🔥 Firewall NFTables

### Architecture Modulaire

```
/etc/nftables.d/
├── 10-base.nft          # SSH, DNS, NTP
├── 50-k8s.nft           # Kubernetes (ajouté par k8s role)
└── 99-logging.nft       # Logging rules
```

Chaque fichier est include dans `/etc/nftables.conf`:
```bash
include "/etc/nftables.d/*.nft"
```

### Base Rules (10-base.nft)

```nftables
table inet firewall {
  # INPUT CHAIN
  chain input {
    type filter hook input priority filter; policy drop;
    
    # Loopback toujours accepté
    iif lo accept
    
    # SSH (port 49281)
    tcp dport 49281 accept
    
    # DNS (53)
    udp dport 53 accept
    tcp dport 53 accept
    
    # NTP (123)
    udp dport 123 accept
    
    # ICMP (ping, path MTU discovery)
    icmp type { echo-request } accept
    icmpv6 type { echo-request } accept
    
    # Related/established connections
    ct state { related, established } accept
    
    # Invalid packets
    ct state invalid drop
    
    # Reject everything else + log
    counter drop log prefix "nftables-drop: "
  }
  
  # OUTPUT CHAIN (allow all outbound by default)
  chain output {
    type filter hook output priority filter; policy accept;
  }
}
```

### Kubernetes Rules (50-k8s.nft)

Ajouté automatiquement par `k8s` role:

```nftables
table inet firewall {
  chain input {
    # API Server (control plane)
    tcp dport 6443 accept comment "K3s API Server"
    
    # Kubelet (all nodes)
    tcp dport 10250 accept comment "Kubelet API"
    
    # NodePort services (30000-32767)
    tcp dport 30000-32767 accept comment "NodePort Services"
    udp dport 30000-32767 accept comment "NodePort Services"
    
    # Pod CIDR traffic (cluster-internal)
    ip saddr 10.42.0.0/16 accept comment "Pod CIDR"
    ip daddr 10.42.0.0/16 accept comment "Pod CIDR"
    
    # Service CIDR traffic
    ip saddr 10.43.0.0/16 accept comment "Service CIDR"
    ip daddr 10.43.0.0/16 accept comment "Service CIDR"
    
    # Flannel VXLAN (port 8472)
    udp dport 8472 accept comment "Flannel VXLAN"
  }
}
```

### Management Firewall

```bash
# Vérifier règles actives
sudo nftables list ruleset

# Vérifier table inet firewall
sudo nftables list table inet firewall

# Test: Vérifier port 49281 ouvert
sudo nftables list set inet firewall allowed_ssh_ports

# Recharger après modification
sudo systemctl reload nftables

# Logs de rejets
sudo tail -f /var/log/syslog | grep nftables-drop
```

### Bonnes Pratiques NFTables

✅ **À FAIRE**:
- Commencer par `policy drop` (deny by default)
- Log rejected packets pour audit
- Stateful rules (allow related/established)
- Modulaire (fichiers séparés par fonctionnalité)

❌ **À ÉVITER**:
- `policy accept` (allow by default)
- Acces permissive à ports sensibles
- Permettre toutes IPs sans restriction
- Log trop verbose (impacts performance)

---

## 🚫 Fail2Ban - Intrusion Prevention

### Configuration

```ini
# /etc/fail2ban/jail.local
[DEFAULT]
bantime = 3600           # Ban for 1 hour
findtime = 600           # Look at past 10 minutes
maxretry = 5             # Ban after 5 failures
destemail = admin@dgsynthex.online
sendername = Fail2Ban

# Action to take: send email + ban IP
action = %(action_mwl)s

[sshd]
enabled = true
port = 49281
filter = sshd
logpath = /var/log/auth.log
maxretry = 3             # SSH: stricter (3 failures)
bantime = 7200           # SSH: longer ban (2 hours)

[nftables-blacklist]
enabled = true
filter = nftables-blacklist
logpath = /var/log/syslog
action = nftables-blacklist
```

### Monitoring

```bash
# Voir blocked IPs
sudo fail2ban-client status sshd

# Unban une IP
sudo fail2ban-client set sshd unbanip <IP>

# Logs
sudo tail -f /var/log/fail2ban.log
```

---

## 🔑 Secrets Management

### Ansible Vault

Tous les secrets sensibles sont stockés chiffrés dans:
```
group_vars/all/secret.yml
```

**Sécurisation**:
- ✅ Fichier chiffré (AES256)
- ✅ Password-protected
- ✅ Versionnage GIT OK (fichier chiffré)
- ✅ No hardcoded passwords

**Utilisation**:
```bash
# Créer
ansible-vault create group_vars/all/secret.yml

# Éditer
ansible-vault edit group_vars/all/secret.yml

# View
ansible-vault view group_vars/all/secret.yml

# Exécuter playbook
ansible-playbook ... --ask-vault-pass
```

**Bonnes Pratiques**:
- ✅ Utiliser Vault password manager (1Password, Bitwarden) 
- ✅ Store vault password sécurisé sur CI/CD
- ✅ Rotate passwords régulièrement
- ❌ Hardcoder password dans `.vault_pass` (insécurisé)
- ❌ Commit fichier secret.yml sans chiffrage

### Secrets Kubernetes

Dans K8s, secrets stockés en base64 en etcd (NOT encrypted by default):

```bash
# Vérifier secrets
sudo kubectl get secrets -n apps

# View secret (base64)
sudo kubectl get secret postgres-creds -n apps -o yaml

# View decoded
sudo kubectl get secret postgres-creds -n apps \
  -o jsonpath='{.data.password}' | base64 -d; echo
```

**Pour Production**: Activer etcd encryption:

```yaml
# K3s encryption config
apiVersion: apiserver.config.k8s.io/v1
kind: EncryptionConfiguration
resources:
  - resources:
    - secrets
    providers:
    - aescbc:
        keys:
        - name: key1
          secret: <base64-encoded-key>
    - identity: {}
```

---

## 🔐 Kubernetes Security

### Network Policies

Isoler traffic entre pods:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: apps
spec:
  podSelector: {}
  policyTypes:
  - Ingress
  # Aucun ingress ne passe par défaut
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-n8n-from-traefik
  namespace: apps
spec:
  podSelector:
    matchLabels:
      app: n8n
  policyTypes:
  - Ingress
  ingress:
  - from:
    - namespaceSelector:
        matchLabels:
          name: kube-system
    - podSelector:
        matchLabels:
          app: traefik
```

### RBAC (Role-Based Access Control)

Limiter permissions par role:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: n8n-role
  namespace: apps
rules:
- apiGroups: [""]
  resources: ["configmaps", "secrets"]
  verbs: ["get", "list", "watch"]
---
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: n8n-rolebinding
  namespace: apps
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: n8n-role
subjects:
- kind: ServiceAccount
  name: n8n
  namespace: apps
```

### Pod Security Policies

Restreindre capabilities pods:

```yaml
apiVersion: policy/v1beta1
kind: PodSecurityPolicy
metadata:
  name: restricted
spec:
  privileged: false
  allowPrivilegeEscalation: false
  requiredDropCapabilities:
  - ALL
  volumes:
  - 'configMap'
  - 'emptyDir'
  - 'projected'
  - 'secret'
  - 'downwardAPI'
  - 'persistentVolumeClaim'
  hostNetwork: false
  hostIPC: false
  hostPID: false
  runAsUser:
    rule: 'MustRunAsNonRoot'
  fsGroup:
    rule: 'RunAsAny'
```

---

## 🔒 TLS/HTTPS Configuration

### Traefik + Let's Encrypt

Automatiquement configuré dans `k8s` role:

```yaml
# traefik-config.yaml.j2
certificatesResolvers:
  letsencrypt:
    acme:
      email: admin@dgsynthex.online
      storage: /data/acme.json
      httpChallenge:
        entryPoint: web
      caServer: "https://acme-v02.api.letsencrypt.org/directory"
```

**Vérifier Certificats**:
```bash
# Voir tous les certificats
sudo kubectl get certificate -A

# Details d'un certificat
sudo kubectl describe certificate -n apps n8n-tls

# Renew manually si besoin
sudo kubectl delete certificate -n apps n8n-tls

# Traefik logs
sudo kubectl logs -n kube-system -l app=traefik | grep acme
```

**Troubleshooting ACME**:
```bash
# Si Let's Encrypt cert ne s'obtient:

# 1. Vérifier DNS résolu
nslookup n8n.dgsynthex.online

# 2. Vérifier Traefik peut accéder port 80/443 (ACME challenge)
sudo kubectl port-forward -n kube-system svc/traefik 80:80 443:443 &
curl -I http://n8n.dgsynthex.online

# 3. Vérifier logs Traefik
sudo kubectl logs -n kube-system -l app=traefik --tail=50

# 4. Reset ACME data si blocké
sudo kubectl delete secret traefik-acme -n kube-system
sudo kubectl delete deployment traefik -n kube-system
# K3s redéploie automatiquement
```

---

## 📋 Audit Logging

### Sources Logging

```
/var/log/auth.log          → SSH attempts, sudo usage
/var/log/syslog            → System events, nftables logs
/var/log/fail2ban.log      → Intrusion attempts
/var/log/kubernetes/       → K8s API server logs
/var/log/audit/            → Kernel audit logs (si enabled)
```

### Enable Kernel Audit

```bash
# Auditer tous les appels système
sudo auditctl -w /etc/ -p wa -k system_config_changes

# Auditer K3s API server
sudo auditctl -w /var/lib/rancher/k3s/ -p wa -k k3s_changes

# Voir audit events
sudo tail -f /var/log/audit/audit.log | grep k3s

# Exporter pour SIEM
sudo ausearch -k k3s_changes
```

### Log Monitoring

```bash
# Watch auth attempts
sudo tail -f /var/log/auth.log | grep -i sshd

# Watch firewall rejections
sudo tail -f /var/log/syslog | grep nftables-drop

# Watch intrusion attempts
sudo tail -f /var/log/fail2ban.log

# Failed Kubernetes API requests
sudo kubectl logs -n kube-system -l component=kube-apiserver | grep error
```

---

## 🔍 Security Checklist

### Host Security
- ✅ SSH keys only (PasswordAuthentication: no)
- ✅ SSH port non-standard (49281)
- ✅ Root login disabled
- ✅ Firewall enabled (NFTables)
- ✅ Fail2Ban enabled
- ✅ OS updates applied
- ✅ Sudo without password (only for automation)

### Kubernetes Security
- ✅ RBAC enabled (K3s default)
- ✅ NetworkPolicies deployed
- ✅ Pod security policies
- ✅ Secret encryption (TODO: enable etcd encryption)
- ✅ TLS/HTTPS enabled
- ✅ Ingress authentication (OAuth/basic auth)

### Application Security
- ✅ Strong passwords (Vault managed)
- ✅ Database user isolation
- ✅ API rate limiting
- ✅ Input validation
- ✅ Session management
- ❌ TODO: WAF (Web Application Firewall)
- ❌ TODO: IDS (Intrusion Detection System)

### Operational Security
- ✅ Audit logging enabled
- ✅ Log retention configured
- ✅ Backup strategy (etcd snapshots)
- ✅ Disaster recovery plan
- ❌ TODO: SIEM integration
- ❌ TODO: Security scanning (SonarQube, Trivy)

---

## 🚨 Incident Response

### SSH Brute Force Detected

```bash
# 1. Check fail2ban status
sudo fail2ban-client status sshd

# 2. See IPs currently banned
sudo fail2ban-client set sshd banip

# 3. View logs
sudo tail -f /var/log/fail2ban.log

# 4. If needed, increase ban time
sudo fail2ban-client set sshd bantime 86400  # 24 hours
```

### Firewall Attack

```bash
# Monitor rejection rate
sudo watch "grep nftables-drop /var/log/syslog | wc -l"

# See most common rejected IPs
sudo grep nftables-drop /var/log/syslog | \
  awk '{print $NF}' | sort | uniq -c | sort -rn | head -10

# Block suspicious IP permanently
sudo nft add rule inet firewall input ip saddr <IP> drop
sudo systemctl save nftables
```

### Kubernetes Compromise

```bash
# Check for suspicious pods
sudo kubectl get pods -A --field-selector=status.phase=Running | grep restart

# Audit user actions
sudo kubectl get events -A --sort-by='.lastTimestamp'

# Revoke compromised tokens
sudo kubectl delete secret <token-secret> -n <namespace>

# Audit what pods accessed
sudo kubectl logs -l app=<app> -A --all-containers=true --timestamps=true
```

---

Voir aussi: [ARCHITECTURE.md](ARCHITECTURE.md), [INSTALLATION.md](INSTALLATION.md), [ROLES.md](ROLES.md)
