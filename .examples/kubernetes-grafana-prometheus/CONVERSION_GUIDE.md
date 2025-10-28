# Docker Compose to Kubernetes Conversion Guide

This document explains how each element from the Docker Compose configuration has been converted to Kubernetes resources.

## Overview

The Docker Compose file at `.examples/docker-compose-grafana-prometheus/docker-compose.yml` has been fully converted to Kubernetes manifests suitable for Azure Kubernetes Service (AKS) deployment.

## Conversion Mapping

### Docker Compose Services → Kubernetes Resources

#### 1. Gatus Service

**Docker Compose:**
```yaml
services:
  gatus:
    image: twinproduction/gatus
    restart: always
    ports:
      - "8080:8080"
    volumes:
      - ./config:/config
    networks:
      - metrics
```

**Kubernetes Resources:**
- **Deployment**: `gatus` with image `twinproduction/gatus:latest`
- **Service**: `gatus` (LoadBalancer) exposing port 8080
- **ConfigMap**: `gatus-config` containing `config.yaml`
- **Volume Mount**: ConfigMap mounted at `/config`

#### 2. Prometheus Service

**Docker Compose:**
```yaml
services:
  prometheus:
    image: prom/prometheus:v3.5.0
    restart: always
    command: --config.file=/etc/prometheus/prometheus.yml
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus/prometheus.yml:/etc/prometheus/prometheus.yml
    networks:
      - metrics
```

**Kubernetes Resources:**
- **Deployment**: `prometheus` with image `prom/prometheus:v3.5.0`
- **Service**: `prometheus` (ClusterIP) exposing port 9090
- **ConfigMap**: `prometheus-config` containing `prometheus.yml`
- **Volume Mount**: ConfigMap mounted at `/etc/prometheus`
- **Command Args**: `--config.file=/etc/prometheus/prometheus.yml`

#### 3. Grafana Service

**Docker Compose:**
```yaml
services:
  grafana:
    image: grafana/grafana:12.1.0
    restart: always
    environment:
      GF_SECURITY_ADMIN_PASSWORD: secret
    ports:
      - "3000:3000"
    volumes:
      - ./grafana/grafana.ini/:/etc/grafana/grafana.ini:ro
      - ./grafana/provisioning/:/etc/grafana/provisioning/:ro
    networks:
      - metrics
```

**Kubernetes Resources:**
- **Deployment**: `grafana` with image `grafana/grafana:12.1.0`
- **Service**: `grafana` (LoadBalancer) exposing port 3000
- **ConfigMaps**: 
  - `grafana-config` (grafana.ini)
  - `grafana-datasources` (datasource provisioning)
  - `grafana-dashboard-provider` (dashboard provider)
  - `grafana-dashboard-gatus` (dashboard JSON)
- **Volume Mounts**: Multiple ConfigMaps mounted at respective paths
- **Environment Variable**: `GF_SECURITY_ADMIN_PASSWORD=secret`

### Networks

**Docker Compose:**
```yaml
networks:
  metrics:
    driver: bridge
```

**Kubernetes:**
- Replaced with Kubernetes Service DNS
- Services communicate via DNS names: `gatus`, `prometheus`, `grafana`
- All services in the same namespace can communicate directly

### Volume Mounts → ConfigMaps

All volume mounts have been converted to ConfigMaps:

| Docker Compose Volume | Kubernetes ConfigMap | Mount Path |
|-----------------------|---------------------|------------|
| `./config` | `gatus-config` | `/config` |
| `./prometheus/prometheus.yml` | `prometheus-config` | `/etc/prometheus/prometheus.yml` |
| `./grafana/grafana.ini` | `grafana-config` | `/etc/grafana/grafana.ini` |
| `./grafana/provisioning/datasources/` | `grafana-datasources` | `/etc/grafana/provisioning/datasources` |
| `./grafana/provisioning/dashboards/` | `grafana-dashboard-provider` + `grafana-dashboard-gatus` | `/etc/grafana/provisioning/dashboards` |

### Port Mappings → Services

| Docker Compose | Kubernetes Service Type | Access Method |
|----------------|------------------------|---------------|
| `8080:8080` (host:container) | LoadBalancer on port 8080 | External IP or Ingress |
| `9090:9090` (host:container) | ClusterIP on port 9090 | Internal only |
| `3000:3000` (host:container) | LoadBalancer on port 3000 | External IP or Ingress |

### Restart Policy

**Docker Compose:**
```yaml
restart: always
```

**Kubernetes:**
- Implicit in Deployments (pods are automatically restarted)
- Default `restartPolicy: Always` in pod spec

## Additional Kubernetes Features

The Kubernetes deployment includes several enhancements not present in Docker Compose:

### Resource Limits
```yaml
resources:
  limits:
    cpu: 250m
    memory: 100Mi
  requests:
    cpu: 50m
    memory: 30Mi
```

### Health Probes
```yaml
readinessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 5
  periodSeconds: 10

livenessProbe:
  httpGet:
    path: /health
    port: 8080
  initialDelaySeconds: 10
  periodSeconds: 10
```

### Labels
```yaml
labels:
  app: gatus
```

### Namespace Isolation
```yaml
metadata:
  namespace: monitoring
```

## Configuration Files Included

All configuration files referenced in the Docker Compose volumes are included:

1. **Gatus Config** (`config/config.yaml`):
   - Metrics enabled
   - Example endpoints (twin.sh, example.com, github.com, domain expiration check)

2. **Prometheus Config** (`prometheus/prometheus.yml`):
   - Scrape config for Gatus on port 8080
   - 10-second scrape interval

3. **Grafana Config** (`grafana/grafana.ini`):
   - Anonymous access enabled
   - Reporting disabled
   - Console logging
   - Default light theme

4. **Grafana Datasource** (`grafana/provisioning/datasources/prometheus.yml`):
   - Prometheus datasource at `http://prometheus:9090`
   - Set as default

5. **Dashboard Provider** (`grafana/provisioning/dashboards/dashboard.yml`):
   - Auto-provisioning from `/etc/grafana/provisioning/dashboards`

6. **Gatus Dashboard** (`grafana/provisioning/dashboards/gatus.json`):
   - Complete Grafana dashboard with panels for:
     - Certificate warnings
     - Overall availability
     - Monitored services count
     - Checks per minute
     - Success rate
     - Service status overview
     - Response times

## Usage Comparison

### Start Services

**Docker Compose:**
```bash
cd .examples/docker-compose-grafana-prometheus
docker-compose up
```

**Kubernetes:**
```bash
cd .examples/kubernetes-grafana-prometheus
kubectl apply -f gatus-grafana-prometheus.yaml
```

### Stop Services

**Docker Compose:**
```bash
docker-compose down
```

**Kubernetes:**
```bash
kubectl delete -f gatus-grafana-prometheus.yaml
# OR
kubectl delete namespace monitoring
```

### View Logs

**Docker Compose:**
```bash
docker-compose logs gatus
docker-compose logs prometheus
docker-compose logs grafana
```

**Kubernetes:**
```bash
kubectl logs -n monitoring -l app=gatus
kubectl logs -n monitoring -l app=prometheus
kubectl logs -n monitoring -l app=grafana
```

### Check Status

**Docker Compose:**
```bash
docker-compose ps
```

**Kubernetes:**
```bash
kubectl get pods -n monitoring
kubectl get all -n monitoring
```

## Network Communication

### Docker Compose
Services communicate using container names over the `metrics` bridge network:
- Prometheus scrapes: `http://gatus:8080/metrics`
- Grafana queries: `http://prometheus:9090`

### Kubernetes
Services communicate using DNS names provided by Kubernetes:
- Prometheus scrapes: `http://gatus:8080/metrics`
- Grafana queries: `http://prometheus:9090`

The DNS names are the same, making the transition seamless!

## Benefits of Kubernetes Deployment

1. **Scalability**: Easy to scale deployments with `kubectl scale`
2. **High Availability**: Kubernetes ensures pods are always running
3. **Health Monitoring**: Automatic health checks and restarts
4. **Resource Management**: CPU and memory limits prevent resource exhaustion
5. **Declarative Configuration**: Infrastructure as code with YAML
6. **Rolling Updates**: Zero-downtime deployments
7. **Namespace Isolation**: Logical separation of resources
8. **Integration**: Works with other Kubernetes tools (Ingress, NetworkPolicy, etc.)

## Production Recommendations

For production use, consider:

1. **Use Ingress** instead of LoadBalancer services for cost efficiency
2. **Add Persistent Volumes** for Prometheus and Grafana data persistence
3. **Implement RBAC** for proper access control
4. **Use Secrets** for sensitive data instead of ConfigMaps
5. **Enable TLS/SSL** for secure communication
6. **Add NetworkPolicies** to restrict traffic between pods
7. **Use HorizontalPodAutoscaler** for automatic scaling
8. **Implement backup strategies** for important data
9. **Add monitoring and alerting** for the monitoring stack itself
10. **Use proper resource limits** based on actual usage patterns
