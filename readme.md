# Kubernetes Deployment Guide

## Overview

This project deploys a complete Kubernetes stack including:

- Nginx website with HTTPS support
- Reverse proxy for S3 website at `/hello` path
- MongoDB v3.6 replica set (3 nodes)
- Prometheus for MongoDB metrics
- Grafana for metrics visualization

## Prerequisites

- MicroK8s installed and running
- Domain: `thebrainsurf.site` pointing to server IP
- Ports open: 80, 443, 22
- Nginx Ingress Controller enabled

## Access Information

### URLs

- **Main Website**: https://thebrainsurf.site
- **S3 Proxy**: https://thebrainsurf.site/hello
- **Grafana**: https://grafana.thebrainsurf.site
- **Prometheus**: https://prometheus.thebrainsurf.site

### Credentials

- **Grafana**:
  - Username: `admin`
  - Password: `admin` (change on first login)

## Deployment Steps

### 1. Clone Repository

```bash
cd ~
git clone <repository-url> candidate-432
cd candidate-432
```

### 2. Setup Cert-Manager (for HTTPS)

```bash
# Enable cert-manager
microk8s enable cert-manager

# Wait for cert-manager to be ready
microk8s kubectl wait --for=condition=ready pod -l app=cert-manager -n cert-manager --timeout=300s

# Create Cloudflare API token secret
microk8s kubectl create secret generic cloudflare-api-token \
  --from-literal=api-token=YOUR_CLOUDFLARE_API_TOKEN \
  -n cert-manager

microk8s kubectl create secret generic cloudflare-api-token \
  --from-literal=api-token=YOUR_CLOUDFLARE_API_TOKEN \
  -n default

microk8s kubectl create secret generic cloudflare-api-token \
  --from-literal=api-token=YOUR_CLOUDFLARE_API_TOKEN \
  -n monitoring

# Apply ClusterIssuer and Certificate
microk8s kubectl apply -f cert-manager/cloudflare-issuer.yaml
microk8s kubectl apply -f cert-manager/wildcard-cert.yaml
```

### 3. Deploy Nginx Website

```bash
# Deploy nginx web server
microk8s kubectl apply -f nginx/deployment.yaml
microk8s kubectl apply -f nginx/service.yaml
```

### 4. Deploy Reverse Proxy for S3

```bash
# Deploy proxy configuration and service
microk8s kubectl apply -f proxy/configmap.yaml
microk8s kubectl apply -f proxy/deployment.yaml
microk8s kubectl apply -f proxy/service.yaml
```

### 5. Setup Ingress

```bash
# Apply ingress rules
microk8s kubectl apply -f ingress/nginx-ingress.yaml
```

### 6. Deploy MongoDB Replica Set

```bash
# Deploy MongoDB StatefulSet and Service
microk8s kubectl apply -f mongo/service.yaml
microk8s kubectl apply -f mongo/statefulset.yaml

# Wait for pods to be ready
microk8s kubectl wait --for=condition=ready pod -l app=mongo --timeout=300s

# Initialize replica set
microk8s kubectl exec -it mongo-0 -- mongo --eval '
rs.initiate({
  _id: "rs0",
  members: [
    { _id: 0, host: "mongo-0.mongo.default.svc.cluster.local:27017" },
    { _id: 1, host: "mongo-1.mongo.default.svc.cluster.local:27017" },
    { _id: 2, host: "mongo-2.mongo.default.svc.cluster.local:27017" }
  ]
})'

# Verify replica set status
microk8s kubectl exec -it mongo-0 -- mongo --eval 'rs.status()'
```

### 7. Deploy Monitoring Stack

#### Create Monitoring Namespace

```bash
microk8s kubectl apply -f monitoring/namespace.yaml
```

#### Deploy MongoDB Exporter

```bash
microk8s kubectl apply -f monitoring/mongodb-exporter/mongodb-exporter.yaml
microk8s kubectl apply -f monitoring/mongodb-exporter/service.yaml
```

#### Deploy Prometheus

```bash
microk8s kubectl apply -f monitoring/prometheus/prometheus-config.yaml
microk8s kubectl apply -f monitoring/prometheus/prometheus.yaml
microk8s kubectl apply -f monitoring/prometheus/service.yaml
microk8s kubectl apply -f monitoring/prometheus/prometheus-ingress.yaml
```

#### Deploy Grafana

```bash
microk8s kubectl apply -f monitoring/grafana/grafana-deployment.yaml
microk8s kubectl apply -f monitoring/grafana/service.yaml
microk8s kubectl apply -f monitoring/grafana/grafana-ingress.yaml
```

### 8. Verify Deployment

```bash
# Check all pods
microk8s kubectl get pods -A

# Check services
microk8s kubectl get svc -A

# Check ingress
microk8s kubectl get ingress -A

# Check certificates
microk8s kubectl get certificate -n monitoring
```

## Configuration Details

### MongoDB Replica Set

- **Replica Set Name**: rs0
- **Nodes**: 3 (mongo-0, mongo-1, mongo-2)
- **Version**: 3.6
- **Storage**: 1Gi per node (hostPath)

### Prometheus

- **Scrape Interval**: 15s
- **Targets**: MongoDB Exporter (mongodb-exporter:9216)
- **Metrics**: MongoDB replica set metrics

### Grafana

- **Data Source**: Prometheus (http://prometheus:9090)
- **Default Dashboard**: MongoDB metrics

## Troubleshooting

### Certificate Issues

```bash
# Check certificate status
microk8s kubectl get certificate -n monitoring
microk8s kubectl describe certificate wildcard-thebrainsurf -n monitoring

# Check cert-manager logs
microk8s kubectl logs -n cert-manager -l app=cert-manager --tail=100
```

### MongoDB Issues

```bash
# Check replica set status
microk8s kubectl exec -it mongo-0 -- mongo --eval 'rs.status()'

# Check logs
microk8s kubectl logs mongo-0
```

### Prometheus/Grafana Issues

```bash
# Check Prometheus targets
curl http://localhost:9090/api/v1/targets

# Check Grafana logs
microk8s kubectl logs -n monitoring -l app=grafana
```

## Cleanup

```bash
# Delete all resources
microk8s kubectl delete -f monitoring/
microk8s kubectl delete -f mongo/
microk8s kubectl delete -f proxy/
microk8s kubectl delete -f nginx/
microk8s kubectl delete -f ingress/
microk8s kubectl delete -f cert-manager/
```

## Notes

- SSL certificates are automatically managed by cert-manager using Let's Encrypt
- Cloudflare DNS01 challenge is used for wildcard certificates
- MongoDB data is persisted using hostPath storage
- All services are accessible via HTTPS with valid certificates
