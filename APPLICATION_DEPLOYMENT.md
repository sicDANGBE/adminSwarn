# Déploiement des Applications - Guide Détaillé

## 📦 Vue d'Ensemble des Applications

9 services déployés sur le cluster K3s via Helm charts:

```
┌─────────────────────────────────────────────────────────────┐
│                                                               │
│        Kubernetes (K3s) - Cluster dgsynthex.online           │
│                                                               │
├─────────────── NAMESPACE: kube-system ──────────────────────┤
│                                                               │
│  Traefik        → Ingress Controller, Load Balancer         │
│  CoreDNS        → Cluster DNS                               │
│  Metrics Server → Pod resource metrics                      │
│  Local-path     → Storage provisioner                       │
│                                                               │
├─────────────── NAMESPACE: portainer ──────────────────────┤
│                                                               │
│  Portainer CE   → Kubernetes cluster management UI          │
│                                                               │
├─────────────── NAMESPACE: apps ──────────────────────────┤
│                                                               │
│  PostgreSQL     → Relational database backend               │
│  RabbitMQ       → Message broker (AMQP)                     │
│  N8N            → Workflow automation platform              │
│  Node-RED       → Visual function orchestration             │
│  MQTT Broker    → IoT message broker                        │
│                                                               │
├─────────────── NAMESPACE: monitoring ──────────────────────┤
│                                                               │
│  Prometheus     → Metrics collector                         │
│  Grafana        → Metrics visualization                     │
│  AlertManager   → Alert routing                             │
│                                                               │
├─────────────── NAMESPACE: ollama-system ──────────────────┤
│                                                               │
│  Ollama         → Local LLM inference                       │
│                                                               │
└─────────────────────────────────────────────────────────────┘
```

---

## 🚀 Application 1: Traefik Ingress Controller

**Status**: Built-in K3s (déployé par `k8s` role)  
**Namespace**: `kube-system`  
**Access**: https://traefik.dgsynthex.online

### Rôle
- Reverse proxy & load balancer
- TLS/HTTPS termination (Let's Encrypt)
- Ingress rule processing
- Automatic certificate management

### Configuration

**Fichier Template**: `roles/k8s/templates/traefik-config.yaml.j2`

```yaml
providers:
  kubernetesIngress:
    ingressClass: traefik
    allowEmptyServices: false

entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
  websecure:
    address: ":443"

certificatesResolvers:
  letsencrypt:
    acme:
      email: admin@dgsynthex.online
      storage: /data/acme.json
      httpChallenge:
        entryPoint: web
      caServer: https://acme-v02.api.letsencrypt.org/directory

api:
  dashboard: true
  debug: false

metrics:
  prometheus: {}
```

### Vérification

```bash
# Vérifier Traefik running
sudo kubectl get pods -n kube-system | grep traefik

# Voir Ingress routes
sudo kubectl get ingress -A

# Dashboard
open https://traefik.dgsynthex.online

# API
sudo kubectl port-forward -n kube-system svc/traefik 9000:9000 &
open http://localhost:9000/api/rawdata
```

### Troubleshooting

```bash
# Logs
sudo kubectl logs -n kube-system -l app.kubernetes.io/name=traefik

# Check certificate
sudo kubectl get certificate -A

# Force cert refresh
sudo kubectl delete certificate -n apps <app>-cert
```

---

## 🗄️ Application 2: PostgreSQL Database

**Chart**: Bitnami/postgresql  
**Namespace**: `apps`  
**Internal Access**: `postgresql.apps.svc.cluster.local:5432`

### Rôle
Backend database pour N8N, Node-RED, et autres applications.

### Configuration (vars/main.yml)

```yaml
helm_versions:
  postgres: "15.0.0"  # PostgreSQL 15
```

### Valeurs Helm

```yaml
# Depuis secret.yml (Vault)
auth:
  username: "{{ postgres_user }}"            # "postgres"
  password: "{{ postgres_password }}"        # "STRONG_PASSWORD"
  database: "applications"

primary:
  persistence:
    enabled: true
    storageClass: local-path
    size: 20Gi
    mountPath: /var/lib/postgresql

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi
```

### Accès Intra-Cluster

```bash
# Depuis autre app (ex: N8N):
DATABASE_URL=postgresql://{{ postgres_user }}:{{ postgres_password }}@postgresql.apps.svc.cluster.local:5432/applications

# Tester connexion
sudo kubectl exec -it <n8n-pod> -n apps -- \
  psql -h postgresql.apps.svc.cluster.local \
       -U postgres \
       -d applications
```

### Backup & Restore

```bash
# Backup database
sudo kubectl exec -it <postgres-pod> -n apps -- \
  pg_dump -U postgres applications > backup.sql

# Restore
sudo kubectl exec -i <postgres-pod> -n apps -- \
  psql -U postgres applications < backup.sql

# Backup PVC
sudo kubectl get pvc -n apps postgres-pvc -o yaml > postgres-pvc-backup.yaml
```

### Vérification

```bash
# Pods running
sudo kubectl get pods -n apps | grep postgres

# Database connection
sudo kubectl exec -it <postgres-pod> -n apps -- \
  psql -U postgres -l

# Size
sudo kubectl exec -it <postgres-pod> -n apps -- \
  du -sh /var/lib/postgresql/data
```

---

## 🐰 Application 3: RabbitMQ Message Broker

**Chart**: Bitnami/rabbitmq  
**Version**: 12.0.0  
**Namespace**: `apps`  
**Access**: https://rabbitmq.dgsynthex.online (Management UI)  
**AMQP Port**: 5672

### Rôle
Message broker pour communication asynchrone entre microservices.

### Configuration

```yaml
auth:
  username: guest
  password: "{{ rabbitmq_password }}"
  erlangCookie: "{{ rabbitmq_erlang_cookie }}"

replicaCount: 1  # Increase for HA

clustering:
  enabled: false  # Enable for production HA

resources:
  requests:
    cpu: 250m
    memory: 256Mi
  limits:
    cpu: 500m
    memory: 512Mi

persistence:
  enabled: true
  storageClass: local-path
  size: 10Gi
```

### Management UI

```bash
# Access
https://rabbitmq.dgsynthex.online

# Default credentials in secret.yml
Username: guest
Password: {{ rabbitmq_password }}

# Create exchange
# Create queue
# Bind exchange to queue
# View messages
```

### Python Client

```python
import pika

credentials = pika.PlainCredentials('guest', 'PASSWORD')
connection = pika.BlockingConnection(
    pika.ConnectionParameters(
        host='rabbitmq.apps.svc.cluster.local',
        credentials=credentials
    )
)
channel = connection.channel()

# Publish
channel.basic_publish(
    exchange='',
    routing_key='queue_name',
    body='Message'
)

# Consume
channel.basic_consume(queue='queue_name', on_message_callback=callback)
channel.start_consuming()
```

### Queues de Base (à créer)

```bash
# Connect within pod
sudo kubectl exec -it <rabbitmq-pod> -n apps -- bash

# Create queue via CLI
rabbitmqctl add_queue my_queue
rabbitmqctl add_exchange my_exchange direct
rabbitmqctl bind_queue my_queue my_exchange queue_key
```

---

## 🤖 Application 4: N8N - Workflow Automation

**Chart**: n8n community / OCI  
**Namespace**: `apps`  
**Access**: https://n8n.dgsynthex.online  
**Backend DB**: PostgreSQL (apps namespace)

### Rôle
Visual workflow automation platform. Intègre 400+ services externes.

### Configuration (vars/main.yml)

```yaml
helm_versions:
  n8n: "0.14.0"
```

**Helm Values**:
```yaml
replicaCount: 1

image:
  repository: n8nio/n8n
  tag: latest.alpine
  pullPolicy: IfNotPresent

env:
  - name: DB_TYPE
    value: "postgresdb"
  - name: DB_POSTGRESDB_HOST
    value: "postgresql.apps.svc.cluster.local"
  - name: DB_POSTGRESDB_PORT
    value: "5432"
  - name: DB_POSTGRESDB_DATABASE
    value: "n8n"
  - name: DB_POSTGRESDB_USER
    valueFrom:
      secretKeyRef:
        name: postgres-secrets
        key: username
  - name: DB_POSTGRESDB_PASSWORD
    valueFrom:
      secretKeyRef:
        name: postgres-secrets
        key: password
  - name: N8N_BASIC_AUTH_ACTIVE
    value: "true"
  - name: N8N_BASIC_AUTH_USER
    value: "{{ n8n_admin_email }}"
  - name: N8N_BASIC_AUTH_PASSWORD
    valueFrom:
      secretKeyRef:
        name: n8n-secrets
        key: admin-password
  - name: N8N_ENCRYPTION_KEY
    valueFrom:
      secretKeyRef:
        name: n8n-secrets
        key: encryption-key

persistence:
  enabled: true
  storageClass: local-path
  size: 10Gi
  mountPath: /home/node/.n8n

resources:
  requests:
    cpu: 500m
    memory: 512Mi
  limits:
    cpu: 1000m
    memory: 1Gi

ingress:
  enabled: true
  hosts:
    - host: n8n.dgsynthex.online
      paths:
        - path: /
          pathType: Prefix
  tls:
    - secretName: n8n-tls
      hosts:
        - n8n.dgsynthex.online
```

### Accès Initial

```bash
# https://n8n.dgsynthex.online
# Login:
Username: {{ n8n_admin_email }}
Password: {{ n8n_admin_password }}

# Create workflows
# Connect to external services (Google, GitHub, etc.)
# Schedule executions
# Monitor history
```

### Workflows Recommandés

1. **Auto-backup Database**
   - Cron: Daily 3am
   - PostgreSQL dump → S3/OVH Object Storage

2. **Monitor Cluster Health**
   - Cron: Every 5 minutes
   - Check pod status via Kubernetes API
   - Send alert if unhealthy

3. **Log Aggregation**
   - Trigger: New logs in syslog
   - Parse and store in DBdictribution

4. **Webhook Receiver**
   - GitHub webhooks → N8N → RabbitMQ → N node-RED

### Dépannage

```bash
# Logs
sudo kubectl logs -n apps -l app=n8n -f

# Database connection
sudo kubectl exec -it <n8n-pod> -n apps -- \
  curl http://postgresql.apps.svc.cluster.local:5432

# Reset admin password
sudo kubectl exec -it <n8n-pod> -n apps -- \
  npm run user:changePassword --email=admin@dgsynthex.online
```

---

## 🔴 Application 5: Node-RED - Visual Orchestration

**Chart**: Node-RED community  
**Namespace**: `apps`  
**Access**: https://nodered.dgsynthex.online  
**Default Port**: 1880

### Rôle
Visual programming for IoT and event-driven applications.

### Configuration

```yaml
replicaCount: 1

image:
  repository: nodered/node-red
  tag: latest-alpine

env:
  - name: TZ
    value: "UTC"
  - name: NODE_RED_ENABLE_PROJECTS
    value: "true"

persistence:
  enabled: true
  storageClass: local-path
  size: 5Gi
  mountPath: /data

resources:
  requests:
    cpu: 100m
    memory: 128Mi
  limits:
    cpu: 500m
    memory: 512Mi

ingress:
  enabled: true
  hosts:
    - host: nodered.dgsynthex.online
```

### Nodes Recommandés

```
Core Nodes:
  - inject (trigger)
  - debug (output)
  - function (JS processing)
  - split/join (array handling)

IoT Nodes:
  - mqtt in/out (MQTT protocol)
  - http request
  - websocket

Database:
  - node-red-node-mysql
  - node-red-node-postgres

Dashboard:
  - node-red-dashboard (web UI)
```

### Exemple: MQTT → PostgreSQL

```json
[
  {
    "id": "mqtt_in",
    "type": "mqtt in",
    "topic": "sensors/+/temperature",
    "qos": 1
  },
  {
    "id": "parse_json",
    "type": "json",
    "action": "obj"
  },
  {
    "id": "db_insert",
    "type": "postgres",
    "query": "INSERT INTO sensors (topic, value, timestamp) VALUES($1, $2, NOW())"
  }
]
```

---

## 📡 Application 6: MQTT Broker - IoT Messaging

**Chart**: Bitnami/mosquitto  
**Version**: 4.0.0  
**Namespace**: `apps`  
**Access**: mqtt.dgsynthex.online (port 1883 for MQTT, 8883 for MQTT+TLS)

### Rôle
Lightweight message broker for IoT devices and sensors.

### Configuration

```yaml
auth:
  username: "{{ mqtt_username }}"
  password: "{{ mqtt_password }}"

replicaCount: 1

service:
  type: ClusterIP
  ports:
    - port: 1883
      name: mqtt
    - port: 8883
      name: mqtt-tls
    - port: 9001
      name: websocket

persistence:
  enabled: true
  storageClass: local-path
  size: 5Gi
```

### Client Configuration

```bash
# Python
pip install paho-mqtt

import paho.mqtt.client as mqtt

client = mqtt.Client("client_id")
client.username_pw_set("{{ mqtt_username }}", "{{ mqtt_password }}")
client.connect("mqtt.apps.svc.cluster.local", 1883, 60)

# Publish
client.publish("sensors/living-room/temperature", "22.5")

# Subscribe
def on_message(client, userdata, msg):
    print(f"{msg.topic}: {msg.payload}")

client.on_message = on_message
client.subscribe("sensors/#")
client.loop_forever()
```

### Topics Structure (Recommandé)

```
sensors/
├── living-room/
│   ├── temperature
│   ├── humidity
│   ├── light-level
├── kitchen/
│   ├── temperature
│   ├── motion
├── bedroom/
    └── temperature

devices/
├── heater/
│   ├── status
│   ├── power
├── lights/
    └── brightness

commands/
├── notify
└── alert
```

---

## 🧠 Application 7: Ollama - Local LLM Inference

**Chart**: Ollama community  
**Namespace**: `ollama-system`  
**Access**: https://ollama.dgsynthex.online

### Rôle
Run large language models locally without GPU (or with GPU if available).

### Configuration

```yaml
replicaCount: 1

image:
  repository: ollama/ollama
  tag: latest

env:
  - name: OLLAMA_HOST
    value: "0.0.0.0:11434"
  - name: OLLAMA_MODELS
    value: "/root/.ollama/models"

resources:
  requests:
    cpu: 1000m
    memory: 4Gi  # Highly dependent on model size
  limits:
    cpu: 4000m
    memory: 8Gi

persistence:
  enabled: true
  storageClass: local-path
  size: 50Gi      # IMPORTANT: Models can be large
  mountPath: /root/.ollama

# GPU support (if 'nvidia.com/gpu' available)
# gpuCount: 1
```

### Available Models

```bash
# Pull models
curl http://ollama.svc.cluster.local:11434/api/pull \
  -d '{"model": "llama2"}'

# Lightweight models:
# - tinyllama        (~400MB)
# - mistral          (~4GB)
# - orca-mini        (~1GB)

# Medium models:
# - llama2           (~3.8GB)
# - neural-chat      (~4GB)

# Large models:
# - llama2-70b       (~40GB)
# - mixtral         (~26GB)
```

### API Usage

```python
import requests

# Generate text
response = requests.post(
    'http://ollama.ollama-system.svc.cluster.local:11434/api/generate',
    json={
        'model': 'llama2',
        'prompt': 'What is Kubernetes?',
        'stream': False
    }
)

output = response.json()['response']
print(output)
```

### Integration with N8N

```yaml
# N8N HTTP node calling Ollama
Method: POST
URL: http://ollama.ollama-system.svc.cluster.local:11434/api/generate
Body:
  {
    "model": "mistral",
    "prompt": "{{ triggerData.text }}",
    "stream": false
  }
```

---

## 📊 Application 8: Portainer - Kubernetes Management UI

**Chart**: Portainer CE  
**Version**: 2.21.5  
**Namespace**: `portainer`  
**Access**: https://portainer.dgsynthex.online

### Rôle
Web UI for managing Kubernetes clusters, stacks, and deployments.

### Features

- ✅ **Cluster Management**
  - View nodes, pods, deployments
  - Create/edit/delete resources
  - Monitor resource usage

- ✅ **Registry Management**
  - Connect Docker registries
  - Browse images
  - Deploy from images

- ✅ **Stack Management**
  - Deploy Kubernetes manifests
  - Docker Compose stacks
  - Multi-container orchestration

- ✅ **Access Control**
  - RBAC integration
  - User/team management
  - Role-based views

### Initial Setup

```bash
# 1. Access https://portainer.dgsynthex.online
# 2. Set admin password (first login)
# 3. Select 'Local' environment (already Kubernetes cluster)
# 4. Dashboard shows cluster status

# Create custom dashboards
# Manage namespaces
# Deploy test applications
```

### Dépannage

```bash
# Pods
sudo kubectl get pods -n portainer

# Logs
sudo kubectl logs -n portainer -l app=portainer

# Reset admin password
sudo kubectl exec -it <portainer-pod> -n portainer -- \
  portainer --admin-password=newpassword
```

---

## 📈 Application 9: Prometheus + Grafana - Monitoring Stack

### Prometheus

**Chart**: kube-prometheus-stack (includes Prometheus, Grafana, AlertManager)  
**Namespace**: `monitoring`  
**Version**: 45.0.0

**Rôle**: Metrics collection and storage

```yaml
prometheus:
  prometheusSpec:
    serviceMonitorSelectorNilUsesHelmValues: false
    retention: 30d
    resources:
      requests:
        cpu: 250m
        memory: 512Mi
      limits:
        cpu: 500m
        memory: 1Gi
    storage:
      volumeClaimTemplate:
        spec:
          storageClassName: local-path
          accessModes: ["ReadWriteOnce"]
          resources:
            requests:
              storage: 20Gi
```

**Scrape Targets**
- Kubernetes API server
- Node exporters
- Pod metrics
- Custom ServiceMonitors

### Grafana

**Access**: https://grafana.dgsynthex.online  
**Default Credentials**: admin / {{ grafana_admin_password }}

**Features**:
- Dashboards customization
- Alerting rules
- Data source management
- User access control

**Pre-configured Dashboards**:
```
- Kubernetes Cluster Monitoring
- Node Exporter Full
- Pod CPU and Memory Usage
- Traefik Dashboard
```

### AlertManager

**Configuration**: Routing alerts to email, Slack, webhook

```yaml
alertmanager:
  config:
    global:
      resolve_timeout: 5m
    route:
      receiver: 'default'
      group_by: ['alertname', 'instance']
      group_wait: 10s
      group_interval: 10s
      repeat_interval: 12h
    receivers:
      - name: 'default'
        email_configs:
          - to: admin@dgsynthex.online
            from: alerts@dgsynthex.online
            smarthost: smtp.gmail.com:587
            auth_username: admin@dgsynthex.online
            auth_password: '{{ email_app_password }}'
```

### Custom Metrics

```yaml
# Create custom ServiceMonitor for app
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: app-metrics
  namespace: apps
spec:
  selector:
    matchLabels:
      monitoring: "true"
  endpoints:
  - port: metrics
    interval: 30s
```

---

## 🔄 Orchestration Entre Applications

### Data Flow Example

```
MQTT Broker (IoT sensors)
    ↓
Node-RED (process data)
    ↓
RabbitMQ (queue messages)
    ↓
N8N (orchestrate workflows)
    ↓
PostgreSQL (store results)
    ↓
Grafana (visualize metrics)
```

### Integration Patterns

#### Pattern 1: Event-Driven

```
Sensor → MQTT → Node-RED → RabbitMQ → N8N → Database
```

#### Pattern 2: Scheduled Batch

```
N8N (cron) → API call → N8N → S3 backup → Email notification
```

#### Pattern 3: Webhook-Driven

```
GitHub → N8N webhook → Build pipeline → Deploy to K8s → Slack notification
```

---

Voir aussi: [INSTALLATION.md](INSTALLATION.md), [ROLES.md](ROLES.md), [ARCHITECTURE.md](ARCHITECTURE.md)
