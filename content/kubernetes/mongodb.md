---
title: "MongoDB on Kubernetes"
type: "docs"
weight: 3
description: "MongoDB deployment with Community and Percona Operators"
---

## MongoDB Community Operator

After deploying MongoDB using the scripts from <a href="https://github.com/raphideb/kube" target="_blank">github.com/raphideb/kube</a>, you have a fully usable MongoDB replica set managed by the MongoDB Community- or Percona Operator.

The deployment includes:

- **MongoDB Community- or Percona Operator** - Kubernetes-native MongoDB management
- **Replica Set** - Multi-node MongoDB cluster
- **Sample database** - Pre-configured MongoDB deployment
- **Grafana integration** - Custom monitoring dashboard with Percona exporter

## Connecting to MongoDB

### Using Port Forwarding

Forward MongoDB port to your local machine:

```bash
kubectl port-forward -n mongodb svc/mongodb-svc 27017:27017
```

Connect with mongosh from another terminal:

```bash
mongosh -u admin mongodb://localhost:27017/admin
```

The default password is in the mongodb.yml file.

### Direct Pod Access

Get the pod name and connect:

```bash
kubectl get pods -n mongodb
kubectl exec -it mongodb-0 -n mongodb -- mongosh
```

Authenticate after connecting:

```javascript
use admin
db.auth("admin", "password")
```

## Replica Set Operations

### View Replica Set Status

Connect to MongoDB and check replica set:

```bash
kubectl exec -it mongodb-0 -n mongodb -- mongosh

use admin
db.auth("admin", "password")
rs.status()
```

Output shows which member is PRIMARY and which are SECONDARY:

```javascript
{
  members: [
    { _id: 0, name: 'mongodb-0:27017', health: 1, state: 1, stateStr: 'PRIMARY' },
    { _id: 1, name: 'mongodb-1:27017', health: 1, state: 2, stateStr: 'SECONDARY' },
    { _id: 2, name: 'mongodb-2:27017', health: 1, state: 2, stateStr: 'SECONDARY' }
  ]
}
```

### Test Automatic Failover

Delete the primary pod and watch the replica set elect a new primary:

```bash
# Watch pods in one terminal
kubectl get pods -n mongodb -w

# In another terminal, delete the primary
kubectl delete pod mongodb-0 -n mongodb
```

Connect and check the new primary:

```bash
kubectl exec -it mongodb-1 -n mongodb -- mongosh

use admin
db.auth("admin", "password")
rs.status()
```

A new primary is elected within seconds, and mongodb-0 rejoins as a secondary.

## Scaling the Replica Set

### Scale to More Members

Increase from 3 to 5 members:

```bash
kubectl patch mongodbcommunity mongodb-cluster -n mongodb --type merge \
  -p '{"spec":{"members":5}}'
```

Watch new pods being created:

```bash
kubectl get pods -n mongodb -w
```

New members automatically sync data and join the replica set.

### Scale Down

Reduce to 2 members:

```bash
kubectl patch mongodbcommunity mongodb-cluster -n mongodb --type merge \
  -p '{"spec":{"members":2}}'
```

## Database Operations

### Create Database and Collection

```bash
kubectl exec -it mongodb-0 -n mongodb -- mongosh

use admin
db.auth("admin", "password")

use myapp
db.createCollection("users")
db.users.insertOne({
  username: "alice",
  email: "alice@example.com",
  created: new Date()
})

db.users.find()
```

### Create User with Permissions

```javascript
use admin
db.auth("admin", "password")

db.createUser({
  user: "myapp_user",
  pwd: "secure_password",
  roles: [
    { role: "readWrite", db: "myapp" }
  ]
})
```

Test the new user:

```bash
mongosh -u myapp_user -p secure_password mongodb://localhost:27017/myapp
```

### Import Data

Create a JSON file locally:

```json
[
  {"name": "Alice", "age": 30, "city": "New York"},
  {"name": "Bob", "age": 25, "city": "San Francisco"},
  {"name": "Charlie", "age": 35, "city": "Seattle"}
]
```

Copy to pod and import:

```bash
kubectl cp users.json mongodb/mongodb-0:/tmp/users.json

kubectl exec -it mongodb-0 -n mongodb -- mongoimport \
  --db myapp \
  --collection users \
  --file /tmp/users.json \
  --jsonArray \
  --username admin \
  --password password \
  --authenticationDatabase admin
```

Verify:

```bash
kubectl exec -it mongodb-0 -n mongodb -- mongosh

use admin
db.auth("admin", "password")
use myapp
db.users.find()
```

## Performance Testing

### Insert Test Data

Create a simple benchmark script:

```javascript
use testdb

for (let i = 0; i < 100000; i++) {
  db.testcollection.insertOne({
    index: i,
    data: "test data " + i,
    timestamp: new Date(),
    random: Math.random()
  })
}
```

Run it:

```bash
kubectl exec -it mongodb-0 -n mongodb -- mongosh

use admin
db.auth("admin", "password")
// paste the script above
```

### Query Performance

Test query performance:

```javascript
use testdb

// Create index
db.testcollection.createIndex({ index: 1 })

// Query with explain
db.testcollection.find({ index: { $gt: 50000 } }).explain("executionStats")
```

### Monitor in Grafana

While tests run, open Grafana:

http://\<your-host-ip\>:30000

Import the custom dashboard from the repository:

```
MongoDB_Percona_Grafana.json
```

Watch real-time metrics:

- Operations per second
- Connection counts
- Replication lag
- Memory usage
- Cache statistics

## Monitoring and Troubleshooting

### Check Cluster Status

```bash
kubectl get mongodbcommunity -n mongodb
kubectl get pods -n mongodb
```

### View Events

```bash
kubectl get events -n mongodb --sort-by='.lastTimestamp'
```

### View Pod Logs

Check MongoDB logs:

```bash
kubectl logs -f mongodb-0 -n mongodb
```

Check operator logs:

```bash
kubectl logs -n mongodb -l app.kubernetes.io/name=mongodb-kubernetes-operator
```

### Resource Usage

```bash
kubectl top pods -n mongodb
```

### Check Replication Lag

```bash
kubectl exec -it mongodb-1 -n mongodb -- mongosh

use admin
db.auth("admin", "password")
rs.printSlaveReplicationInfo()
```

## Backup and Recovery

### Create Backup with mongodump

Backup the entire database:

```bash
kubectl exec mongodb-0 -n mongodb -- mongodump \
  --out=/tmp/backup \
  --username=admin \
  --password=password \
  --authenticationDatabase=admin
```

Copy backup from pod:

```bash
kubectl cp mongodb/mongodb-0:/tmp/backup ./mongodb-backup
```

### Restore with mongorestore

Copy backup to pod:

```bash
kubectl cp ./mongodb-backup mongodb/mongodb-0:/tmp/restore
```

Restore:

```bash
kubectl exec mongodb-0 -n mongodb -- mongorestore \
  /tmp/restore \
  --username=admin \
  --password=password \
  --authenticationDatabase=admin
```

### Backup Specific Database

```bash
kubectl exec mongodb-0 -n mongodb -- mongodump \
  --db=myapp \
  --out=/tmp/backup-myapp \
  --username=admin \
  --password=password \
  --authenticationDatabase=admin
```

## Useful MongoDB Commands

Connect and run these administrative commands:

### Database Statistics

```javascript
use myapp
db.stats()
```

### Collection Statistics

```javascript
use myapp
db.users.stats()
```

### Index Information

```javascript
use myapp
db.users.getIndexes()
```

### Current Operations

```javascript
use admin
db.currentOp()
```

### Server Status

```javascript
use admin
db.serverStatus()
```

### Check Connection Count

```javascript
use admin
db.serverStatus().connections
```

## Deploy Additional Clusters

The repository includes a template YAML file for deploying more MongoDB clusters.

View the template:

```bash
cat mongodb.yml
```

Deploy a new cluster:

```bash
kubectl apply -f mongodb.yml
```

This creates a separate MongoDB replica set with its own storage and resources.

## Cleanup

### Delete Cluster

```bash
kubectl delete mongodbcommunity mongodb-cluster -n mongodb
```

This deletes pods and services but preserves PVCs (data).

### Delete Everything Including Data

```bash
kubectl delete mongodbcommunity mongodb-cluster -n mongodb
kubectl delete pvc -n mongodb --all
```

## Working with Aggregations

### Create Sample Data

```javascript
use sales

db.orders.insertMany([
  { product: "laptop", quantity: 2, price: 1200, date: new Date("2024-01-15") },
  { product: "mouse", quantity: 5, price: 25, date: new Date("2024-01-16") },
  { product: "keyboard", quantity: 3, price: 75, date: new Date("2024-01-17") },
  { product: "laptop", quantity: 1, price: 1200, date: new Date("2024-01-18") },
  { product: "monitor", quantity: 2, price: 300, date: new Date("2024-01-19") }
])
```

### Run Aggregation Pipeline

Total sales by product:

```javascript
db.orders.aggregate([
  {
    $group: {
      _id: "$product",
      totalQuantity: { $sum: "$quantity" },
      totalRevenue: { $sum: { $multiply: ["$quantity", "$price"] } }
    }
  },
  { $sort: { totalRevenue: -1 } }
])
```

## Network Policies

Restrict access to MongoDB using Calico network policies:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: mongodb-access
  namespace: mongodb
spec:
  podSelector:
    matchLabels:
      app: mongodb-svc
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: myapp
    ports:
    - protocol: TCP
      port: 27017
```

Apply it:

```bash
kubectl apply -f mongodb-policy.yaml
```

Now only pods labeled with `app: myapp` can connect to MongoDB.

## Percona MongoDB Operator

In addition to the Community Operator, the repository also supports deploying MongoDB using the Percona Operator for MongoDB (`psmdb.percona.com/v1`). This runs Percona Server for MongoDB 8.0 and uses separate namespaces for the operator and database.

The deployment includes:

- **Percona Operator** - Kubernetes-native MongoDB management with advanced features
- **Percona Server for MongoDB 8.0** - Production-ready MongoDB distribution
- **Replica Set** - Multi-node cluster (rs0)
- **Built-in backup** - Integrated backup with percona-backup-mongodb
- **Grafana integration** - Percona exporter for monitoring

### Connecting to Percona MongoDB

#### Direct Pod Access

Get the pod name and connect:

```bash
kubectl get pods -n percona-mongodb
kubectl exec -it percona-mongodb-rs0-0 -n percona-mongodb -- mongosh
```

Authenticate after connecting:

```javascript
use admin
db.auth("clusterAdmin", "password")
```

#### Using Port Forwarding

Forward MongoDB port to your local machine:

```bash
kubectl port-forward -n percona-mongodb svc/percona-mongodb-rs0 27017:27017
```

Connect with mongosh from another terminal:

```bash
mongosh -u clusterAdmin mongodb://localhost:27017/admin
```

The default password is in the percona_mongodb.yml file.

### Replica Set Status

Check the Percona MongoDB cluster status:

```bash
kubectl get psmdb -n percona-mongodb
```

Output shows the cluster state:

```
NAME                STATUS   AGE
percona-mongodb     ready    5m
```

Connect and check replica set members:

```bash
kubectl exec -it percona-mongodb-rs0-0 -n percona-mongodb -- mongosh

use admin
db.auth("clusterAdmin", "password")
rs.status()
```

Output shows which member is PRIMARY and which are SECONDARY:

```javascript
{
  members: [
    { _id: 0, name: 'percona-mongodb-rs0-0:27017', health: 1, state: 1, stateStr: 'PRIMARY' },
    { _id: 1, name: 'percona-mongodb-rs0-1:27017', health: 1, state: 2, stateStr: 'SECONDARY' },
    { _id: 2, name: 'percona-mongodb-rs0-2:27017', health: 1, state: 2, stateStr: 'SECONDARY' }
  ]
}
```

### Test Automatic Failover

Delete the primary pod and watch the replica set elect a new primary:

```bash
# Watch pods in one terminal
kubectl get pods -n percona-mongodb -w

# In another terminal, delete the primary
kubectl delete pod percona-mongodb-rs0-0 -n percona-mongodb
```

Connect and check the new primary:

```bash
kubectl exec -it percona-mongodb-rs0-1 -n percona-mongodb -- mongosh

use admin
db.auth("clusterAdmin", "password")
rs.status()
```

A new primary is elected within seconds, and the deleted pod rejoins as a secondary.

### Scaling the Replica Set

#### Scale to More Members

Increase from 3 to 5 members:

```bash
kubectl patch psmdb percona-mongodb -n percona-mongodb --type merge \
  -p '{"spec":{"replsets":[{"name":"rs0","size":5}]}}'
```

Watch new pods being created:

```bash
kubectl get pods -n percona-mongodb -w
```

New members automatically sync data and join the replica set.

#### Scale Down

Reduce to 2 members:

```bash
kubectl patch psmdb percona-mongodb -n percona-mongodb --type merge \
  -p '{"spec":{"replsets":[{"name":"rs0","size":2}]}}'
```

### Database Operations

#### Create Database and Collection

```bash
kubectl exec -it percona-mongodb-rs0-0 -n percona-mongodb -- mongosh

use admin
db.auth("clusterAdmin", "password")

use myapp
db.createCollection("users")
db.users.insertOne({
  username: "alice",
  email: "alice@example.com",
  created: new Date()
})

db.users.find()
```

#### Import Data

Create a JSON file locally:

```json
[
  {"name": "Alice", "age": 30, "city": "New York"},
  {"name": "Bob", "age": 25, "city": "San Francisco"},
  {"name": "Charlie", "age": 35, "city": "Seattle"}
]
```

Copy to pod and import:

```bash
kubectl cp users.json percona-mongodb/percona-mongodb-rs0-0:/tmp/users.json

kubectl exec -it percona-mongodb-rs0-0 -n percona-mongodb -- mongoimport \
  --db myapp \
  --collection users \
  --file /tmp/users.json \
  --jsonArray \
  --username userAdmin \
  --password password \
  --authenticationDatabase admin
```

### User Management

Percona MongoDB includes built-in system users with predefined roles:

- **userAdmin** - Database user administration
- **clusterAdmin** - Cluster-wide administration
- **clusterMonitor** - Read-only cluster monitoring
- **backup** - Backup operations

Create an application user:

```javascript
use admin
db.auth("userAdmin", "password")

db.createUser({
  user: "myapp_user",
  pwd: "secure_password",
  roles: [
    { role: "readWrite", db: "myapp" }
  ]
})
```

Test the new user:

```bash
mongosh -u myapp_user -p secure_password mongodb://localhost:27017/myapp
```

### Backup Configuration

Percona MongoDB Operator includes built-in backup support using `percona-backup-mongodb:2.11.0`.

Check backup status:

```bash
kubectl get psmdb percona-mongodb -n percona-mongodb -o yaml | grep -A 20 backup
```

The backup agent runs as a sidecar container in each replica set pod. Configure backup storage and schedules in the `percona_mongodb.yml` template.

### Deploy Additional Clusters

The repository includes a template YAML file for deploying more Percona MongoDB clusters.

View the template:

```bash
cat percona_mongodb.yml
```

Deploy a new cluster:

```bash
kubectl apply -f percona_mongodb.yml
```

This creates a separate Percona MongoDB replica set with its own storage and resources.

### Monitoring

The Percona exporter is automatically configured for Grafana integration.

Open Grafana:

http://\<your-host-ip\>:30000

Import the custom dashboard from the repository:

```
MongoDB_Percona_Grafana.json
```

Watch real-time metrics:

- Operations per second
- Connection counts
- Replication lag
- Memory usage
- Cache statistics

### Cleanup

#### Delete Cluster

```bash
kubectl delete psmdb percona-mongodb -n percona-mongodb
```

This deletes pods and services but preserves PVCs (data).

#### Delete Everything Including Data

```bash
kubectl delete psmdb percona-mongodb -n percona-mongodb
kubectl delete pvc -n percona-mongodb --all
```

#### Delete Operator

```bash
kubectl delete namespace percona-mongodb-operator
```

### Network Policies

Restrict access to Percona MongoDB using Calico network policies:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: percona-mongodb-access
  namespace: percona-mongodb
spec:
  podSelector:
    matchLabels:
      app.kubernetes.io/instance: percona-mongodb
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: myapp
    ports:
    - protocol: TCP
      port: 27017
```

Apply it:

```bash
kubectl apply -f percona-mongodb-policy.yaml
```

Now only pods labeled with `app: myapp` can connect to Percona MongoDB.

## Getting Started

For installation instructions and deployment details, see the README files in the <a href="https://github.com/raphideb/kube" target="_blank">script repository</a>.

