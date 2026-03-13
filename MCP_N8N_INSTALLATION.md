# MCP Integration pour N8N - Guide d'Installation

**Date**: Mars 2026  
**Référence**: https://github.com/czlonkowski/n8n-mcp  
**License**: MIT Open Source

---

## 📋 Qu'est-ce que MCP pour N8N?

**MCP** (Model Context Protocol) = Serveur qui connecte Claude/ChatGPT/Cursor directement à N8N
- ✅ Construire des workflows N8N via prompts IA
- ✅ Accès à 1000+ nœuds N8N
- ✅ Mises à jour différentielles (80-90% tokens économisés)
- ✅ 2700+ templates de workflows disponibles

**Exemple Use Case**:
```
Human: "Crée un workflow qui récupère mes commits GitHub et envoie un récap Slack chaque soir"
↓
Claude/ChatGPT interroge MCP server → récupère structure N8N
↓
IA construit le workflow complet automatiquement
↓
L'IA l'ajoute à N8N via API
```

---

## 🏗️ 3 Options d'Installation

### Option 1: Mode Hébergé (Simplest) ⭐ Recommandé pour POC

**Avantages**:
- ✅ Zéro configuration serveur
- ✅ OAuth setup automatique
- ✅ Support official
- ✅ Gratuit (100 appels/jour)
- ✅ Fonctionne avec Claude Desktop, ChatGPT, Cursor

**Inconvénients**:
- ❌ Données transitent par service tiers
- ❌ Limité à 100 appels/jour (gratuit)
- ❌ Dépendance sur service externe

**Installation**:
1. Aller à: https://mcp.n8n.io
2. Se connecter avec compte N8N
3. Autoriser OAuth
4. Obtenir une clé API
5. Ajouter à Claude/ChatGPT/Cursor settings

**Coût**: $0/mois (gratuit 100 appels)

---

### Option 2: Self-Hosted Docker (Rapide)

**Avantages**:
- ✅ Données 100% en-domaine
- ✅ Pas de limitation d'appels
- ✅ Facile à déployer
- ✅ Peut être hors-ligne

**Inconvénients**:
- ⚠️ Maintenance serveur
- ⚠️ Nécessite Node.js 18+ ou Docker
- ⚠️ Pas de UI, CLI based

**Installation via Docker Compose**:

```yaml
version: '3.9'
services:
  n8n-mcp:
    image: node:20-alpine
    container_name: n8n-mcp-server
    working_dir: /app
    volumes:
      - ./n8n-mcp:/app
    ports:
      - "8000:8000"  # Port du serveur MCP
    environment:
      - N8N_API_URL=http://n8n.dgsynthex.online
      - N8N_API_KEY=${N8N_API_KEY}  # Vault secret
      - MCP_SERVER_PORT=8000
    command: >
      sh -c "npm install && npm start"
    restart: unless-stopped
    networks:
      - microentreprise
```

**Installation manuelle**:
```bash
# Cloner le repo
git clone https://github.com/czlonkowski/n8n-mcp.git
cd n8n-mcp

# Installer dépendances
npm install

# Configurer
export N8N_API_URL="https://n8n.dgsynthex.online"
export N8N_API_KEY="sk_..."

# Lancer serveur
npm start
# Server écoute sur localhost:8000
```

**Coût**: $0/mois (serveur + bande passante VPS)

---

### Option 3: Self-Hosted dans Kubernetes ⭐ Recommandé pour Production

**Avantages**:
- ✅ Cohérent avec architecture K3s
- ✅ Automatable via Ansible
- ✅ Intégration réseau native (DNS K3s)
- ✅ Monitorable avec Prometheus
- ✅ HA + scaling possible
- ✅ Données 100% en-domaine

**Inconvénients**:
- ⚠️ Plus complexe à configurer
- ⚠️ Nécessite déploiement K3s
- ⚠️ Coût d'infrastructure (minimal sur K3s)

**Architecture**:
```
┌─────────────────────────────────────────────┐
│          K3s Kubernetes Cluster             │
│  ┌──────────────────────────────────────┐   │
│  │     app_k8s namespace               │   │
│  │  ┌──────────────┐  ┌──────────────┐ │   │
│  │  │     N8N      │  │  MCP Server  │ │   │
│  │  │   Pod        │  │   Pod        │ │   │
│  │  └──────────────┘  └──────────────┘ │   │
│  │         │           ↓              │   │
│  │         └→ HTTP :5678   (N8N API)  │   │
│  │         ← HTTP :8000   (MCP Server)│   │
│  └──────────────────────────────────────┘   │
│                                              │
│  ┌──────────────────────────────────────┐   │
│  │    admin namespace                   │   │
│  │    (Traefik Ingress)                │   │
│  │    mcp-server-ingress               │   │
│  │    → https://mcp.dgsynthex.online   │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
         ↓
    Claude/ChatGPT/Cursor
    (via HTTPS TLS)
```

**Implémentation Ansible Proposée**:

Créer: `roles/app_k8s/tasks/mcp-server.yml`

```yaml
---
#############################################################################
# N8N MCP Server Installation - Kubernetes
# Purpose: Deploy MCP (Model Context Protocol) server for N8N AI integration
# Reference: https://github.com/czlonkowski/n8n-mcp
#############################################################################

# 1. Create Namespace (if not existing)
- name: "MCP - Ensure namespace exists"
  kubernetes.core.k8s:
    state: present
    definition:
      apiVersion: v1
      kind: Namespace
      metadata:
        name: app_k8s

# 2. Create ConfigMap with startup script
- name: "MCP - Create startup script ConfigMap"
  kubernetes.core.k8s:
    state: present
    namespace: app_k8s
    definition:
      apiVersion: v1
      kind: ConfigMap
      metadata:
        name: mcp-startup-script
      data:
        init.sh: |
          #!/bin/sh
          set -e
          
          echo "Installing n8n-mcp..."
          cd /app
          
          # Install from npm
          npm install n8n-mcp
          
          # Create config
          cat > .env << EOF
          N8N_API_URL=http://n8n:5678
          N8N_API_KEY=$N8N_API_KEY
          MCP_SERVER_PORT=8000
          ENABLE_TEMPLATE_SEARCH=true
          TEMPLATE_SEARCH_LIMIT=50
          LOG_LEVEL=info
          EOF
          
          echo "Starting MCP Server..."
          npx n8n-mcp

# 3. Create Secret for N8N API Key
- name: "MCP - Create N8N API secret"
  kubernetes.core.k8s:
    state: present
    namespace: app_k8s
    definition:
      apiVersion: v1
      kind: Secret
      metadata:
        name: n8n-api-secret
      type: Opaque
      data:
        api-key: "{{ (n8n_admin_password | b64encode) }}"
        # Note: In production, use dedicated N8N API token

# 4. Create ServiceAccount for MCP
- name: "MCP - Create ServiceAccount"
  kubernetes.core.k8s:
    state: present
    namespace: app_k8s
    definition:
      apiVersion: v1
      kind: ServiceAccount
      metadata:
        name: mcp-server
        namespace: app_k8s

# 5. Deploy MCP Server Pod
- name: "MCP - Deploy MCP Server"
  kubernetes.core.k8s:
    kubeconfig: /etc/rancher/k3s/k3s.yaml
    state: present
    namespace: app_k8s
    definition:
      apiVersion: apps/v1
      kind: Deployment
      metadata:
        name: mcp-server
        namespace: app_k8s
      spec:
        replicas: 1
        selector:
          matchLabels:
            app: mcp-server
        template:
          metadata:
            labels:
              app: mcp-server
          spec:
            serviceAccountName: mcp-server
            containers:
              - name: mcp-server
                image: node:20-alpine
                imagePullPolicy: IfNotPresent
                command:
                  - /bin/sh
                  - /app/init.sh
                ports:
                  - containerPort: 8000
                    name: http
                env:
                  - name: N8N_API_URL
                    value: "http://n8n:5678"
                  - name: N8N_API_KEY
                    valueFrom:
                      secretKeyRef:
                        name: n8n-api-secret
                        key: api-key
                  - name: MCP_SERVER_PORT
                    value: "8000"
                  - name: NODE_ENV
                    value: "production"
                  - name: LOG_LEVEL
                    value: "info"
                volumeMounts:
                  - name: startup-script
                    mountPath: /app
                    readOnly: false
                  - name: tmp
                    mountPath: /tmp
                resources:
                  requests:
                    cpu: "100m"
                    memory: "256Mi"
                  limits:
                    cpu: "500m"
                    memory: "512Mi"
                livenessProbe:
                  httpGet:
                    path: /health
                    port: 8000
                  initialDelaySeconds: 30
                  periodSeconds: 10
                readinessProbe:
                  httpGet:
                    path: /health
                    port: 8000
                  initialDelaySeconds: 10
                  periodSeconds: 5
                securityContext:
                  allowPrivilegeEscalation: false
                  runAsNonRoot: true
                  runAsUser: 1000
                  capabilities:
                    drop:
                      - ALL
            volumes:
              - name: startup-script
                configMap:
                  name: mcp-startup-script
                  defaultMode: 0755
              - name: tmp
                emptyDir: {}

# 6. Create Service for MCP Server
- name: "MCP - Create Service"
  kubernetes.core.k8s:
    kubeconfig: /etc/rancher/k3s/k3s.yaml
    state: present
    namespace: app_k8s
    definition:
      apiVersion: v1
      kind: Service
      metadata:
        name: mcp-server
        namespace: app_k8s
      spec:
        selector:
          app: mcp-server
        ports:
          - protocol: TCP
            port: 8000
            targetPort: 8000
        type: ClusterIP

# 7. Create Traefik Middleware for rate limiting MCP
- name: "MCP - Create Traefik rate limit middleware"
  kubernetes.core.k8s:
    kubeconfig: /etc/rancher/k3s/k3s.yaml
    state: present
    namespace: app_k8s
    definition:
      apiVersion: traefik.io/v1alpha1
      kind: Middleware
      metadata:
        name: mcp-rate-limit
        namespace: app_k8s
      spec:
        rateLimit:
          average: 1000                      # 1000 req/sec (AI usage)
          burst: 200                         # Allow spikes
          period: 1s

# 8. Create Ingress for MCP Server
- name: "MCP - Create Ingress with TLS"
  kubernetes.core.k8s:
    kubeconfig: /etc/rancher/k3s/k3s.yaml
    state: present
    namespace: app_k8s
    definition:
      apiVersion: networking.k8s.io/v1
      kind: Ingress
      metadata:
        name: mcp-server
        namespace: app_k8s
        annotations:
          traefik.ingress.kubernetes.io/router.entrypoints: websecure
          traefik.ingress.kubernetes.io/router.tls: "true"
          traefik.ingress.kubernetes.io/router.tls.certresolver: "le"
          traefik.ingress.kubernetes.io/router.middlewares: "app_k8s-admin-auth@kubernetescrd,app_k8s-mcp-rate-limit@kubernetescrd"
      spec:
        ingressClassName: traefik
        rules:
          - host: "mcp.{{ main_domain }}"
            http:
              paths:
                - path: /
                  pathType: Prefix
                  backend:
                    service:
                      name: mcp-server
                      port:
                        number: 8000

# 9. Create NetworkPolicy for MCP
- name: "MCP - Create NetworkPolicy for MCP Server"
  kubernetes.core.k8s:
    kubeconfig: /etc/rancher/k3s/k3s.yaml
    state: present
    namespace: app_k8s
    definition:
      apiVersion: networking.k8s.io/v1
      kind: NetworkPolicy
      metadata:
        name: mcp-allow-ingress
        namespace: app_k8s
      spec:
        podSelector:
          matchLabels:
            app: mcp-server
        policyTypes:
          - Ingress
          - Egress
        ingress:
          - from:
              - namespaceSelector:
                  matchLabels:
                    name: kube-system
            ports:
              - protocol: TCP
                port: 8000
        egress:
          # Outbound to Kubernetes DNS
          - to:
              - namespaceSelector:
                  matchLabels:
                    name: kube-system
            ports:
              - protocol: UDP
                port: 53
          # Outbound to N8N service
          - to:
              - podSelector:
                  matchLabels:
                    app: n8n
            ports:
              - protocol: TCP
                port: 5678
          # Outbound HTTPS (external APIs, webhooks)
          - to:
              - podSelector: {}
            ports:
              - protocol: TCP
                port: 443

# 10. Log deployment status
- name: "MCP - Display deployment info"
  debug:
    msg: |
      ✅ MCP Server deployed successfully!
      
      Access Point: https://mcp.{{ main_domain }}
      Internal Service: http://mcp-server:8000 (from within cluster)
      
      Next Steps:
      1. Wait for pod to be ready (check logs below)
      2. Configure Claude/ChatGPT with API endpoint
      3. Set environment variable: MCP_URL=https://mcp.{{ main_domain }}
      
      To view logs:
        kubectl logs -n app_k8s -l app=mcp-server -f
      
      To test endpoint:
        curl -k https://mcp.{{ main_domain }}/health
```

---

## 🚀 Déploiement recommandé (Option 3 - K8s)

### Étape 1: Ajouter tâche MCP à l'orchestration

Ajouter dans `roles/app_k8s/tasks/main.yml`:

```yaml
# ... existing tasks ...

- include_tasks: n8n.yml
- include_tasks: n8n-security.yml
- include_tasks: mcp-server.yml          # 👈 Nouveau!
- include_tasks: nodered.yml
- include_tasks: nodered-security.yml
```

### Étape 2: Configurer N8N pour utiliser MCP

Ajouter dans `roles/app_k8s/tasks/n8n.yml` (section environment):

```yaml
env:
  # ... existing env vars ...
  - name: N8N_MCP_SERVER_URL
    value: "http://mcp-server:8000"         # Intra-cluster access
  - name: N8N_MCP_ENABLED
    value: "true"
  - name: N8N_MCP_TEMPLATE_SEARCH_ENABLED
    value: "true"
```

### Étape 3: Déployer

```bash
# Option A: Déployer seulement MCP pour tester
ansible-playbook -i inventory.yml site.yml \
  --tags app_k8s \
  -e "deploy_mcp_only=true" \
  --ask-vault-pass

# Option B: Déploiement complet (N8N + MCP + Node-RED + sécurité)
ansible-playbook -i inventory.yml site.yml \
  --tags app_k8s \
  --ask-vault-pass
```

### Étape 4: Vérifier le déploiement

```bash
# Voir le pod
kubectl get pods -n app_k8s -l app=mcp-server

# Voir les logs
kubectl logs -n app_k8s -l app=mcp-server -f

# Tester l'endpoint
curl -k https://mcp.dgsynthex.online/health

# Vérifier latence N8N → MCP
kubectl exec -it <n8n-pod> -n app_k8s -- \
  curl -s http://mcp-server:8000/health | jq .
```

---

## 🔌 Configuration des Clients AI

### Claude Desktop

Ajouter dans `~/.claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "n8n": {
      "command": "curl",
      "args": [
        "https://mcp.dgsynthex.online"
      ],
      "env": {
        "MCP_API_KEY": "sk_..."
      }
    }
  }
}
```

### ChatGPT / OpenAI

1. Aller à: ChatGPT Settings → Plugins → Custom
2. URL: `https://mcp.dgsynthex.online`
3. API Key: (depuis N8N)
4. Authorization: Bearer token

### Cursor (VS Code Fork)

1. Settings → AI → MCP Servers
2. Server: `n8n`
3. URL: `https://mcp.dgsynthex.online`
4. Type: `REST API`

---

## 🛡️ Sécurité MCP

### ✅ Best Practices

| Aspect | Recommandation |
|--------|----------------|
| **Auth** | Activer API Key + TLS certificate |
| **RBAC** | Créer ServiceAccount dédié MCP |
| **Network** | NetworkPolicy restrict egress vers N8N seulement |
| **Rate Limit** | 1000 req/s (IA peut être "bruteforcy") |
| **Backups** | Exporter workflows N8N avant modifications IA |
| **Audit** | Logger tous les appels MCP dans N8N |
| **Secrets** | Jamais laisser l'IA accéder aux Vault secrets |

### ⚠️ Avertissements

**NE PAS laisser l'IA modifier production directement!**

```bash
# ✅ BON: L'IA crée draft dans dev
AI → MCP → N8N /draft/workflow
Human → Test → Promote to prod

# ❌ MAUVAIS: L'IA modifie prod directement
AI → MCP → N8N /workflow/prod
# Risk: IA supprime nœuds "inutiles", casse facturation, etc.
```

Processus sécurisé:

```
1. IA crée workflow en DEV
   MCP_ENVIRONMENT=dev
   
2. Human teste et valide
   curl https://n8n.dgsynthex.online/workflow/dev/test
   
3. Human approuve et exporte backup
   kubectl exec n8n -- n8n export
   
4. Human promeut à PROD (manual ou approval-based)
   kubectl apply -f workflows-prod.yaml
```

---

## 📊 Monitoring MCP Server

### Prometheus Metrics

```yaml
# Requests per second
rate(http_requests_total{service="mcp-server"}[1m])

# Error rate
rate(http_requests_total{service="mcp-server",status=~"5.."}[1m])

# Latency to N8N
http_request_duration_seconds{path="/api/n8n",quantile="0.95"}

# Memory usage
container_memory_usage_bytes{pod_name="mcp-server"}
```

### Grafana Dashboard

```alert
MCP Server Health:
- Pod running? ✅
- Responding to /health? ✅
- Connected to N8N? ✅
- Request rate < 1000 req/s? ✅
```

---

## 🔄 Mise à jour & Maintenance

### Mise à jour MCP Server

```bash
# Voir version actuelle
kubectl exec -it <mcp-pod> -n app_k8s -- npm list n8n-mcp

# Mettre à jour sans downtime (rolling update)
kubectl set image deployment/mcp-server \
  mcp-server=node:20-alpine \
  -n app_k8s

# Vérifier
kubectl rollout status deployment/mcp-server -n app_k8s
```

### Rollback en cas de problème

```bash
# Voir history
kubectl rollout history deployment/mcp-server -n app_k8s

# Rollback à version antérieure
kubectl rollout undo deployment/mcp-server -n app_k8s
```

---

## ❓ FAQ

**Q: Quel est le coût MCP server?**  
R: $0 - C'est MIT open source. Seul coût: resources K3s (256Mi RAM, 100m CPU = négligeable)

**Q: Peut-on faire du multi-instance MCP?**  
R: Oui! Faire `replicas: 2-3` dans Deployment et ajouter un Load Balancer

**Q: La limite de 100 appels/jour s'applique-t-elle en self-hosted?**  
R: Non! Self-hosted = illimité. Limites définies par votre rate limit (1000 req/s ici)

**Q: Comment l'IA accède à mes workflows?**  
R: Via N8N API (REST).  MCP expose les workflows + nœuds + templates à l'IA de manière sécurisée

**Q: Peut-on combiner MCP avec Chrome DevTools MCP?**  
R: Oui! L'IA peut construire workflow N8N ET debug Chrome en même temps

**Q: Timeout trop long lors de la première requête MCP?**  
R: Normal! npm install prend 30-60s. Augmenter `initialDelaySeconds: 60` dans probe

---

## 📞 Support & Ressources

- **Repository**: https://github.com/czlonkowski/n8n-mcp
- **N8N API Docs**: https://docs.n8n.io/api/
- **MCP Spec**: https://modelcontextprotocol.io/
- **Issues/Questions**: GitHub Issues sur n8n-mcp repo

---

Voir aussi: [APPLICATION_DEPLOYMENT.md](APPLICATION_DEPLOYMENT.md), [N8N_NODERED_SECURITY.md](N8N_NODERED_SECURITY.md), [ARCHITECTURE.md](ARCHITECTURE.md)
