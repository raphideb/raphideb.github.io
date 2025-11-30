---
title: "MongoDB on Kubernetes"
type: "docs"
weight: 3
description: "MongoDB deployment with Community Operator"
---

## MongoDB Community Operator

After deploying MongoDB using the scripts from <a href="https://github.com/raphideb/kube" target="_blank">github.com/raphideb/kube</a>, you have a fully usable MongoDB replica set managed by the MongoDB Community Operator.

The deployment includes:

- **MongoDB Community Operator** - Kubernetes-native MongoDB management
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

## Getting Started

For installation instructions and deployment details, see the README files in the <a href="https://github.com/raphideb/kube" target="_blank">script repository</a>.

