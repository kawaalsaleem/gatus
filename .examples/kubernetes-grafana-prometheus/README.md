# Gatus with Grafana and Prometheus on Azure Kubernetes Service (AKS)

This example demonstrates how to deploy Gatus with Grafana and Prometheus on Azure Kubernetes Service (AKS). The setup provides a complete monitoring stack with:

- **Gatus**: Health monitoring and uptime tracking
- **Prometheus**: Metrics collection and storage
- **Grafana**: Visualization and dashboards

## Prerequisites

- Azure subscription with an AKS cluster
- `kubectl` configured to access your AKS cluster
- Access to create LoadBalancer services (or modify to use Ingress)

## Quick Start

### 1. Apply the Kubernetes manifests

Deploy all resources with a single command:

```bash
kubectl apply -f gatus-grafana-prometheus.yaml
```

This will create:
- A `monitoring` namespace
- ConfigMaps for all configurations (Gatus, Prometheus, Grafana)
- Deployments for Gatus, Prometheus, and Grafana
- Services for external access

### 2. Wait for pods to be ready

```bash
kubectl wait --for=condition=ready pod -l app=gatus -n monitoring --timeout=120s
kubectl wait --for=condition=ready pod -l app=prometheus -n monitoring --timeout=120s
kubectl wait --for=condition=ready pod -l app=grafana -n monitoring --timeout=120s
```

### 3. Get service endpoints

#### Option A: Using LoadBalancer (default)

Wait for the LoadBalancer IP addresses to be assigned:

```bash
kubectl get svc -n monitoring
```

You should see output similar to:

```
NAME         TYPE           CLUSTER-IP     EXTERNAL-IP      PORT(S)          AGE
gatus        LoadBalancer   10.0.123.45    20.30.40.50      8080:30123/TCP   2m
grafana      LoadBalancer   10.0.123.46    20.30.40.51      3000:30124/TCP   2m
prometheus   ClusterIP      10.0.123.47    <none>           9090/TCP         2m
```

Access the services:
- **Gatus UI**: `http://<GATUS-EXTERNAL-IP>:8080`
- **Grafana**: `http://<GRAFANA-EXTERNAL-IP>:3000`
- **Prometheus**: Accessible only within the cluster at `http://prometheus:9090`

#### Option B: Using Port Forwarding (for testing)

If you prefer not to use LoadBalancer or want to test locally:

```bash
# Gatus
kubectl port-forward -n monitoring svc/gatus 8080:8080

# Grafana (in a new terminal)
kubectl port-forward -n monitoring svc/grafana 3000:3000

# Prometheus (in a new terminal)
kubectl port-forward -n monitoring svc/prometheus 9090:9090
```

Then access:
- **Gatus UI**: http://localhost:8080
- **Grafana**: http://localhost:3000
- **Prometheus**: http://localhost:9090

## Default Credentials

### Grafana
- **Username**: Not required (anonymous access enabled)
- **Password**: Not required (anonymous access enabled)
- **Admin Password** (if needed): `secret`

Anonymous access is enabled with admin role for quick setup. For production, disable anonymous access and use proper authentication.

## Configuration

### Customizing Gatus Configuration

Edit the `gatus-config` ConfigMap to add or modify monitored endpoints:

```bash
kubectl edit configmap gatus-config -n monitoring
```

Or create a custom config file and update the ConfigMap:

```bash
kubectl create configmap gatus-config -n monitoring --from-file=config.yaml=./my-config.yaml --dry-run=client -o yaml | kubectl apply -f -
```

After updating, restart Gatus to apply changes:

```bash
kubectl rollout restart deployment/gatus -n monitoring
```

### Customizing Prometheus Scrape Configuration

Edit the `prometheus-config` ConfigMap:

```bash
kubectl edit configmap prometheus-config -n monitoring
```

Restart Prometheus after changes:

```bash
kubectl rollout restart deployment/prometheus -n monitoring
```

### Customizing Grafana

- **Dashboards**: Edit the `grafana-dashboard-gatus` ConfigMap
- **Datasources**: Edit the `grafana-datasources` ConfigMap
- **Settings**: Edit the `grafana-config` ConfigMap

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                      AKS Cluster                             │
│  ┌───────────────────────────────────────────────────────┐  │
│  │                  Monitoring Namespace                  │  │
│  │                                                        │  │
│  │  ┌──────────┐      ┌──────────────┐    ┌──────────┐  │  │
│  │  │  Gatus   │─────▶│  Prometheus  │◀───│ Grafana  │  │  │
│  │  │  :8080   │      │    :9090     │    │  :3000   │  │  │
│  │  └────┬─────┘      └──────────────┘    └────┬─────┘  │  │
│  │       │                                      │        │  │
│  │       │ /metrics                             │ Query  │  │
│  │       │                                      │        │  │
│  │  ┌────▼──────────────────────────────────────▼─────┐  │  │
│  │  │           ConfigMaps (Configs & Data)          │  │  │
│  │  └────────────────────────────────────────────────┘  │  │
│  └───────────────────────────────────────────────────────┘  │
│                                                              │
│  ┌──────────────┐                          ┌──────────────┐ │
│  │ LoadBalancer │                          │ LoadBalancer │ │
│  │   (Gatus)    │                          │  (Grafana)   │ │
│  └──────┬───────┘                          └──────┬───────┘ │
└─────────┼──────────────────────────────────────────┼─────────┘
          │                                          │
          ▼                                          ▼
    External Access                           External Access
```

## Monitoring Flow

1. **Gatus** monitors configured endpoints and exposes metrics at `/metrics`
2. **Prometheus** scrapes metrics from Gatus every 10 seconds
3. **Grafana** queries Prometheus and displays metrics on dashboards
4. The pre-configured dashboard shows:
   - Certificate expiration warnings
   - Overall availability percentage
   - Number of monitored services
   - Checks per minute
   - Success rate per service
   - Service status overview table
   - Response times graph

## Production Considerations

### Security

1. **Disable Grafana Anonymous Access**: Edit `grafana-config` ConfigMap and set:
   ```ini
   [auth.anonymous]
   enabled = false
   ```

2. **Use Secrets for Passwords**: Replace the plain-text admin password with a Kubernetes Secret:
   ```bash
   kubectl create secret generic grafana-admin -n monitoring --from-literal=password=your-secure-password
   ```
   
   Then update the Grafana deployment to use the secret.

3. **Network Policies**: Implement network policies to restrict traffic between pods.

### High Availability

1. **Increase Replicas**: For Grafana and Prometheus (note: Prometheus HA requires additional configuration)
   ```bash
   kubectl scale deployment grafana -n monitoring --replicas=2
   ```

2. **Persistent Storage**: Add PersistentVolumeClaims for Prometheus and Grafana to preserve data across restarts:
   ```yaml
   volumes:
   - name: prometheus-storage
     persistentVolumeClaim:
       claimName: prometheus-pvc
   ```

### Ingress

Instead of LoadBalancer services, use an Ingress controller (like NGINX Ingress) for better control and cost efficiency:

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: monitoring-ingress
  namespace: monitoring
  annotations:
    cert-manager.io/cluster-issuer: letsencrypt-prod
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - gatus.yourdomain.com
    - grafana.yourdomain.com
    secretName: monitoring-tls
  rules:
  - host: gatus.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: gatus
            port:
              number: 8080
  - host: grafana.yourdomain.com
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: grafana
            port:
              number: 3000
```

### Resource Limits

The default resource limits are conservative. Adjust based on your monitoring scale:

- Increase Prometheus resources if monitoring many services
- Increase Grafana resources if you have many concurrent users
- Gatus is lightweight and rarely needs adjustment

## Troubleshooting

### Check pod status
```bash
kubectl get pods -n monitoring
```

### View logs
```bash
# Gatus logs
kubectl logs -n monitoring -l app=gatus

# Prometheus logs
kubectl logs -n monitoring -l app=prometheus

# Grafana logs
kubectl logs -n monitoring -l app=grafana
```

### Verify ConfigMaps
```bash
kubectl get configmaps -n monitoring
kubectl describe configmap gatus-config -n monitoring
```

### Test connectivity
```bash
# Test if Prometheus can reach Gatus
kubectl exec -n monitoring -it deployment/prometheus -- wget -O- http://gatus:8080/metrics

# Test if Grafana can reach Prometheus
kubectl exec -n monitoring -it deployment/grafana -- wget -O- http://prometheus:9090/api/v1/status/config
```

## Cleanup

To remove all resources:

```bash
kubectl delete -f gatus-grafana-prometheus.yaml
```

Or delete the namespace (this removes everything):

```bash
kubectl delete namespace monitoring
```

## Useful Prometheus Queries

Access Prometheus at `http://prometheus:9090` (via port-forward) and try these queries:

### Success Rate
```promql
sum(rate(gatus_results_total{success="true"}[30s])) by (key) / sum(rate(gatus_results_total[30s])) by (key)
```

### Response Time
```promql
gatus_results_duration_seconds
```

### Total Results per Minute
```promql
sum(rate(gatus_results_total[5m])*60) by (key)
```

### Total Successful Results per Minute
```promql
sum(rate(gatus_results_total{success="true"}[5m])*60) by (key)
```

### Total Unsuccessful Results per Minute
```promql
sum(rate(gatus_results_total{success="false"}[5m])*60) by (key)
```

## Additional Resources

- [Gatus Documentation](https://github.com/TwiN/gatus)
- [Prometheus Documentation](https://prometheus.io/docs/)
- [Grafana Documentation](https://grafana.com/docs/)
- [AKS Documentation](https://docs.microsoft.com/en-us/azure/aks/)
