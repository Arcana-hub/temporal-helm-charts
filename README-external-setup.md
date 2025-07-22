# Temporal Helm Chart - External Cassandra and Elasticsearch Configuration

This configuration file (`values-external-cassandra-elasticsearch.yaml`) sets up Temporal to use external Cassandra and Elasticsearch clusters instead of deploying them as part of the Helm chart.

## Configuration Overview

### External Dependencies
- **Cassandra**: External cluster for primary data storage
- **Elasticsearch**: External cluster for advanced visibility features

### Temporal Services Included
- **Frontend Service**: Handles gRPC API requests
- **History Service**: Manages workflow execution history
- **Matching Service**: Handles task queue operations
- **Worker Service**: Executes internal workflows
- **Admin Tools**: Command-line tools for administration
- **Web UI**: Browser-based interface for monitoring workflows

## Key Configuration Details

### Cassandra Configuration
```yaml
persistence:
  default:
    driver: "cassandra"
    cassandra:
      hosts: ["cassandra-node1.example.com", "cassandra-node2.example.com"]
      port: 9042
      keyspace: "temporal"
      user: "temporal_user"
      password: "temporal_pass"
```

### Elasticsearch Configuration
```yaml
elasticsearch:
  enabled: false
  external: true
  host: "elasticsearch.external.com"
  scheme: "https"
  port: 9200
  version: "v7"
  username: "elastic"
  password: "elastic123"
```

### Node Scheduling
All Temporal services are configured with:
- **Node Selector**: `workload: temporal-services`
- **Tolerations**: Tolerates nodes with `workload=temporal-services` taint

### Important Settings
- **History Shards**: Set to 4096 (as requested)
- **Schema Migration**: Enabled for database setup
- **Internal Dependencies**: All disabled (Cassandra, MySQL, PostgreSQL, Prometheus, Grafana)

## Prerequisites

### Cassandra Requirements
1. External Cassandra cluster must be accessible from Kubernetes cluster
2. Required keyspaces:
   - `temporal` (for main data)
   - `temporal_visibility` (for visibility data)
3. User `temporal_user` must have read/write access to both keyspaces

### Elasticsearch Requirements
1. External Elasticsearch cluster must be accessible from Kubernetes cluster
2. Version 7.x supported
3. User `elastic` must have index creation and read/write permissions
4. Index `temporal_visibility_v1_dev` will be created automatically

### Kubernetes Requirements
1. Nodes labeled with `workload: temporal-services`
2. Nodes tainted with `workload=temporal-services:NoExecute` (optional)

## Deployment Commands

### Install Temporal
```bash
helm install temporal ./charts/temporal -f values-external-cassandra-elasticsearch.yaml
```

### Upgrade Temporal
```bash
helm upgrade temporal ./charts/temporal -f values-external-cassandra-elasticsearch.yaml
```

### Check Status
```bash
kubectl get pods -l app.kubernetes.io/name=temporal
```

## Post-Deployment Verification

### 1. Check Schema Migration Jobs
```bash
kubectl get jobs -l app.kubernetes.io/name=temporal
```

### 2. Verify Service Connectivity
```bash
# Check frontend service
kubectl port-forward svc/temporal-frontend 7233:7233

# Check web UI
kubectl port-forward svc/temporal-web 8080:8080
```

### 3. Test with Admin Tools
```bash
kubectl exec -it deployment/temporal-admintools -- tctl cluster health
```

## Troubleshooting

### Common Issues

1. **Schema Migration Fails**
   - Verify Cassandra connectivity
   - Check user permissions
   - Ensure keyspaces exist or can be created

2. **Elasticsearch Connection Issues**
   - Verify host and port accessibility
   - Check authentication credentials
   - Ensure HTTPS/HTTP scheme matches cluster configuration

3. **Pod Scheduling Issues**
   - Verify node labels: `kubectl get nodes --show-labels`
   - Check node taints: `kubectl describe nodes`

### Logs
```bash
# Server logs
kubectl logs deployment/temporal-frontend
kubectl logs deployment/temporal-history
kubectl logs deployment/temporal-matching
kubectl logs deployment/temporal-worker

# Schema migration logs
kubectl logs job/temporal-schema-setup
kubectl logs job/temporal-schema-update
```

## Security Considerations

1. **Passwords in Plain Text**: This configuration includes passwords directly in the values file. For production, consider:
   - Using Kubernetes secrets
   - External secret management systems
   - Vault integration

2. **Network Security**: Ensure proper network policies and firewall rules between:
   - Kubernetes cluster and Cassandra
   - Kubernetes cluster and Elasticsearch

3. **TLS Configuration**: Consider enabling TLS for:
   - Cassandra connections
   - Elasticsearch connections
   - Inter-service communication

## Customization Options

### Resource Limits
Add resource limits to any service:
```yaml
server:
  frontend:
    resources:
      limits:
        cpu: 1000m
        memory: 1Gi
      requests:
        cpu: 500m
        memory: 512Mi
```

### Ingress Configuration
Enable ingress for external access:
```yaml
server:
  frontend:
    ingress:
      enabled: true
      hosts:
        - temporal-api.example.com

web:
  ingress:
    enabled: true
    hosts:
      - temporal-ui.example.com
```

### High Availability
For production deployments:
```yaml
server:
  replicaCount: 3
  frontend:
    podDisruptionBudget:
      minAvailable: 2
```

## Monitoring Integration

To enable monitoring, set:
```yaml
prometheus:
  enabled: true

grafana:
  enabled: true
```

This will deploy Prometheus and Grafana with pre-configured Temporal dashboards.