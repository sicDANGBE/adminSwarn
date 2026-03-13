# N8N & Node-RED - Améliorations Sécurité Avancées

**Date**: Mars 2026  
**Version**: 2.0 (Enhanced Security)  
**Scope**: Kubernetes deployment security hardening

---

## 📋 Récapitulatif Améliorations

### N8N Security Enhancements

| Feature | Before | After | Benefit |
|---------|--------|-------|---------|
| Rate Limiting | None | Traefik Middleware (100 req/s) | DDoS protection |
| RBAC | No ServiceAccount | Full RBAC + Role/RoleBinding | Pod isolation |
| Network Egress | Allow all | Restricted to necessary services only | Malware containment |
| Pod Disruption Budget | None | minAvailable: 1 | HA & reliability |
| ResourceQuota | None | CPU/Memory/Pods limits | Resource fairness |
| Webhook Control | Unlimited | Timeout + rate limits | Abuse prevention |
| ServiceAccount | None | Dedicated + minimal permissions | Least privilege |
| Audit Logging | Disabled | Enabled with audit annotations | Compliance |
| CORS | Wide open | Restricted to domain only | XSS prevention |
| Ingress Rate Limit | None | Per-route rate limit | API security |

### Node-RED Security Enhancements

| Feature | Before | After | Benefit |
|---------|--------|-------|---------|
| Rate Limiting | None | Traefik Middleware (50 req/s) | DDoS protection |
| RBAC | No ServiceAccount | Full RBAC with ConfigMap access | Pod isolation |
| Network Egress | Allow all | Restricted to MQTT/PostgreSQL/RabbitMQ | Lateral movement prevention |
| External Modules | Disabled (good) | Still disabled, documented | Supply chain security |
| Function Timeout | None | 30 seconds hardcoded | Resource DoS prevention |
| Pod Disruption Budget | None | minAvailable: 1 | High availability |
| Dangerous Nodes | Not restricted | Disabled (file, exec) | Arbitrary code prevention |
| Diagnostic Mode | Not controlled | Explicitly disabled | Data leak prevention |
| Audit Config | None | Full audit trail enabled | Compliance logging |
| HTTP Request Limits | Unbounded | 10MB max body size | Memory exhaustion prevention |

---

## 🔐 Détail des Améliorations

### 1. Rate Limiting (Traefik Middleware)

**N8N**:
```yaml
rateLimit:
  average: 100              # 100 requests per second
  burst: 50                 # Allow spikes of 50
  period: 1s
```

**Node-RED**:
```yaml
rateLimit:
  average: 50               # Lower threshold for Node-RED
  burst: 25
  period: 1s
```

**Protections**:
- Prévient brute-force attacks
- DDoS mitigation
- Abuse protection (webhooks)
- Fair resource sharing

**Testing**:
```bash
# Générer traffic pour tester rate limit
for i in {1..200}; do 
  curl -u admin:password https://n8n.dgsynthex.online & 
done
# Après 100+ requests, vous recevrez 429 (Too Many Requests)
```

---

### 2. Service Accounts & RBAC

**Création ServiceAccount distincte**:
```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: n8n
  namespace: admin
```

**Role avec permissions minimales**:
```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: n8n
rules:
  - apiGroups: [""]
    resources: ["configmaps"]
    verbs: ["get", "list", "watch"]
  # Pas d'accès Secrets, Pods, etc.
```

**Avantages**:
- Least privilege principle
- Container compromise limitation
- API access control
- Audit trail per serviceaccount

**Vérification**:
```bash
# Voir token associé
kubectl get serviceaccount n8n -n admin -o yaml

# Voir role binding
kubectl get rolebinding n8n -n admin -o yaml

# Tester accès API depuis pod
kubectl exec -it <n8n-pod> -n admin -- \
  curl -H "Authorization: Bearer $KUBERNETES_TOKEN" \
  https://kubernetes.default.svc.cluster.local/api/v1/configmaps
```

---

### 3. Network Policies - Egress Control

**N8N Egress Policy**:
```yaml
egress:
  # ✅ DNS (nécessaire)
  - to:
      - namespaceSelectorSelector: kube-system
    ports:
      - protocol: UDP
        port: 53
  
  # ✅ Databases & Message Brokers
  - to:
      - namespaceSelector: apps
    ports:
      - protocol: TCP
        port: 5432   # PostgreSQL
      - protocol: TCP
        port: 5672   # RabbitMQ
  
  # ✅ External HTTPS (webhooks, integrations)
  - to:
      - podSelector: {}
    ports:
      - protocol: TCP
        port: 443
  
  # ❌ Blocage: SSH, File transfers, P2P
```

**Node-RED Egress Policy**:
```yaml
egress:
  # ✅ DNS
  # ✅ MQTT (1883, 8883)
  # ✅ Databases (PostgreSQL)
  # ✅ Message Brokers (RabbitMQ)
  # ✅ External HTTP/HTTPS
  # ❌ Blocage: File operations, Exec
```

**Protections**:
- Lateral movement prevention
- Malware C&C communication blocking
- Data exfiltration prevention
- Credential theft containment

**Testing**:
```bash
# Vérifier politique est appliquée
kubectl get networkpolicies -n admin

# Tester connexion bloquée
kubectl exec -it <n8n-pod> -n admin -- \
  ping 8.8.8.8  # Should timeout (no ICMP allowed)

# Tester connexion permise
kubectl exec -it <n8n-pod> -n admin -- \
  curl https://api.github.com  # Should work
```

---

### 4. Pod Disruption Budgets (High Availability)

```yaml
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: n8n
spec:
  minAvailable: 1
  selector:
    matchLabels:
      app: n8n
```

**Protections**:
- Pods non evicted pendant maintenance
- Uptime garantie
- Graceful degradation
- Cluster updates sans downtime

**Scenarios**:
- Node drain (maintenance) → Pod reschedule garantie
- Dnsmasq restart → Pod survit
- Voluntary disruption → Respected

---

### 5. Resource Quotas

```yaml
apiVersion: v1
kind: ResourceQuota
metadata:
  name: admin-quota
  namespace: admin
spec:
  hard:
    requests.cpu: "2000m"
    requests.memory: "2Gi"
    limits.cpu: "4000m"
    limits.memory: "4Gi"
    pods: "10"
    services: "10"
```

**Protections**:
- Resource fairness (admin namespace can't hog all)
- Runaway pod detection
- Memory leak protection
- CPU overcommit prevention

**Monitoring**:
```bash
# Voir utilisation actuelle
kubectl describe quota admin-quota -n admin

# Template:
# Requests: 250m CPU, 512Mi Memory (N8N pod)
# Hard limit: 2000m CPU, 2Gi Memory
# Available: ~1750m CPU, ~1.5Gi Memory pour autres pods
```

---

### 6. Webhook & External API Rate Limiting

**N8N Configuration**:
```env
N8N_WEBHOOK_TIMEOUT=30              # seconds
N8N_MAX_REQUEST_BODY_SIZE=10mb      # prevent memory exhaustion
N8N_CORS_ENABLED=true
N8N_CORS_ORIGIN=https://n8n.dgsynthex.online
N8N_WEBHOOK_RATE_LIMIT_ENABLED=true
```

**Protections**:
- Slow client DoS prevention (timeout)
- Large payload attacks blocked
- CORS prevents cross-domain abuse
- Webhook flooding protection

**Configuration Example**:
```yaml
# In N8N webhook:
# Max 30 seconds execution time
# Max 10MB payload size
# Only accept requests from configured domain
```

---

### 7. Kubernetes Security Context

**Pod Level**:
```yaml
securityContext:
  fsGroup: 1000                    # Group ID for volumes
  runAsGroup: 1000
  runAsNonRoot: true               # NO root execution
  runAsUser: 1000                  # UID 1000 (node user)
  seccompProfile:
    type: RuntimeDefault           # Kernel security module
```

**Container Level**:
```yaml
securityContext:
  allowPrivilegeEscalation: false  # NO setuid/setgid
  capabilities:
    drop:
      - ALL                        # Drop all Linux capabilities
  readOnlyRootFilesystem: false    # N8N writes config
  runAsNonRoot: true
  runAsUser: 1000
```

**Protections**:
- Container escape prevention
- Privilege escalation blocking
- Filesystem write control
- Capabilities limiting

**Verification**:
```bash
# Check running as UID 1000
kubectl exec -it <n8n-pod> -n admin -- id
# Output: uid=1000(node) gid=1000 groups=1000,1000

# Verify capabilities dropped
kubectl exec -it <n8n-pod> -n admin -- \
  grep CapEff /proc/self/status
# Output: CapEff: 0000000000000000 (none)
```

---

### 8. Audit Logging

**N8N Audit Configuration**:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: n8n-audit-config
data:
  enabled-modules: |
    - core
    - api
    - workflow
    - webhook
    - auth
    - external-api
```

**Node-RED Audit Configuration**:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: nodered-audit-config
data:
  enabled-audits: |
    - flow-deploy
    - node-create
    - node-delete
    - wire-change
    - auth-attempt
    - api-call
```

**What's Logged**:
- Authentication attempts (success/failure)
- Workflow/Flow changes
- API calls to external services
- Webhook executions
- Node modifications

**Access Logs**:
```bash
# View N8N logs
kubectl logs -n admin -l app=n8n -f | grep -i audit

# Export for SIEM
kubectl logs -n admin -l app=n8n --tail=1000 > n8n-audit.log
```

---

### 9. CORS Restrictions

**N8N CORS Config**:
```env
N8N_CORS_ORIGIN=https://n8n.dgsynthex.online
```

**Browser Protection**:
```
Origin: https://evil.attacker.com
↓
Traefik: Only https://n8n.dgsynthex.online allowed
↓
Request blocked (403 CORS)
```

**Prevents**:
- Cross-site request forgery (CSRF)
- XSS attacks from other domains
- Credential theft via web pages

---

### 10. Health Checks (Liveness, Readiness, Startup)

**N8N Probes**:
```yaml
livenessProbe:
  httpGet:
    path: /healthz
    port: 5678
  initialDelaySeconds: 60
  periodSeconds: 30           # Check every 30s
  failureThreshold: 3         # Kill after 3 failures
```

**Benefits**:
- Automatic restart of crashed pods
- Load balancer knows unhealthy pods
- Zero-downtime deployments
- Quick failure detection

**Testing**:
```bash
# Manually check health endpoint
kubectl exec -it <n8n-pod> -n admin -- \
  curl http://localhost:5678/healthz

# Watch restarts
kubectl get pods -n admin -w
# If pod restarts frequently, check logs:
kubectl logs -n admin <pod> --previous
```

---

## 🚀 Déploiement des Améliorations

### Option 1: Appliquer sur cluster existant

Si N8N/Node-RED sont déjà déployés:

```bash
# Les nouvelles tâches s'ajoutent automatiquement
ansible-playbook -i inventory.yml site.yml \
  --tags app_k8s \
  --ask-vault-pass

# Kubernetes remplacera les vieilles ressources avec les nouvelles
# ⚠️ Déploiement zero-downtime (1-2 minutes d'interruption possible)
```

### Option 2: Deployment frais

```bash
# Nettoyer namespace existant
kubectl delete namespace admin

# Redéployer complet
ansible-playbook -i inventory.yml site.yml \
  --tags app_k8s \
  --ask-vault-pass
```

### Rollback (si problème)

```bash
# Restaurer version antérieure du pod
kubectl rollout undo deployment/n8n -n admin
kubectl rollout undo deployment/nodered -n admin

# Ou réappliquer from git
git checkout <previous-commit>
ansible-playbook -i inventory.yml site.yml --tags app_k8s
```

---

## 📊 Monitoring des Améliorations de Sécurité

### Prometheus Metrics

```bash
# Container CPU/Memory usage
container_cpu_usage_seconds_total{pod="n8n"}
container_memory_usage_bytes{pod="nodered"}

# Network policy violations (if logging enabled)
network_policy_dropped_packets{pod="n8n"}

# Rate limit hits
traefik_entrypoint_requests_total{code="429"}

# Pod disruption budget status
kube_pod_disruption_budget_disruptions_allowed
```

### Log Monitoring

```bash
# N8N Audit logs
kubectl logs -n admin -l app=n8n | grep -i "auth\|webhook\|api"

# Node-RED Flow changes
kubectl logs -n admin -l app=nodered | grep -i "flow\|deploy"

# Network policy denials
kubectl logs -n admin error | grep -i "networkpolicy"
```

### Grafana Dashboard Recommended

```json
{
  "dashboard": "N8N & Node-RED Security Monitoring",
  "panels": [
    "Request Rate (429 errors)",
    "Pod CPU/Memory usage vs Limits",
    "Network Policy Violations",
    "Auth Failures per minute",
    "Webhook Timeouts",
    "Resource Quota Exhaustion"
  ]
}
```

---

## 🛡️ Best Practices

### DO's ✅

- ✅ Rotate secrets regularly (API keys, passwords)
- ✅ Monitor audit logs for anomalies
- ✅ Test network policies before production
- ✅ Keep Kubernetes & container images updated
- ✅ Review ServiceAccount permissions quarterly
- ✅ Enable Prometheus scraping for metrics
- ✅ Use secrets for ALL sensitive data
- ✅ Restrict Ingress to HTTPS only

### DON'Ts ❌

- ❌ Don't run containers as root
- ❌ Don't expose admin UI to public internet
- ❌ Don't store secrets in ConfigMaps
- ❌ Don't disable security contexts
- ❌ Don't allow unlimited resource usage
- ❌ Don't skip network policies
- ❌ Don't hardcode credentials in images
- ❌ Don't disable audit logging

---

## 📞 Troubleshooting

### Pod won't start

```bash
# Check events
kubectl describe pod <pod> -n admin

# Check logs
kubectl logs <pod> -n admin

# Common issues:
# - ResourceQuota exceeded
# - PVC not ready
# - Image pull failure
```

### Rate limiting being too strict

```bash
# Adjust numbers in Middleware
kubectl edit middleware n8n-rate-limit -n admin
# Increase average, burst values

# Or temporarily disable
kubectl patch ingress n8n -n admin \
  -p '{"metadata":{"annotations":{"traefik...":"false"}}}'
```

### Network policy blocking legitimate traffic

```bash
# Check current policies
kubectl get networkpolicies -n admin

# Test denied traffic
kubectl exec -it <pod> -n admin -- \
  curl -v http://<destination>:<port>

# Adjust egress rules if needed
kubectl edit networkpolicy n8n-egress-control -n admin
```

---

Voir aussi: [SECURITY.md](../SECURITY.md), [APPLICATION_DEPLOYMENT.md](../APPLICATION_DEPLOYMENT.md), [INSTALLATION.md](../INSTALLATION.md)
