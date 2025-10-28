# Quick Reference Guide

This guide provides a quick reference for the Kubernetes deployment of Gatus with Grafana and Prometheus.

## Resource Summary

The `gatus-grafana-prometheus.yaml` file contains:

### Namespaces
- `monitoring` - Contains all resources

### ConfigMaps
1. `gatus-config` - Gatus application configuration
2. `prometheus-config` - Prometheus scrape configuration
3. `grafana-config` - Grafana main configuration (grafana.ini)
4. `grafana-datasources` - Grafana datasource provisioning (Prometheus)
5. `grafana-dashboard-provider` - Grafana dashboard provider configuration
6. `grafana-dashboard-gatus` - Gatus monitoring dashboard JSON

### Deployments
1. `gatus` - Health monitoring service
2. `prometheus` - Metrics collection and storage
3. `grafana` - Visualization and dashboards

### Services
1. `gatus` (LoadBalancer) - Exposes port 8080
2. `prometheus` (ClusterIP) - Internal service on port 9090
3. `grafana` (LoadBalancer) - Exposes port 3000

## Quick Commands

### Deploy
```bash
kubectl apply -f gatus-grafana-prometheus.yaml
```

### Check Status
```bash
kubectl get all -n monitoring
```

### Get External IPs
```bash
kubectl get svc -n monitoring
```

### View Logs
```bash
kubectl logs -n monitoring -l app=gatus --tail=50
kubectl logs -n monitoring -l app=prometheus --tail=50
kubectl logs -n monitoring -l app=grafana --tail=50
```

### Port Forward (for local access)
```bash
kubectl port-forward -n monitoring svc/gatus 8080:8080 &
kubectl port-forward -n monitoring svc/grafana 3000:3000 &
kubectl port-forward -n monitoring svc/prometheus 9090:9090 &
```

### Update Configuration
```bash
# Edit Gatus config
kubectl edit configmap gatus-config -n monitoring

# Restart Gatus to apply changes
kubectl rollout restart deployment/gatus -n monitoring
```

### Scale Deployments
```bash
kubectl scale deployment grafana -n monitoring --replicas=2
```

### Delete Everything
```bash
kubectl delete -f gatus-grafana-prometheus.yaml
# OR
kubectl delete namespace monitoring
```

## Service Endpoints

After deployment, access services via:

- **Gatus**: http://\<EXTERNAL-IP\>:8080
- **Grafana**: http://\<EXTERNAL-IP\>:3000
- **Prometheus**: Internal only (or use port-forward)

## Configuration File Locations

Inside containers:

- **Gatus**: `/config/config.yaml`
- **Prometheus**: `/etc/prometheus/prometheus.yml`
- **Grafana**: 
  - `/etc/grafana/grafana.ini`
  - `/etc/grafana/provisioning/datasources/prometheus.yml`
  - `/etc/grafana/provisioning/dashboards/dashboard.yml`
  - `/etc/grafana/provisioning/dashboards/gatus.json`

## Default Settings

### Resource Limits

**Gatus:**
- Requests: 50m CPU, 30Mi memory
- Limits: 250m CPU, 100Mi memory

**Prometheus:**
- Requests: 100m CPU, 128Mi memory
- Limits: 500m CPU, 512Mi memory

**Grafana:**
- Requests: 100m CPU, 128Mi memory
- Limits: 500m CPU, 512Mi memory

### Grafana Credentials
- Anonymous access enabled with Admin role
- Admin password: `secret`

## Comparison with Docker Compose

| Docker Compose | Kubernetes |
|----------------|------------|
| `docker-compose up` | `kubectl apply -f gatus-grafana-prometheus.yaml` |
| `docker-compose down` | `kubectl delete -f gatus-grafana-prometheus.yaml` |
| `docker-compose logs` | `kubectl logs -n monitoring -l app=<service>` |
| `docker-compose ps` | `kubectl get pods -n monitoring` |
| Port 8080 on host | LoadBalancer service on port 8080 |
| Port 3000 on host | LoadBalancer service on port 3000 |
| Port 9090 on host | ClusterIP (internal only) |
| Volume mounts | ConfigMaps |
| Bridge network | Kubernetes service DNS |

## Network Communication

In Kubernetes, services communicate using DNS names:

- Prometheus scrapes Gatus at: `http://gatus:8080/metrics`
- Grafana queries Prometheus at: `http://prometheus:9090`

This replaces Docker Compose service names with Kubernetes service DNS.

## Troubleshooting

### Pods not starting
```bash
kubectl describe pod -n monitoring <pod-name>
kubectl logs -n monitoring <pod-name>
```

### ConfigMap not loading
```bash
kubectl describe configmap -n monitoring <configmap-name>
```

### Service not accessible
```bash
kubectl get svc -n monitoring
kubectl describe svc -n monitoring <service-name>
```

### Test internal connectivity
```bash
kubectl exec -n monitoring -it deployment/prometheus -- wget -O- http://gatus:8080/metrics
```
