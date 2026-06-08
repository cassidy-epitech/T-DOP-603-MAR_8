<div align="center">

# 🗳️ Bernstein

</div>

Bernstein is a Kubernetes deployment for a voting application. The project
orchestrates the application services, databases, reverse proxy, and monitoring
tool required by the T-DOP-603 subject.

## 🏗️ Architecture

```text
User
  |
  v
Traefik
  |--------------------|
  v                    v
Poll                 Result
  |                    ^
  v                    |
Redis ---> Worker ---> PostgreSQL
```

Components:

- `poll` — Flask web application receiving votes.
- `redis` — queue storing votes before processing.
- `worker` — Java application moving votes from Redis to PostgreSQL.
- `postgres` — persistent database storing votes in the `votes` table.
- `result` — Node.js web application displaying results.
- `traefik` — reverse proxy and load balancer.
- `cadvisor` — monitoring daemon scheduled on every node.

## 📋 Requirements

- Docker Desktop with Kubernetes enabled
- `kubectl`
- A running Kubernetes context

Check that Kubernetes is ready:

```bash
kubectl get nodes
```

## 🧭 Quick Command Recap

This is the short version to launch and check the project.

### Vérifier Le Cluster

```bash
kubectl get nodes
kubectl get pods -A
kubectl get svc -A
kubectl get ingress -A
```

### Appliquer Le Projet

```bash
kubectl apply --recursive -f bernstein
```

### Créer La Table Votes

Wait until `postgres-0` is running, then create the `votes` table:

```bash
kubectl exec -i postgres-0 -- psql -U admin -d bernsteindb -c 'CREATE TABLE IF NOT EXISTS votes (id text PRIMARY KEY, vote text NOT NULL);'
```

### Ajouter Les Hosts

This lets your browser understand `poll.dop.io` and `result.dop.io` locally:

```bash
echo "127.0.0.1 poll.dop.io result.dop.io" | sudo tee -a /etc/hosts
```

### URLs Navigateur

```text
http://poll.dop.io:30021
http://result.dop.io:30021
http://localhost:30042/dashboard/
```

### Tests Curl

```bash
curl -I http://poll.dop.io:30021
curl -I http://result.dop.io:30021
curl -I http://localhost:30042/dashboard/
```

Expected result: each command should return `200 OK`.

### Vérifier cAdvisor

```bash
kubectl get pods -n kube-system -l app=cadvisor
kubectl get daemonset -n kube-system cadvisor
```

Expected result: cAdvisor should be `1/1 Running`.

### Vérifier PostgreSQL

```bash
kubectl exec postgres-0 -- psql -U admin -d bernsteindb -c '\dt'
```

Expected result: the `votes` table should be listed.

## 🚀 Deploy Step By Step

This section deploys the project one part at a time. Run the commands in this
order.

### 1. Check Kubernetes

Make sure the cluster is available before deploying anything:

```bash
kubectl get nodes
```

You should see a node with the `Ready` status.

### 2. Deploy cAdvisor

cAdvisor is the monitoring tool. It runs as a DaemonSet in `kube-system`.

```bash
kubectl apply -f bernstein/cadvisor/cadvisor.daemonset.yaml
```

Check it:

```bash
kubectl get pods -n kube-system -l app=cadvisor
kubectl get daemonset -n kube-system cadvisor
```

Expected result: cAdvisor should be `1/1 Running`.

### 3. Deploy PostgreSQL

PostgreSQL needs its secret, config, volume, workload, and service.

```bash
kubectl apply -f bernstein/postgres/postgres.secret.yaml
kubectl apply -f bernstein/postgres/postgres.configmap.yaml
kubectl apply -f bernstein/postgres/postgres.volume.yaml
kubectl apply -f bernstein/postgres/postgres.statefulset.yaml
kubectl apply -f bernstein/postgres/postgres.service.yaml
```

Check it:

```bash
kubectl get pods -l app=postgres
kubectl get pvc
kubectl get svc postgres
```

Wait until `postgres-0` is `1/1 Running`, then create the `votes` table:

```bash
kubectl exec -i postgres-0 -- psql -U admin -d bernsteindb -c 'CREATE TABLE IF NOT EXISTS votes (id text PRIMARY KEY, vote text NOT NULL);'
```

Check the table:

```bash
kubectl exec postgres-0 -- psql -U admin -d bernsteindb -c '\dt'
```

### 4. Deploy Redis

Redis stores votes temporarily before the worker consumes them.

```bash
kubectl apply -f bernstein/redis/redis.configmap.yaml
kubectl apply -f bernstein/redis/redis.statefulset.yaml
kubectl apply -f bernstein/redis/redis.service.yaml
```

Check it:

```bash
kubectl get pods -l app=redis
kubectl get svc redis
```

Expected result: `redis-0` should be `1/1 Running`.

### 5. Deploy Poll

Poll is the web app that receives votes and sends them to Redis.

```bash
kubectl apply -f bernstein/poll/poll.statefulset.yaml
kubectl apply -f bernstein/poll/poll.service.yaml
kubectl apply -f bernstein/poll/poll.ingress.yaml
```

Check it:

```bash
kubectl get pods -l app=poll
kubectl get svc poll
kubectl get ingress poll
```

Expected result: both Poll pods should be `1/1 Running`.

### 6. Deploy Worker

Worker reads votes from Redis and writes them into PostgreSQL.

```bash
kubectl apply -f bernstein/worker/worker.statefulset.yaml
```

Check it:

```bash
kubectl get pods -l app=worker
```

Expected result: `worker-0` should be `1/1 Running`.

### 7. Deploy Result

Result is the web app that reads votes from PostgreSQL and displays the results.

```bash
kubectl apply -f bernstein/result/result.statefulset.yaml
kubectl apply -f bernstein/result/result.service.yaml
kubectl apply -f bernstein/result/result.ingress.yaml
```

Check it:

```bash
kubectl get pods -l app=result
kubectl get svc result
kubectl get ingress result
```

Expected result: both Result pods should be `1/1 Running`.

### 8. Deploy Traefik

Traefik reads the Ingress rules and exposes Poll, Result, and its dashboard.

```bash
kubectl apply -f bernstein/traefik/traefik.rbac.yaml
kubectl apply -f bernstein/traefik/traefik.statefulset.yaml
kubectl apply -f bernstein/traefik/traefik.service.yaml
```

Check it:

```bash
kubectl get pods -n kube-public -l app=traefik
kubectl get svc -n kube-public traefik
```

Expected result: both Traefik pods should be `1/1 Running`.

### 9. Configure Local Hostnames

Poll and Result use hostnames. Add them to your local `/etc/hosts` file:

```bash
echo "127.0.0.1 poll.dop.io result.dop.io" | sudo tee -a /etc/hosts
```

### Quick Deploy

If you already understand the deployment order, you can apply everything at
once:

```bash
kubectl apply --recursive -f bernstein
```

## 🌐 Access

| Application | URL |
| --- | --- |
| 🗳️ Poll | http://poll.dop.io:30021 |
| 📊 Result | http://result.dop.io:30021 |
| 🔀 Traefik dashboard | http://localhost:30042/dashboard/ |

## ✅ Verify

Check pods:

```bash
kubectl get pods -A
```

Check services:

```bash
kubectl get svc -A
```

Check ingresses:

```bash
kubectl get ingress -A
```

Check cAdvisor. It is not exposed through Traefik and is verified through
Kubernetes:

```bash
kubectl get pods -n kube-system -l app=cadvisor
kubectl get daemonset -n kube-system cadvisor
```

Check PostgreSQL tables:

```bash
kubectl exec postgres-0 -- psql -U admin -d bernsteindb -c '\dt'
```

Test HTTP routes:

```bash
curl -I http://poll.dop.io:30021
curl -I http://result.dop.io:30021
curl -I http://localhost:30042/dashboard/
```

> Expected result: the HTTP checks should return `200 OK`.

## 🛠️ Useful Commands

Describe a pod:

```bash
kubectl describe pod <pod-name>
```

Read logs:

```bash
kubectl logs <pod-name>
```

Read Traefik logs:

```bash
kubectl logs -n kube-public -l app=traefik
```

Read cAdvisor logs:

```bash
kubectl logs -n kube-system -l app=cadvisor
```

## 🔧 Troubleshooting

If `poll.dop.io:30021` or `result.dop.io:30021` does not open, check that
`/etc/hosts` contains:

```text
127.0.0.1 poll.dop.io result.dop.io
```

If pods stay in `Pending`, check resources and volumes:

```bash
kubectl get events -A --sort-by=.lastTimestamp
kubectl get pvc -A
```

If Traefik does not route requests, check:

```bash
kubectl get ingress -A
kubectl get svc -n kube-public traefik
kubectl logs -n kube-public -l app=traefik
```

## 🧹 Cleanup

Delete all project resources:

```bash
kubectl delete --recursive -f bernstein
```
