---
title: 'Deploying Rails 8 on Google Cloud: GKE Cluster Setup'
description: 'Part 3 of 4 — Building a production GKE cluster with ingress, TLS, Cloud SQL Proxy, health checks, and monitoring for Rails 8.'
pubDate: '2025-02-15'
---

This is the third article in a four-part series on deploying Rails 8 to Google Cloud. We've set up the [GCP foundation](/blog/deploying-rails-8-gcp-foundation) and [Cloud SQL](/blog/deploying-rails-8-cloud-sql-mysql). Now we'll build the Kubernetes cluster that will run your Rails application.

If you're new to Kubernetes, don't worry — GKE handles the hard parts. You'll need to understand deployments, services, ingress, and secrets. That's about it for a Rails app.

## Cluster Creation

**Zonal vs Regional**: A zonal cluster runs the control plane in a single zone. A regional cluster replicates it across three zones. Regional is more resilient but costs more. For most Rails apps, a zonal cluster with multiple nodes provides a good balance of cost and availability.

**Release channel**: Use the **Regular** channel. It's the sweet spot — you get stable Kubernetes versions with monthly updates. Rapid is too bleeding-edge for production; Stable lags too far behind on security patches.

**Node pool**: Start with three nodes. This gives you resilience (one node can go down for maintenance while two handle traffic) and room for rolling updates.

For machine types, `n2-standard-4` (4 vCPU, 16GB RAM) is a solid starting point for Rails apps. Each node comfortably runs several Rails deployments with their sidecars. You can always resize the node pool later.

## Node Configuration

**Container-Optimized OS (COS)**: This is the default and you should keep it. It's a minimal, hardened OS designed specifically for running containers. Less attack surface, faster boot, automatic updates.

**Auto-repair and auto-upgrade**: Enable both. Auto-repair replaces unhealthy nodes automatically. Auto-upgrade keeps nodes on the latest patch version within your release channel.

**Shielded nodes**: Enable integrity monitoring. This verifies that node boot software hasn't been tampered with. It's free and there's no reason not to use it.

**Disk**: 100GB `pd-balanced` (SSD-backed) gives good performance for container image caching and ephemeral storage. Avoid standard persistent disks — the IOPS difference matters for container startup times.

## Networking and Ingress

### IP Aliases

GKE uses IP aliases to assign routable IPs to pods and services from secondary subnet ranges. This is the default and recommended mode. It eliminates the need for kube-proxy NAT rules and integrates cleanly with GCP networking.

### Ingress for Multi-Domain Routing

A single GKE Ingress resource can route traffic for multiple domains to different backend services. This is ideal if you run multiple apps (or a multi-tenant app) on the same cluster.

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: app-ingress
  annotations:
    kubernetes.io/ingress.global-static-ip-name: app-static-ip
spec:
  rules:
    - host: app.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: app-service
                port:
                  number: 80
    - host: api.example.com
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: api-service
                port:
                  number: 80
```

The ingress controller creates a GCP HTTP(S) Load Balancer behind the scenes, with URL maps for host-based and path-based routing.

### Services

Use `NodePort` services for your Rails apps. The GKE ingress controller requires NodePort (or LoadBalancer) type services as backends. Traffic flows: Internet → Load Balancer → NodePort → Pod.

```yaml
apiVersion: v1
kind: Service
metadata:
  name: app-service
spec:
  type: NodePort
  ports:
    - port: 80
      targetPort: 3000
      protocol: TCP
      name: http
  selector:
    app: myapp
```

## TLS with cert-manager

Install cert-manager to automate TLS certificate provisioning and renewal via Let's Encrypt.

```bash
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.14.0/cert-manager.yaml
```

Create a ClusterIssuer for Let's Encrypt production:

```yaml
apiVersion: cert-manager.io/v1
kind: ClusterIssuer
metadata:
  name: letsencrypt-production
spec:
  acme:
    server: https://acme-v02.api.letsencrypt.org/directory
    email: your-email@example.com
    privateKeySecretRef:
      name: letsencrypt-production-key
    solvers:
      - http01:
          ingress:
            class: gce
```

Reference it in your ingress with a TLS block, and cert-manager will automatically obtain and renew certificates. Certificates renew 30 days before expiration — you'll never manually manage a cert again.

## Cloud SQL Auth Proxy Sidecar

Your Rails pods connect to Cloud SQL through the Auth Proxy, which runs as a sidecar container in each pod. The proxy handles authentication, encryption, and connection management.

```yaml
- name: cloud-sql-proxy
  image: gcr.io/cloud-sql-connectors/cloud-sql-proxy:2.15.0
  args:
    - "--private-ip"
    - "--structured-logs"
    - "--port=3306"
    - "PROJECT:REGION:INSTANCE"
  resources:
    requests:
      memory: "15Mi"
      cpu: "3m"
  securityContext:
    runAsNonRoot: true
  volumeMounts:
    - name: sql-credentials
      mountPath: /secrets/
      readOnly: true
```

The proxy connects via private IP (no public internet exposure) and uses a service account JSON key mounted as a volume. Your Rails app connects to `127.0.0.1:3306` as if MySQL were local.

Keep the proxy's resource requests minimal — it's lightweight. 15Mi memory and 3m CPU is typically enough.

## Health Checks

Rails 8 ships with a built-in health check endpoint at `/up`. Use this for your Kubernetes readiness probe:

```yaml
readinessProbe:
  httpGet:
    path: /up
    port: 3000
  periodSeconds: 5
  successThreshold: 3
  failureThreshold: 2
```

The readiness probe tells Kubernetes when a pod is ready to receive traffic. During deployments, new pods won't receive requests until `/up` returns 200. This is essential for zero-downtime rolling updates.

Don't confuse readiness with liveness probes. A liveness probe restarts your container if it's stuck. For Rails, the readiness probe on `/up` is usually sufficient — you don't want Kubernetes restarting pods just because they're temporarily slow.

## Secrets Management

Store sensitive values in Kubernetes Secrets:

- **Rails master key** — `RAILS_MASTER_KEY` for decrypting credentials
- **Database credentials** — `MYSQL_USERNAME` and `MYSQL_PASSWORD`
- **Service account keys** — JSON files for Cloud SQL and Cloud Storage access

```bash
kubectl create secret generic app-secrets \
  --from-literal=RAILS_MASTER_KEY=your-master-key \
  --from-literal=MYSQL_USERNAME=your-db-user \
  --from-literal=MYSQL_PASSWORD=your-db-password
```

Mount service account keys as volumes (read-only) rather than environment variables. This prevents them from appearing in process listings or crash dumps.

For a more secure approach, consider GKE Workload Identity, which eliminates the need for service account JSON keys entirely by mapping Kubernetes service accounts to GCP IAM service accounts.

## Monitoring and Logging

**Managed Prometheus**: GKE integrates with Google Cloud Managed Service for Prometheus. Enable it on your cluster for metrics collection without running your own Prometheus server.

**Cloud Logging**: By default, GKE ships all container stdout/stderr to Cloud Logging. Configure your Rails app to log to stdout (the Rails 8 default) and use structured JSON logging so you can filter and search logs effectively.

Silence noisy health check logs in your Rails config:

```ruby
config.silence_healthcheck_path = "/up"
```

## Maintenance Windows

Set a maintenance window for node upgrades and cluster updates. Choose a time when traffic is lowest — early morning on weekends works well for most apps. During maintenance, GKE performs surge upgrades: it creates a new node, drains an old one, and repeats. Your app stays available throughout.

## What's Next

Your GKE cluster is running with ingress, TLS, Cloud SQL connectivity, health checks, and monitoring. In [Part 4](/blog/deploying-rails-8-to-gke), we'll build the Docker image, write the Kubernetes manifests, and deploy Rails 8 with zero-downtime rolling updates, database migrations, and asset serving via CDN.
