---
title: "Cluster Management"
type: "docs"
weight: 1
description: "Managing your Kubernetes cluster - pods, deployments, monitoring and troubleshooting"
---

## Kubernetes Cluster Management

Once you've deployed your Kubernetes cluster using the scripts from <a href="https://github.com/raphideb/kube" target="_blank">github.com/raphideb/kube</a>, you have a fully usable platform for running containerized workloads.

The cluster setup includes:

- **Docker with cri-dockerd** - Container runtime
- **Kubernetes cluster** - Orchestration platform
- **Calico networking** - Network policies and security
- **Local-path-provisioner** - Persistent storage
- **Helm package manager** - Application deployment

## Managing Pods

### View Running Pods

List all pods across namespaces:

```bash
kubectl get pods -A
```

List pods in a specific namespace:

```bash
kubectl get pods -n postgres
kubectl get pods -n mongodb
kubectl get pods -n oracle
```

Get detailed pod information:

```bash
kubectl describe pod <pod-name> -n <namespace>
```

### Execute Commands in Pods

Start an interactive shell in a pod:

```bash
kubectl exec -it <pod-name> -n <namespace> -- bash
```

Run a single command:

```bash
kubectl exec <pod-name> -n <namespace> -- ls -la /var/lib
```

For PostgreSQL with cnpg plugin:

```bash
kubectl cnpg psql pg-cluster -n postgres
```

### View Pod Logs

View current logs:

```bash
kubectl logs <pod-name> -n <namespace>
```

Follow logs in real-time:

```bash
kubectl logs -f <pod-name> -n <namespace>
```

View logs from previous container (after crash):

```bash
kubectl logs <pod-name> -n <namespace> --previous
```

View last 100 lines:

```bash
kubectl logs <pod-name> -n <namespace> --tail=100
```

### Delete Pods

Delete a specific pod (will be recreated by deployment/statefulset):

```bash
kubectl delete pod <pod-name> -n <namespace>
```

Force delete a stuck pod:

```bash
kubectl delete pod <pod-name> -n <namespace> --grace-period=0 --force
```

## Managing Deployments and StatefulSets

### View Deployments

```bash
kubectl get deployments -A
kubectl get deployments -n <namespace>
```

View detailed deployment info:

```bash
kubectl describe deployment <deployment-name> -n <namespace>
```

### View StatefulSets

Database operators typically use StatefulSets:

```bash
kubectl get statefulsets -A
kubectl get statefulsets -n postgres
```

### Scale Deployments

Scale a deployment up or down:

```bash
kubectl scale deployment <deployment-name> -n <namespace> --replicas=3
```

For CloudNativePG PostgreSQL clusters:

```bash
kubectl patch cluster pg-cluster -n postgres --type merge \
  -p '{"spec":{"instances":3}}'
```

### Update Deployments

Set a new image version:

```bash
kubectl set image deployment/<deployment-name> \
  <container-name>=<new-image>:<tag> -n <namespace>
```

View rollout status:

```bash
kubectl rollout status deployment/<deployment-name> -n <namespace>
```

Rollback to previous version:

```bash
kubectl rollout undo deployment/<deployment-name> -n <namespace>
```

## Monitoring with kubectl

### View Cluster Nodes

```bash
kubectl get nodes
kubectl get nodes -o wide
```

Detailed node information:

```bash
kubectl describe node <node-name>
```

### View Resource Usage

Check node resource usage:

```bash
kubectl top nodes
```

Check pod resource usage:

```bash
kubectl top pods -A
kubectl top pods -n postgres
```

### View Services

List all services:

```bash
kubectl get services -A
kubectl get svc -n postgres
```

Detailed service information:

```bash
kubectl describe svc <service-name> -n <namespace>
```

### View Persistent Volumes

```bash
kubectl get pv
kubectl get pvc -A
kubectl get pvc -n postgres
```

Check which pods are using volumes:

```bash
kubectl describe pvc <pvc-name> -n <namespace>
```

## Monitoring with Prometheus and Grafana

The monitoring stack provides comprehensive cluster observability.

### Access Monitoring UIs

- **Grafana:** http://\<your-host-ip\>:30000 (user/pass: admin)
- **Prometheus:** http://\<your-host-ip\>:30090

### View Metrics in Prometheus

Query pod CPU usage:

```promql
rate(container_cpu_usage_seconds_total{namespace="postgres"}[5m])
```

Query pod memory usage:

```promql
container_memory_usage_bytes{namespace="postgres"}
```

Query pod network traffic:

```promql
rate(container_network_receive_bytes_total[5m])
```

### PodMonitor and ServiceMonitor

List existing monitors:

```bash
kubectl get podmonitor -A
kubectl get servicemonitor -A
```

View monitoring configuration:

```bash
kubectl describe servicemonitor -n monitoring
```

Create a custom PodMonitor:

```yaml
apiVersion: monitoring.coreos.com/v1
kind: PodMonitor
metadata:
  name: myapp-monitor
  namespace: default
spec:
  selector:
    matchLabels:
      app: myapp
  podMetricsEndpoints:
  - port: metrics
    interval: 30s
```

Apply it:

```bash
kubectl apply -f podmonitor.yaml
```

## Events and Troubleshooting

### View Cluster Events

Watch all events in real-time:

```bash
kubectl get events -A --watch
```

View events for a specific namespace:

```bash
kubectl get events -n postgres --sort-by='.lastTimestamp'
```

Filter events by type:

```bash
kubectl get events -A --field-selector type=Warning
```

### Debug Pod Issues

Check why a pod isn't starting:

```bash
kubectl describe pod <pod-name> -n <namespace>
kubectl logs <pod-name> -n <namespace>
```

Check pod status:

```bash
kubectl get pod <pod-name> -n <namespace> -o yaml
```

Common issues to look for:

- Image pull errors
- Resource limits (OOMKilled)
- Volume mount failures
- Configuration errors

### Port Forwarding

Forward a pod port to localhost:

```bash
kubectl port-forward pod/<pod-name> -n <namespace> 5432:5432
```

Forward a service port:

```bash
kubectl port-forward -n postgres svc/pg-cluster-rw 5432:5432
```

Run in background:

```bash
kubectl port-forward -n postgres svc/pg-cluster-rw 5432:5432 &
```

## Namespace Management

### Create Namespaces

```bash
kubectl create namespace development
kubectl create namespace production
```

With labels:

```bash
kubectl create namespace dev
kubectl label namespace dev environment=development
```

### View Namespaces

```bash
kubectl get namespaces
kubectl get ns
```

### Set Default Namespace

Set default for current context:

```bash
kubectl config set-context --current --namespace=postgres
```

Now `kubectl get pods` will show postgres namespace by default.

## Resource Management

### View Resource Quotas

```bash
kubectl get resourcequota -A
kubectl describe resourcequota -n <namespace>
```

### View Limit Ranges

```bash
kubectl get limitrange -A
```

### Apply Resource Limits

Example deployment with limits:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: myapp
spec:
  template:
    spec:
      containers:
      - name: myapp
        image: myapp:latest
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
```

## Working with ConfigMaps and Secrets

### View Secrets

```bash
kubectl get secrets -n postgres
```

Decode a secret:

```bash
kubectl get secret <secret-name> -n postgres -o jsonpath='{.data.password}' | base64 -d
```

### View ConfigMaps

```bash
kubectl get configmaps -A
kubectl describe configmap <configmap-name> -n <namespace>
```

## Network Policies with Calico

Your cluster includes Calico for network policies. Here are a few useful examples:

### Restrict Database Access

Allow only specific pods to access PostgreSQL:

```yaml
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: postgres-access
  namespace: postgres
spec:
  podSelector:
    matchLabels:
      cnpg.io/cluster: pg-cluster
  policyTypes:
  - Ingress
  ingress:
  - from:
    - podSelector:
        matchLabels:
          app: myapp
    ports:
    - protocol: TCP
      port: 5432
```

Apply it:

```bash
kubectl apply -f postgres-policy.yaml
```

### Enable WireGuard Encryption

Encrypt all pod-to-pod traffic:

```bash
kubectl patch felixconfiguration default --type merge \
  --patch '{"spec":{"wireguardEnabled":true}}'
```

Verify:

```bash
kubectl get felixconfiguration default -o yaml | grep wireguard
```

### View Network Policies

```bash
kubectl get networkpolicies -A
kubectl describe networkpolicy <policy-name> -n <namespace>
```

For more Calico examples, see CALICO_USAGE.md in the <a href="https://github.com/raphideb/kube" target="_blank">script repository</a>.

## Cluster Cleanup

### Delete Resources

Delete by file:

```bash
kubectl delete -f deployment.yaml
```

Delete by type and name:

```bash
kubectl delete deployment <deployment-name> -n <namespace>
kubectl delete service <service-name> -n <namespace>
```

Delete all pods in namespace (dangerous):

```bash
kubectl delete pods --all -n <namespace>
```

### Delete Entire Namespace

This deletes everything in the namespace:

```bash
kubectl delete namespace <namespace>
```

## Useful Commands

Quick reference for common operations:

```bash
# Get everything in a namespace
kubectl get all -n <namespace>

# Watch resources update in real-time
kubectl get pods -n postgres -w

# Show labels for all pods
kubectl get pods -n postgres --show-labels

# Filter by label
kubectl get pods -l app=myapp -n default

# Export resource as YAML
kubectl get deployment <name> -n <namespace> -o yaml > deployment.yaml

# Dry-run to preview changes
kubectl apply -f deployment.yaml --dry-run=client

# Force refresh/restart deployment
kubectl rollout restart deployment/<deployment-name> -n <namespace>
```

