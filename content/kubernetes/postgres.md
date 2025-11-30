---
title: "PostgreSQL on Kubernetes"
type: "docs"
weight: 2
description: "PostgreSQL deployment with CloudNativePG operator"
---

## CloudNativePG Operator

After deploying PostgreSQL using the scripts from <a href="https://github.com/raphideb/kube" target="_blank">github.com/raphideb/kube</a>, you have a fully usable PostgreSQL cluster managed by the CloudNativePG operator.

The deployment includes:

- **CloudNativePG operator** - Kubernetes-native PostgreSQL management
- **cnpg plugin** - Easy database access via kubectl
- **PostgreSQL cluster** - Multi-instance cluster with replication
- **Grafana integration** - Automatic monitoring setup

## Connecting to PostgreSQL

### Using cnpg Plugin (Recommended)

Connect directly with psql:

```bash
kubectl cnpg psql pg-cluster -n postgres
```

### Using Port Forwarding

Forward the PostgreSQL port to your local machine:

```bash
kubectl port-forward -n postgres svc/pg-cluster-rw 5432:5432
```

Then connect from another terminal:

```bash
psql -h localhost -p 5432 -U postgres postgres
```

### Direct Pod Access

Get the pod name and connect:

```bash
kubectl get pods -n postgres
kubectl exec -it pg-cluster-1 -n postgres -- bash
psql -U postgres
```

## High Availability and Failover

### View Cluster Status

Check the cluster and its instances:

```bash
kubectl get cluster -n postgres
kubectl get pods -n postgres
```

Output shows which pod is primary and which are replicas:

```
NAME           AGE   INSTANCES   READY   STATUS
pg-cluster     5m    3           3       Cluster in healthy state

NAME             READY   STATUS    ROLE
pg-cluster-1     1/1     Running   Primary
pg-cluster-2     1/1     Running   Replica
pg-cluster-3     1/1     Running   Replica
```

### Test Automatic Failover

Delete the primary pod and watch the operator promote a replica:

```bash
# Watch cluster status in one terminal
kubectl get cluster -n postgres -w

# In another terminal, delete the primary
kubectl delete pod pg-cluster-1 -n postgres
```

The operator automatically promotes a replica to primary within seconds. A new replica pod is created to maintain the desired instance count.

### Check Replication Status

Connect and check replication:

```bash
kubectl cnpg psql pg-cluster -n postgres

SELECT client_addr, state, sync_state
FROM pg_stat_replication;
```

## Scaling the Cluster

### Scale to More Instances

Increase from 3 to 5 instances:

```bash
kubectl patch cluster pg-cluster -n postgres --type merge \
  -p '{"spec":{"instances":5}}'
```

Watch new pods being created:

```bash
kubectl get pods -n postgres -w
```

### Scale Down

Reduce to 2 instances:

```bash
kubectl patch cluster pg-cluster -n postgres --type merge \
  -p '{"spec":{"instances":2}}'
```

The operator removes replicas while preserving the primary.

## Performance Testing with pgbench

### Initialize pgbench Tables

Connect to the database and create pgbench schema:

```bash
kubectl cnpg psql pg-cluster -n postgres

CREATE DATABASE pgbench;
\q
```

Get the pod name and initialize:

```bash
POD=$(kubectl get pod -n postgres -l role=primary -o jsonpath='{.items[0].metadata.name}')
kubectl exec -it $POD -n postgres -- pgbench -i -s 100 -d pgbench -U postgres
```

### Run TPC-B Benchmark

Run a 60-second benchmark:

```bash
kubectl exec -it $POD -n postgres -- \
  pgbench -c 20 -j 20 -T 60 -d pgbench -U postgres
```

Output shows transactions per second (TPS):

```
transaction type: <builtin: TPC-B (sort of)>
scaling factor: 100
number of clients: 20
duration: 60 s
number of transactions: 318422
tps = 5307.477304
```

### Monitor in Grafana

While pgbench runs, open Grafana:

http://\<your-host-ip\>:30000

Import Dashboard ID: **20417** for CloudNativePG

Watch real-time metrics:

- Transaction rate
- Active connections
- Cache hit ratio
- Replication lag

## Configuration Changes

### View Current Configuration

```bash
kubectl get cluster pg-cluster -n postgres -o yaml
```

### Modify PostgreSQL Parameters

Edit the cluster to add custom parameters:

```bash
kubectl edit cluster pg-cluster -n postgres
```

Add under `spec.postgresql`:

```yaml
spec:
  postgresql:
    parameters:
      max_connections: "200"
      shared_buffers: "256MB"
      work_mem: "8MB"
      maintenance_work_mem: "128MB"
```

Save and exit. The operator applies changes with a rolling update.

Watch the update:

```bash
kubectl get pods -n postgres -w
```

### Verify Configuration

Connect and check:

```bash
kubectl cnpg psql pg-cluster -n postgres

SHOW max_connections;
SHOW shared_buffers;
```

## Backup and Recovery

### View Backup Configuration

Check if backups are configured:

```bash
kubectl get cluster pg-cluster -n postgres -o yaml | grep -A 10 backup
```

### List Backups

```bash
kubectl get backup -n postgres
```

### Trigger Manual Backup

Create an on-demand backup:

```yaml
apiVersion: postgresql.cnpg.io/v1
kind: Backup
metadata:
  name: pg-cluster-backup-manual
  namespace: postgres
spec:
  cluster:
    name: pg-cluster
```

Apply it:

```bash
kubectl apply -f backup.yaml
```

Monitor backup progress:

```bash
kubectl get backup -n postgres -w
kubectl describe backup pg-cluster-backup-manual -n postgres
```

## Monitoring and Troubleshooting

### Check Cluster Events

View operator events:

```bash
kubectl get events -n postgres --sort-by='.lastTimestamp'
```

### View Pod Logs

Check PostgreSQL logs:

```bash
kubectl logs -f pg-cluster-1 -n postgres
```

View previous container logs (after crash):

```bash
kubectl logs pg-cluster-1 -n postgres --previous
```

### Resource Usage

Check CPU and memory usage:

```bash
kubectl top pods -n postgres
```

### Connection Pooling

The cluster includes PgBouncer for connection pooling. Check pooler pods:

```bash
kubectl get pods -n postgres -l cnpg.io/poolerName
```

Connect through pooler:

```bash
kubectl port-forward -n postgres svc/pg-cluster-pooler-rw 5432:5432
psql -h localhost -p 5432 -U postgres postgres
```

## Database Administration

### Create Additional Databases

```bash
kubectl cnpg psql pg-cluster -n postgres

CREATE DATABASE myapp;
CREATE USER myapp_user WITH PASSWORD 'secure_password';
GRANT ALL PRIVILEGES ON DATABASE myapp TO myapp_user;
\q
```

### Run SQL Scripts

Create a script locally:

```sql
-- schema.sql
CREATE TABLE users (
  id SERIAL PRIMARY KEY,
  username VARCHAR(50) UNIQUE NOT NULL,
  email VARCHAR(100) UNIQUE NOT NULL,
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

CREATE INDEX idx_users_email ON users(email);
```

Copy to pod and execute:

```bash
POD=$(kubectl get pod -n postgres -l role=primary -o jsonpath='{.items[0].metadata.name}')
kubectl cp schema.sql postgres/$POD:/tmp/schema.sql
kubectl exec -it $POD -n postgres -- psql -U postgres -d myapp -f /tmp/schema.sql
```

### Check Database Size

```bash
kubectl cnpg psql pg-cluster -n postgres

SELECT pg_database.datname,
       pg_size_pretty(pg_database_size(pg_database.datname)) AS size
FROM pg_database
ORDER BY pg_database_size(pg_database.datname) DESC;
```

## Deploy Additional Clusters

The repository includes a template YAML file to deploy more PostgreSQL clusters.

View the template:

```bash
cat pg-cluster.yml
```

Deploy a new cluster:

```bash
kubectl apply -f pg-cluster.yml
```

This creates a separate PostgreSQL cluster with its own storage and resources.

## Cleanup

### Delete a Single Cluster

```bash
kubectl delete cluster pg-cluster -n postgres
```

This deletes pods and services but preserves PVCs (data).

### Delete Everything Including Data

```bash
kubectl delete cluster pg-cluster -n postgres
kubectl delete pvc -n postgres --all
```

## Useful Queries for Monitoring

Connect to the database and run these queries:

### Active Connections

```sql
SELECT count(*) FROM pg_stat_activity
WHERE state = 'active';
```

### Long Running Queries

```sql
SELECT pid, now() - pg_stat_activity.query_start AS duration, query
FROM pg_stat_activity
WHERE state = 'active'
  AND now() - pg_stat_activity.query_start > interval '5 minutes'
ORDER BY duration DESC;
```

### Database Cache Hit Ratio

```sql
SELECT sum(heap_blks_read) as heap_read,
       sum(heap_blks_hit) as heap_hit,
       sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) * 100 as cache_hit_ratio
FROM pg_statio_user_tables;
```

### Table Sizes

```sql
SELECT schemaname, tablename,
       pg_size_pretty(pg_total_relation_size(schemaname||'.'||tablename)) AS size
FROM pg_tables
WHERE schemaname NOT IN ('pg_catalog', 'information_schema')
ORDER BY pg_total_relation_size(schemaname||'.'||tablename) DESC
LIMIT 10;
```

## Getting Started

For installation instructions and deployment details, see the README files in the <a href="https://github.com/raphideb/kube" target="_blank">script repository</a>.

