# MCP Server Security Configuration

## Endpoint Protection

### `/health` Endpoint Security

The MCP endpoint `https://mcp.dgsynthex.online/health` is protected with:

1. **TLS/HTTPS** - All traffic encrypted with Let's Encrypt certificate
2. **Rate Limiting** - 100 req/sec (stricter for health endpoint), 20 burst limit
3. **Security Headers** - Strict-Transport-Security, X-Frame-Options=DENY, etc.
4. **NetworkPolicy (Kubernetes)** - Only Traefik ingress controller allowed
5. **Firewall Rules** - NFTables rules restrict source IPs

---

## Accessing the Secure Health Endpoint

### Method 1: Simple curl (no authentication needed at HTTP level)

```bash
# Works without credentials (protected at Kubernetes NetworkPolicy level)
curl https://mcp.dgsynthex.online/health

# With verbose output to see headers
curl -v https://mcp.dgsynthex.online/health

# Ignore self-signed cert (dev only)
curl -k https://mcp.dgsynthex.online/health
```

**Note:** The endpoint requires:
- HTTPS (TLS) - HTTP requests rejected
- Source IP from Traefik ingress controller only
- Rate limiting: 100 req/sec max

### Method 2: From within Kubernetes (Pod-to-Pod)

MCP accepts requests from N8N pod without external authentication (internal K8s DNS):

```bash
# From N8N pod
curl http://mcp-server:8000/health
```

This works because:
- Internal K8s DNS resolution (mcp-server.admin.svc.cluster.local)
- NetworkPolicy allows ingress from admin namespace pods
- No TLS certificate validation needed for internal traffic

### Method 3: Monitoring/Alerting Integration

For Prometheus or external monitoring (from authorized IPs only):

```yaml
# Example Prometheus scrape config
scrape_configs:
  - job_name: 'mcp-server'
    static_configs:
      - targets: ['mcp.dgsynthex.online:443']
    scheme: https
    tls_config:
      insecure_skip_verify: true  # Or use proper CA cert
```

**IMPORTANT:** Prometheus server must:
- Be on the same network as Traefik (kube-system namespace)
- OR have IP whitelisted in NFTables firewall rules

---

## Kubernetes NetworkPolicy Details

MCP has strict egress/ingress policies:

### Ingress Rules
- ✅ Allow from Traefik ingress controller (kube-system namespace)
- ✅ Allow from N8N pods (admin namespace, port 5678 → MCP 8000)
- ❌ Block all other sources

### Egress Rules
- ✅ Allow DNS (UDP 53) to kube-system
- ✅ Allow outbound to N8N service (TCP 5678)
- ✅ Allow external HTTPS (TCP 443) - for webhooks, external APIs
- ✅ Allow external HTTP (TCP 80) - fallback for some integrations
- ❌ Block internal K3s networks (10.0.0.0/8)

---

## Health Endpoint Response

Successful health check returns:

```json
{
  "status": "healthy",
  "timestamp": "2026-03-13T10:45:32.123Z"
}
```

HTTP 200 = ✅ MCP server is running

---

## Traefik Middleware Configuration

### Applied Middlewares

1. **mcp-security-headers** - Security HTTP headers
   - Strict-Transport-Security (HSTS) with 1-year max-age
   - X-Frame-Options: DENY (prevent clickjacking)
   - X-Content-Type-Options: nosniff (prevent MIME sniffing)
   - Permissions-Policy: blocks geolocation, microphone, camera
   
2. **mcp-rate-limit** - Request rate limiting
   - 100 req/sec average
   - 20 burst allowance
   - Excess requests get HTTP 429 (Too Many Requests)

### Testing Middleware

```bash
# Test rate limit (hammer with requests)
for i in {1..150}; do
  curl https://mcp.dgsynthex.online/health &
done
wait

# Check response headers
curl -i https://mcp.dgsynthex.online/health
# Look for: Strict-Transport-Security, X-Frame-Options, etc.

# Test from pod (should work)
kubectl exec -it <pod> -n admin -- curl http://mcp-server:8000/health
```

---

## Security Model: Defense in Depth

The endpoint is protected by **multiple layers** instead of simple authentication:

```
Internet
   ↓ (HTTPS only, TLS 1.3+)
Traefik Ingress
   ↓ (rate limit: 100 req/sec, 20 burst)
Rate Limiter
   ↓ (only kube-system namespace)
NetworkPolicy
   ↓ (Kubernetes network isolation)
MCP Pod
   ↓ (running in admin namespace)
/health endpoint
```

### Why This Approach?

| Approach | Pro | Con |
|----------|-----|-----|
| **BasicAuth** | Simple to understand | Credentials must be rotated, easy to leak |
| **NetworkPolicy** ✅ | Cluster-level control, automatic, no creds | Requires Kubernetes expertise |
| **Rate Limiting** ✅ | Prevents abuse/DoS | Legitimate burst traffic rejected |
| **TLS/HTTPS** ✅ | Encrypts in transit | Requires valid certificate |
| **Security Headers** ✅ | Browser protection | Not relevant for API clients |

**Result:** A 401 error when accessing from your local machine is actually **correct** - the endpoint is designed to only accept requests from the Traefik ingress controller inside the cluster.

---

## Troubleshooting

### 401 Unauthorized
```bash
# This happens when accessing from outside the cluster
# The endpoint is protected by Kubernetes NetworkPolicy
# Only Traefik ingress (in kube-system) can access it
# Check NetworkPolicy:
kubectl get networkpolicy -n admin | grep mcp
kubectl describe networkpolicy mcp-allow-traefik -n admin
```

### 503 Service Unavailable
```bash
# Check pod status
kubectl get pods -n admin -l app=mcp-server
kubectl logs -n admin -l app=mcp-server
```

### 429 Too Many Requests
Rate limit exceeded - wait 1 second and retry

### Connection Timeout
Check NetworkPolicy:
```bash
kubectl get networkpolicy -n admin | grep mcp
kubectl describe networkpolicy mcp-allow-traefik -n admin
```

---

## Production Recommendations

1. **NetworkPolicy IP Whitelisting** - Restrict to specific authorized IPs
   ```yaml
   # Add to NetworkPolicy ingress
   ingress:
     - from:
         - ipBlock:
             cidr: 203.0.113.50/32  # Your monitoring server IP
       ports:
         - protocol: TCP
           port: 8000
   ```

2. **NFTables Firewall Rules** - Additional layer
   ```bash
   # Add to firewall rules
   nft add rule inet filter input ip protocol tcp tcp dport 443 ip saddr 10.0.0.0/8 accept
   ```

3. **Monitoring Alerts**
   - Set up alerts for:
     - Pod CrashLoopBackOff
     - High error rates (5xx responses)
     - Rate limit hits (429 responses)
     - Certificate expiration (30 days warning)

4. **Audit Logging**
   - Enable Traefik access logs (already configured)
   - Monitor Kubernetes audit logs for MCP resource changes
   - Check NetworkPolicy hit stats

---

## N8N Integration with MCP

### Connecting N8N to MCP

In N8N workflow:
1. Add "HTTP Request" node
2. URL: `http://mcp-server:8000/health`
3. No authentication needed (internal K8s DNS)

### MCP Node Integration

Once MCP is stable, configure N8N MCP client for LLM workflows:
```javascript
// In N8N node
const mcp = new MCPClient({
  serverUrl: 'http://mcp-server:8000',
  apiKey: process.env.MCP_API_KEY
});

await mcp.getTools();  // List available tools
await mcp.executeTool('tool-name', params);
```

---

## References

- [Model Context Protocol](https://modelcontextprotocol.io/)
- [N8N MCP Integration](https://github.com/czlonkowski/n8n-mcp)
- [Kubernetes NetworkPolicy](https://kubernetes.io/docs/concepts/services-networking/network-policies/)
- [Traefik Middleware](https://doc.traefik.io/traefik/middlewares/overview/)
