# Bernstein

Bernstein is a Kubernetes deployment for a voting application. The project
orchestrates the application services, databases, reverse proxy, and monitoring
tool required by the T-DOP-603 subject.

## Architecture

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

- `poll`: Flask web application receiving votes.
- `redis`: queue storing votes before processing.
- `worker`: Java application moving votes from Redis to PostgreSQL.
- `postgres`: persistent database storing votes in the `votes` table.
- `result`: Node.js web application displaying results.
- `traefik`: reverse proxy and load balancer.
- `cadvisor`: monitoring daemon scheduled on every node.

## Requirements

- Docker Desktop with Kubernetes enabled
- `kubectl`
- A running Kubernetes context

Check that Kubernetes is ready:

```bash
kubectl get nodes
```

## Deploy

Apply every manifest:

```bash
kubectl apply --recursive -f bernstein
```

Create the `votes` table:

```bash
kubectl exec -i postgres-0 -- psql -U admin -d bernsteindb -c 'CREATE TABLE IF NOT EXISTS votes (id text PRIMARY KEY, vote text NOT NULL);'
```

Map the local hostnames:

```bash
echo "127.0.0.1 poll.dop.io result.dop.io" | sudo tee -a /etc/hosts
```

## Access

| Application | URL |
| --- | --- |
| Poll | http://poll.dop.io:30021 |
| Result | http://result.dop.io:30021 |
| Traefik dashboard | http://localhost:30042/dashboard/ |

## Verify

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

Check cAdvisor:

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

Expected result: the HTTP checks should return `200 OK`.

## Useful Commands

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

## Troubleshooting

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

## Cleanup

Delete all project resources:

```bash
kubectl delete --recursive -f bernstein
```
