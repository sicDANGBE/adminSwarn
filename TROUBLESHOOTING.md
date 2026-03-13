# Dépannage & Diagnostic - Guide Complet

## 🔍 Approche Générale de Dépannage

### 1. Vérifier Logs (TOUJOURS D'ABORD)

```bash
# SSH to failing node
ssh -p 49281 supadmin@ovh.core

# Check systemd logs
sudo journalctl -u k3s -n 50 --no-pager

# Check Ansible logs (si playbook en cours)
tail -f /tmp/ansible.log

# Check specific pod logs
sudo kubectl logs -n <namespace> <pod-name> --tail=100
```

### 2. Vérifier État Système

```bash
# System resources
free -h                              # Memory
df -h                               # Disk
ps aux | grep -E "k3s|docker"       # Processes

# Network
ip addr show                         # IPs
sudo nftables list ruleset           # Firewall
netstat -pln | grep LISTEN          # Open ports

# Kubernetes
sudo kubectl get nodes -o wide       # Node status
sudo kubectl get pods -A --field-selector=status.phase!=Running
sudo kubectl get events -A --sort-by=.metadata.creationTimestamp
```

### 3. Kubectl Debugging Commands

```bash
# Describe problematic resource
sudo kubectl describe pod <pod> -n <ns>
sudo kubectl describe node <node>
sudo kubectl describe ingress <ingress> -n <ns>

# Get resource YAML
sudo kubectl get pod <pod> -n <ns> -o yaml

# Exec into pod
sudo kubectl exec -it <pod> -n <ns> -- /bin/bash

# Port forward for testing
sudo kubectl port-forward -n <ns> svc/<svc> 8000:8000 &
curl http://localhost:8000
```

---

## 🆘 Erreurs Courantes & Solutions

### ❌ SSH Connection Refused

**Symptôme**:
```
ssh: connect to host vps-940a692a.vps.ovh.net port 49281: Connection refused
```

**Causes possibles**:
1. SSH service not running
2. Firewall blocking port
3. Wrong IP/hostname
4. Network connectivity issue

**Diagnostic**:
```bash
# Test from local machine
ping vps-940a692a.vps.ovh.net          # Check connectivity
telnet vps-940a692a.vps.ovh.net 49281 # Check port open

# SSH to node (if ansible can't)
ssh -vvv -p 49281 supadmin@vps-940a692a.vps.ovh.net

# If firewall issue:
# Go to OVH panel: restart node or check firewall rules
```

**Solution**:
```bash
# SSH to node via OVH console, fix SSH:
sudo systemctl restart ssh
sudo systemctl status ssh

# Verify SSH config
sudo sshd -t  # Test config syntax

# If stuck, rebuild from OVH panel
```

---

### ❌ K3s Service Not Starting

**Symptôme**:
```bash
$ sudo systemctl status k3s
● k3s.service - Lightweight Kubernetes
   Loaded: loaded (/etc/systemd/system/k3s.service; enabled; vendor preset: enabled)
   Active: failed
```

**Causes**:
1. Swap enabled (K8s requirement)
2. Kernel module not loaded
3. Port already in use
4. Insufficient disk/memory

**Diagnostic**:
```bash
# Check swap
free -h | grep Swap         # Should be 0
sudo swapon --show         # Should be empty

# Check logs
sudo journalctl -u k3s -n 100 --no-pager

# Check ports
sudo netstat -pln | grep 6443
sudo netstat -pln | grep 10250

# Check disk
df -h / | tail -1
du -sh /var/lib/rancher/k3s

# Check memory
free -h
```

**Solution**:
```bash
# Disable swap
sudo swapoff -a
sudo sed -i '/ swap / s/^/#/' /etc/fstab

# Load kernel modules
sudo modprobe overlay
sudo modprobe br_netfilter

# Free up disk (if needed)
sudo docker system prune -a    # If Docker still exists

# Restart K3s
sudo systemctl restart k3s
sudo systemctl status k3s

# Wait for nodes to become Ready
watch sudo kubectl get nodes
```

---

### ❌ Pods Not Starting (ImagePullBackOff)

**Symptôme**:
```bash
$ sudo kubectl get pods -A
pods/n8n-xyz             0/1     ImagePullBackOff
```

**Causes**:
1. DNS not resolving
2. Image doesn't exist
3. Registry authentication failed
4. Network connectivity issue

**Diagnostic**:
```bash
# Describe pod
sudo kubectl describe pod n8n-xyz -n apps

# Test DNS resolution
sudo kubectl exec -it <pod> -n apps -- \
  apt update && apt install -y dnsutils
sudo kubectl exec -it <pod> -n apps -- \
  nslookup docker.io

# Check CoreDNS
sudo kubectl get pods -n kube-system | grep coredns
sudo kubectl logs -n kube-system -l k8s-app=kube-dns

# Test pulling image manually (on node)
sudo /var/lib/rancher/k3s/bin/crictl pull docker.io/library/nginx:latest
```

**Solution**:
```bash
# Fix DNS (often needed post K3s install)
sudo cat > /etc/resolv.conf <<EOF
nameserver 1.1.1.1
nameserver 8.8.8.8
nameserver 213.186.33.99
EOF

# OR restart K3s (refreshes config)
sudo systemctl restart k3s

# Delete pod to force reschedule
sudo kubectl delete pod n8n-xyz -n apps

# Verify
sudo kubectl get pods -n apps -w  # Wait for Running
```

---

### ❌ Nodes Not Joining Cluster

**Symptôme**:
```bash
$ sudo kubectl get nodes
NAME          STATUS      ROLES
ovh.core      Ready       control-plane,master
ovh.worker.01 NotReady    worker
ovh.worker.02 NotReady    worker
```

**Causes**:
1. K3s token mismatch
2. API server unreachable
3. Network issue between nodes
4. Different K3s versions

**Diagnostic**:
```bash
# On worker node:
sudo journalctl -u k3s -f   # Real-time logs

# Check if can reach API
sudo curl -k https://ovh.core:6443/api/v1  # Should respond

# Check DNS resolution
nslookup ovh.core

# Check K3s token
grep token /var/lib/rancher/k3s/agent/etc/config.yaml
# Compare with server: cat /var/lib/rancher/k3s/server/node-token

# Check firewall (on worker)
sudo nftables list ruleset | grep 6443
# Should allow TCP 6443 to server
```

**Solution**:
```bash
# Check token matches on server
ssh -p 49281 supadmin@ovh.core
sudo cat /var/lib/rancher/k3s/server/node-token

# SSH to worker, compare token
ssh -p 49281 supadmin@ovh.worker.01
sudo cat /var/lib/rancher/k3s/agent/etc/config.yaml

# If mismatch, restart agent:
sudo systemctl stop k3s-agent
sudo rm -rf /var/lib/rancher/k3s/agent/etc/config.yaml
# K3s agent will rejoin with correct token

# Or redeploy worker via Ansible:
ansible-playbook -i inventory.yml site.yml --tags k8s --limit ovh.worker.01

# Wait for worker to join
watch sudo kubectl get nodes
```

---

### ❌ Helm Install Failing

**Symptôme**:
```
Error: INSTALLATION FAILED: Unable to connect to the server: dial tcp: lookup kubernetes.default on ...: no such host
```

**Causes**:
1. kubeconfig path wrong
2. Kubernetes API unreachable
3. kubectl not configured
4. Helm module error

**Diagnostic**:
```bash
# Check kubeconfig exists
sudo ls -la /etc/rancher/k3s/k3s.yaml

# Test kubernetes access
sudo kubernetes.core.k8s:
  kubeconfig: /etc/rancher/k3s/k3s.yaml
  host: api_group_v1
  kind: Namespace
  name: test
  state: present

# Test helm directly
sudo helm list -n apps
sudo helm search repo bitnami

# Check Ansible module
python3 -c "from kubernetes import client; print('Module OK')"
```

**Solution**:
```bash
# Ensure kubeconfig accessible
sudo kubectl get nodes  # Test locally

# If using Ansible:
# Ensure in playbook:
kubeconfig: /etc/rancher/k3s/k3s.yaml

# Or check K3s running:
sudo systemctl status k3s

# Restart K3s if needed:
sudo systemctl restart k3s
sleep 10
sudo kubectl wait --for condition=ready node --all --timeout=120s

# Retry Helm deploy:
ansible-playbook -i inventory.yml site.yml --tags app_k8s
```

---

### ❌ Ingress Not Working (No HTTPS)

**Symptôme**:
```bash
$ curl https://n8n.dgsynthex.online
curl: (60) SSL certificate problem
```

**Causes**:
1. DNS not pointing to Traefik
2. Ingress not created
3. TLS certificate not issued
4. Traefik not routing

**Diagnostic**:
```bash
# Check Ingress created
sudo kubectl get ingress -A

# Check TLS certificate
sudo kubectl get certificate -A
sudo kubectl describe certificate -n apps <cert>

# Check Traefik logs
sudo kubectl logs -n kube-system -l app.kubernetes.io/name=traefik --tail=50

# Test DNS resolution
nslookup n8n.dgsynthex.online

# Check what IP DNS points to
dig n8n.dgsynthex.online +short

# Test connectivity to cluster (from outside)
curl -I http://n8n.dgsynthex.online   # Should redirect to HTTPS
curl -k https://n8n.dgsynthex.online  # With insecure cert
```

**Solution**:
```bash
# 1. Verify Ingress exists
sudo kubectl get ingress n8n -n apps -o yaml

# 2. Verify DNS points to Traefik IP
# Get Traefik IP:
sudo kubectl get svc -n kube-system traefik -o wide

# Update OVH DNS panel to point to this IP

# 3. Check certificate issuing
sudo kubectl logs -n kube-system -l app=traefik | grep -i acme

# 4. Force cert renewal
sudo kubectl delete certificate -n apps n8n-cert
# Traefik will recreate, may take 1-2 min

# 5. Test with curl
curl -I https://n8n.dgsynthex.online  # Should be 200 with valid cert
```

---

### ❌ Persistent Volume Not Mounting

**Symptôme**:
```bash
$ sudo kubectl get pvc
NAME              STATUS    VOLUME
postgres-pvc      Pending   (none)
```

**Causes**:
1. Local-path provisioner not running
2. Insufficient disk space
3. Node pressure (memory/disk)
4. PVC wrong StorageClass

**Diagnostic**:
```bash
# Check storage class exists
sudo kubectl get storageclass

# Check local-path provisioner
sudo kubectl get pods -n kube-system | grep local-path

# Check events on PVC
sudo kubectl describe pvc postgres-pvc -n apps

# Check node disk
sudo df -h /var/lib/rancher/k3s/storage

# Check node status
sudo kubectl describe node ovh.core | grep -A 5 Conditions
```

**Solution**:
```bash
# Ensure local-path provisioner running
sudo kubectl logs -n kube-system -l app.kubernetes.io/name=local-path-provisioner

# Check node has disk space
sudo df -h | grep -E "/$"
# Delete unnecessary files if full:
sudo rm -rf /tmp/*
sudo docker system prune -a  # If using Docker

# Check node conditions
sudo kubectl describe node ovh.core

# If node has DiskPressure:
sudo du -sh /* | sort -h
sudo journalctl --disk-usage
sudo journalctl --vacuum=100M  # Trim logs

# If needed, restart local-path provisioner
sudo kubectl delete pod -n kube-system -l app.kubernetes.io/name=local-path-provisioner
```

---

### ❌ Application Not Accessible (500 Error)

**Symptôm e**:
```bash
$ curl https://n8n.dgsynthex.online
HTTP/1.1 502 Bad Gateway
```

**Causes**:
1. App pod not running
2. App service not created
3. Traefik routing wrong
4. App crashed

**Diagnostic**:
```bash
# Check pod status
sudo kubectl get pods -n apps -l app=n8n

# Check logs
sudo kubectl logs -n apps -l app=n8n --tail=100

# Check service
sudo kubectl get svc -n apps | grep n8n

# Check endpoints
sudo kubectl get endpoints -n apps n8n

# Test connectivity from pod to backend
sudo kubectl exec -it <n8n-pod> -n apps -- \
  curl http://localhost:8080  # App's internal port

# Check Traefik routing
sudo kubectl logs -n kube-system -l app=traefik | grep n8n
```

**Solution**:
```bash
# Check if app is running
sudo kubectl logs -n apps -l app=n8n

# Common reasons:
# 1. App waiting for database - ensure postgres running
sudo kubectl get pods -n apps | grep postgres

# 2. App config wrong - check env vars
sudo kubectl describe pod -n apps <n8n-pod>

# 3. Memory/CPU issues - check resources
sudo kubectl top pods -n apps

# 4. Restart application
sudo kubectl rollout restart -n apps deployment/n8n

# 5. Check Ingress route
sudo kubectl describe ingress -n apps n8n
```

---

### ❌ Database Connection Error

**Symptom**:
```
No password has been provided for the "postgres" user (Kubernetes)
```

**Causes**:
1. Secret not created
2. Variable interpolation failed
3. Password character encoding issue
4. Ansible Vault not decrypted

**Diagnostic**:
```bash
# Check secret exists
sudo kubectl get secret -n apps postgres-credentials -o yaml

# Check secret value (base64 decoded)
sudo kubectl get secret -n apps postgres-credentials \
  -o jsonpath='{.data.password}' | base64 -d

# Check pod env vars
sudo kubectl exec -it <pod> -n apps -- env | grep POSTGRES

# Check app logs for actual error
sudo kubectl logs -n apps -l app=n8n | grep -i "password\|connection"
```

**Solution**:
```bash
# 1. Ensure secret.yml decrypted during playbook
ansible-playbook -i inventory.yml site.yml --tags app_k8s --ask-vault-pass

# 2. If secret missing, recreate:
sudo kubectl create secret generic postgres-credentials \
  --from-literal=password='YourStrongPassword' \
  -n apps

# 3. Restart pods to pick up secret
sudo kubectl rollout restart -n apps deployment/n8n

# 4. Check password special characters (some need escaping)
# Use Ansible to generate strong password safely:
# n8n_password: "{{ lookup('password', '/tmp/pass length=24 chars=ascii_letters,digits') }}"
```

---

### ❌ Firewall Blocking Connection

**Symptom**:
```bash
$ telnet ovh.worker.01 6443
Trying X.X.X.X...
Connection timed out
```

**Causes**:
1. NFTables rule missing
2. OVH cloud firewall blocking
3. ISP firewall blocking
4. Network route issue

**Diagnostic**:
```bash
# Check NFTables rules
sudo nftables list table inet firewall | grep 6443

# Check iptables (if still used)
sudo iptables -L -n | grep 6443

# Check route to node
traceroute ovh.worker.01

# Check from node perspective
sudo netstat -pln | grep 6443

# Check if port actually bound
sudo ss -tlnp | grep 6443
```

**Solution**:
```bash
# Add rule to NFTables if missing
sudo nft add rule inet firewall input tcp dport 6443 accept

# Persist
sudo systemctl save nftables

# Check OVH panel - may have anti-DDoS enabled
# Go to OVH panel: vps.ovh.com > Security > Anti-DDoS

# Test connectivity
nc -zv ovh.core 6443
# Should say "succeeded"
```

---

## 📊 Performance Troubleshooting

### High CPU Usage

```bash
# Find process using CPU
top -b -n 1 | sort -k 9 -r | head -10

# Find container using CPU
sudo kubectl top pods -A --sort-by=cpu

# If K3s/kubelet high:
sudo systemctl restart k3s
```

### High Memory Usage

```bash
# Check memory
free -h

# Check container mem
sudo kubectl top pods -A --sort-by=memory

# If OOMKilled:
# Increase node RAM or add new worker node
# OR reduce replica count for apps
```

### Disk Space Issues

```bash
# Find what's using disk
du -sh /* | sort -h

# Check container storage
df -h /var/lib/rancher/k3s

# Clean up K3s if full:
sudo /var/lib/rancher/k3s/bin/crictl image ls
sudo /var/lib/rancher/k3s/bin/crictl image rm <image-id>

# Clean logs
journalctl --vacuum=100M
```

---

## 🧪 Testing Commands

### Connectivity Tests

```bash
# Test SSH
ssh -p 49281 -T supadmin@ovh.core "echo OK"

# Test Kubernetes API
sudo kubectl cluster-info

# Test DNS
sudo kubectl exec -it <pod> -n default -- nslookup kubernetes.default

# Test inter-pod communication
sudo kubectl exec -it <pod1> -n apps -- \
  curl http://<pod2-service>:port

# Test ingress
curl -I https://n8n.dgsynthex.online
```

### Deployment Tests

```bash
# Dry-run Ansible
ansible-playbook -i inventory.yml site.yml --check

# Verbose output
ansible-playbook -i inventory.yml site.yml -vvv

# Test specific role
ansible-playbook -i inventory.yml site.yml --tags common --check

# Test command on all nodes
ansible -i inventory.yml k8s_cluster -m shell \
  -a "free -h && df -h /"
```

---

Voir aussi: [INSTALLATION.md](INSTALLATION.md), [SECURITY.md](SECURITY.md), [ROLES.md](ROLES.md)
