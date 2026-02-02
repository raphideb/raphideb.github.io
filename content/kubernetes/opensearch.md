---
title: "OpenSearch on Kubernetes"
type: "docs"
weight: 5
description: "OpenSearch deployment with Opster Operator"
---

## OpenSearch Operator

After deploying OpenSearch using the scripts from <a href="https://github.com/raphideb/kube" target="_blank">github.com/raphideb/kube</a>, you have a fully usable OpenSearch cluster managed by the Opster OpenSearch Operator (`opensearch.opster.io/v1`).

The deployment includes:

- **OpenSearch Operator** - Kubernetes-native OpenSearch management (Opster)
- **3-node cluster** - Multi-role nodes (cluster_manager, data, ingest)
- **OpenSearch Dashboards** - Web UI for visualization and management
- **Security configuration** - Admin credentials and internal users
- **Grafana integration** - Prometheus metrics via ServiceMonitor

## Connecting to OpenSearch

### Using NodePort (Direct Access)

OpenSearch is exposed via NodePort on port 30200:

```bash
curl -k -u admin:admin https://<your-host-ip>:30200
```

Output shows cluster information:

```json
{
  "name": "opensearch-cluster-nodes-0",
  "cluster_name": "opensearch-cluster",
  "cluster_uuid": "...",
  "version": {
    "distribution": "opensearch",
    "number": "2.11.1"
  }
}
```

### OpenSearch Dashboards

Access the Dashboards UI on NodePort 30601:

```
http://<your-host-ip>:30601
```

Login with admin credentials:

- **Username:** admin
- **Password:** admin

### Using Port Forwarding

Forward OpenSearch port to your local machine:

```bash
kubectl port-forward -n opensearch svc/opensearch-cluster 9200:9200
```

Connect from another terminal:

```bash
curl -k -u admin:admin https://localhost:9200
```

## Cluster Health and Status

### Check Cluster Health

```bash
curl -k -u admin:admin https://<your-host-ip>:30200/_cluster/health?pretty
```

Output shows the cluster state:

```json
{
  "cluster_name": "opensearch-cluster",
  "status": "green",
  "number_of_nodes": 3,
  "number_of_data_nodes": 3,
  "active_primary_shards": 5,
  "active_shards": 10,
  "unassigned_shards": 0
}
```

### View Cluster Nodes

```bash
curl -k -u admin:admin https://<your-host-ip>:30200/_cat/nodes?v
```

### List Indices

```bash
curl -k -u admin:admin https://<your-host-ip>:30200/_cat/indices?v
```

### Cluster Settings

```bash
curl -k -u admin:admin https://<your-host-ip>:30200/_cluster/settings?pretty
```

## Index Operations

### Create an Index

Create an index with custom mappings:

```bash
curl -k -u admin:admin -X PUT https://<your-host-ip>:30200/myapp \
  -H 'Content-Type: application/json' \
  -d '{
    "settings": {
      "number_of_shards": 1,
      "number_of_replicas": 1
    },
    "mappings": {
      "properties": {
        "title": { "type": "text" },
        "description": { "type": "text" },
        "timestamp": { "type": "date" },
        "status": { "type": "keyword" },
        "priority": { "type": "integer" }
      }
    }
  }'
```

### Insert Documents

Insert a single document:

```bash
curl -k -u admin:admin -X POST https://<your-host-ip>:30200/myapp/_doc \
  -H 'Content-Type: application/json' \
  -d '{
    "title": "First document",
    "description": "This is a test document",
    "timestamp": "2024-01-15T10:00:00Z",
    "status": "active",
    "priority": 1
  }'
```

### Bulk Insert

Insert multiple documents at once:

```bash
curl -k -u admin:admin -X POST https://<your-host-ip>:30200/_bulk \
  -H 'Content-Type: application/json' \
  -d '
{"index": {"_index": "myapp"}}
{"title": "Second document", "description": "Another test", "timestamp": "2024-01-16T10:00:00Z", "status": "active", "priority": 2}
{"index": {"_index": "myapp"}}
{"title": "Third document", "description": "Yet another test", "timestamp": "2024-01-17T10:00:00Z", "status": "inactive", "priority": 3}
'
```

### Search Documents

Basic search:

```bash
curl -k -u admin:admin -X GET https://<your-host-ip>:30200/myapp/_search?pretty \
  -H 'Content-Type: application/json' \
  -d '{
    "query": {
      "match": {
        "description": "test"
      }
    }
  }'
```

Filtered search with aggregation:

```bash
curl -k -u admin:admin -X GET https://<your-host-ip>:30200/myapp/_search?pretty \
  -H 'Content-Type: application/json' \
  -d '{
    "query": {
      "bool": {
        "must": [
          { "match": { "status": "active" } }
        ],
        "filter": [
          { "range": { "priority": { "gte": 1, "lte": 5 } } }
        ]
      }
    },
    "aggs": {
      "status_count": {
        "terms": { "field": "status" }
      }
    }
  }'
```

### Delete an Index

```bash
curl -k -u admin:admin -X DELETE https://<your-host-ip>:30200/myapp
```

## Scaling the Cluster

### Scale to More Nodes

Increase from 3 to 5 nodes:

```bash
kubectl patch opensearchcluster opensearch-cluster -n opensearch --type merge \
  -p '{"spec":{"nodePools":[{"component":"nodes","replicas":5}]}}'
```

Watch new pods being created:

```bash
kubectl get pods -n opensearch -w
```

### Scale Down

Reduce to 2 nodes:

```bash
kubectl patch opensearchcluster opensearch-cluster -n opensearch --type merge \
  -p '{"spec":{"nodePools":[{"component":"nodes","replicas":2}]}}'
```

## Dashboard Usage

OpenSearch Dashboards provides a web interface for:

- **Discover** - Explore and search your data
- **Visualize** - Create charts, graphs, and maps
- **Dashboards** - Build and share dashboards
- **Dev Tools** - Run queries directly from the browser
- **Index Management** - Manage indices and policies

Access at:

```
http://<your-host-ip>:30601
```

Use Dev Tools to run queries interactively:

```
GET _cluster/health
GET _cat/indices?v
GET myapp/_search
```

## Monitoring and Troubleshooting

### Check Cluster Status

```bash
kubectl get opensearchcluster -n opensearch
kubectl get pods -n opensearch
```

### View Pod Logs

Check OpenSearch logs:

```bash
kubectl logs -f opensearch-cluster-nodes-0 -n opensearch
```

Check operator logs:

```bash
kubectl logs -n opensearch-operator -l control-plane=controller-manager
```

### View Events

```bash
kubectl get events -n opensearch --sort-by='.lastTimestamp'
```

### Resource Usage

```bash
kubectl top pods -n opensearch
```

### Check Shard Allocation

```bash
curl -k -u admin:admin https://<your-host-ip>:30200/_cat/shards?v
```

### Check Pending Tasks

```bash
curl -k -u admin:admin https://<your-host-ip>:30200/_cluster/pending_tasks?pretty
```

## Security

### Admin Credentials

Default admin credentials are stored in a Kubernetes secret:

```bash
kubectl get secret opensearch-admin-credentials -n opensearch -o yaml
```

### Internal Users

Manage internal users through the security API:

```bash
curl -k -u admin:admin -X GET https://<your-host-ip>:30200/_plugins/_security/api/internalusers?pretty
```

Create a new user:

```bash
curl -k -u admin:admin -X PUT https://<your-host-ip>:30200/_plugins/_security/api/internalusers/myapp_user \
  -H 'Content-Type: application/json' \
  -d '{
    "password": "secure_password",
    "opendistro_security_roles": ["readall"],
    "backend_roles": ["myapp"]
  }'
```

## Deploy Additional Clusters

The repository includes a template YAML file for deploying more OpenSearch clusters.

View the template:

```bash
cat opensearch.yml
```

Deploy a new cluster:

```bash
kubectl apply -f opensearch.yml
```

This creates a separate OpenSearch cluster with its own storage and resources.

## Cleanup

### Delete Cluster

```bash
kubectl delete opensearchcluster opensearch-cluster -n opensearch
```

This deletes pods and services but preserves PVCs (data).

### Delete Everything Including Data

```bash
kubectl delete opensearchcluster opensearch-cluster -n opensearch
kubectl delete pvc -n opensearch --all
```

### Delete Operator

```bash
helm uninstall opensearch-operator -n opensearch-operator
kubectl delete namespace opensearch-operator
```

## Network Policies

Restrict access to OpenSearch using Calico network policies:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: opensearch-access
  namespace: opensearch
spec:
  podSelector:
    matchLabels:
      app: opensearch-cluster
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: myapp
    ports:
    - protocol: TCP
      port: 9200
```

Apply it:

```bash
kubectl apply -f opensearch-policy.yaml
```

Now only pods labeled with `app: myapp` can connect to OpenSearch.

## Getting Started

For installation instructions and deployment details, see the README files in the <a href="https://github.com/raphideb/kube" target="_blank">script repository</a>.
