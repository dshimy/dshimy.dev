---
title: 'Deploying Rails 8 on Google Cloud: Containers, Deploys, and Production Config'
description: 'Part 4 of 4 — Dockerizing Rails 8, writing Kubernetes manifests, running migrations, and deploying with zero downtime to GKE.'
pubDate: '2025-04-15'
---

This is the final article in our four-part series on deploying Rails 8 to Google Cloud. We've built the [GCP foundation](/blog/deploying-rails-8-gcp-foundation), configured [Cloud SQL](/blog/deploying-rails-8-cloud-sql-mysql), and set up the [GKE cluster](/blog/deploying-rails-8-gke-cluster). Now we'll containerize Rails 8, write the deployment manifests, and ship it.

## The Dockerfile

A multi-stage Docker build keeps your production image small and secure. Three stages: base, build, and production.

### Base Stage

Start with Ruby slim and install only the runtime dependencies your app needs:

```dockerfile
FROM ruby:3.4-slim AS base

RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y \
    curl default-mysql-client libjemalloc2 libvips && \
    rm -rf /var/lib/apt/lists/*

ENV RAILS_ENV=production \
    BUNDLE_DEPLOYMENT=1 \
    BUNDLE_WITHOUT="development:test"

WORKDIR /rails
```

Key packages:

- **libjemalloc2** — alternative memory allocator that significantly reduces Ruby's memory bloat
- **libvips** — image processing for Active Storage variants
- **default-mysql-client** — for database health checks and migrations

### Build Stage

Install build dependencies, bundle gems, and precompile assets:

```dockerfile
FROM base AS build

RUN apt-get update -qq && \
    apt-get install --no-install-recommends -y \
    build-essential default-libmysqlclient-dev git pkg-config

COPY Gemfile Gemfile.lock ./
RUN bundle install && \
    rm -rf ~/.bundle/ "${BUNDLE_PATH}"/ruby/*/cache

COPY . .
RUN SECRET_KEY_BASE_DUMMY=1 ./bin/rails assets:precompile
RUN bundle exec bootsnap precompile app/ lib/
```

The `SECRET_KEY_BASE_DUMMY=1` trick lets asset precompilation run without a real secret key — you don't want production secrets baked into your Docker image.

Bootsnap precompilation caches Ruby and YAML file parsing, cutting boot time significantly.

### Production Stage

Copy only what you need from the build stage and run as a non-root user:

```dockerfile
FROM base AS production

RUN groupadd --system --gid 1000 rails && \
    useradd rails --uid 1000 --gid 1000 --create-home --shell /bin/bash

COPY --from=build /rails /rails
COPY --from=build /usr/local/bundle /usr/local/bundle

USER 1000:1000
EXPOSE 3000
CMD ["./bin/thrust", "./bin/rails", "server"]
```

Running as a non-root user (UID 1000) is a security best practice. If your container is compromised, the attacker has limited privileges.

### Jemalloc

Add jemalloc to your entrypoint script to reduce memory usage:

```bash
#!/bin/bash
export LD_PRELOAD=/usr/lib/x86_64-linux-gnu/libjemalloc.so.2
exec "$@"
```

In practice, jemalloc reduces Rails memory usage by 20-30%. On a cluster running multiple replicas, this adds up.

## Thruster and Puma

Rails 8 introduced **Thruster**, a lightweight HTTP proxy that wraps Puma. It handles asset caching, gzip compression, and X-Sendfile — things you'd normally need nginx for.

Your container entrypoint is simply:

```
bin/thrust ./bin/rails server
```

Thruster listens on port 80 and proxies to Puma on port 3000. For Kubernetes, you can configure Puma directly on port 3000 and let Thruster handle the rest.

Configure Puma in `config/puma.rb`:

```ruby
threads_count = ENV.fetch("RAILS_MAX_THREADS") { 3 }
threads threads_count, threads_count

port ENV.fetch("PORT") { 3000 }
environment ENV.fetch("RAILS_ENV") { "development" }

plugin :solid_queue if ENV["SOLID_QUEUE_IN_PUMA"]
```

The `plugin :solid_queue` line is optional — it lets Puma supervise Solid Queue in development. In production, you'll run Solid Queue as a separate deployment.

## Kubernetes Manifests

### Web Deployment

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-deployment
spec:
  replicas: 2
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  selector:
    matchLabels:
      app: myapp
      role: web
  template:
    spec:
      containers:
        - name: app
          image: us-west1-docker.pkg.dev/PROJECT/docker/myapp:latest
          ports:
            - containerPort: 3000
              name: http
          envFrom:
            - secretRef:
                name: app-secrets
          resources:
            requests:
              memory: "625Mi"
              cpu: "100m"
          readinessProbe:
            httpGet:
              path: /up
              port: 3000
            periodSeconds: 5
            successThreshold: 3
            failureThreshold: 2
        - name: cloud-sql-proxy
          image: gcr.io/cloud-sql-connectors/cloud-sql-proxy:2.15.0
          args:
            - "--private-ip"
            - "--port=3306"
            - "PROJECT:REGION:INSTANCE"
          resources:
            requests:
              memory: "15Mi"
              cpu: "3m"
```

Key decisions:

- **2 replicas minimum** — one pod can serve traffic while the other is being updated
- **maxUnavailable: 0** — during rolling updates, all existing pods stay running until new ones pass their readiness check
- **625Mi memory** — a reasonable starting point for a Rails app with Thruster; adjust based on your app's actual usage
- **Cloud SQL Proxy sidecar** — runs alongside your app in every pod

### Job Worker Deployment

Run Solid Queue workers as a separate deployment with the same image but a different command:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-jobs-deployment
spec:
  replicas: 1
  template:
    spec:
      containers:
        - name: jobs
          image: us-west1-docker.pkg.dev/PROJECT/docker/myapp:latest
          command: ["bundle", "exec", "rake", "solid_queue:start"]
          envFrom:
            - secretRef:
                name: app-secrets
          resources:
            requests:
              memory: "625Mi"
              cpu: "100m"
        - name: cloud-sql-proxy
          # same config as web deployment
```

One replica is usually sufficient for job workers. Scale up if your queue depth grows.

## Database Migrations

Don't run migrations inside your app containers. If two replicas both try to migrate simultaneously, you'll get lock contention or duplicate migrations.

Instead, run migrations as a Kubernetes Job:

```yaml
apiVersion: batch/v1
kind: Job
metadata:
  name: db-migrate
spec:
  template:
    spec:
      containers:
        - name: migrate
          image: us-west1-docker.pkg.dev/PROJECT/docker/myapp:latest
          command: ["bin/rails", "db:migrate"]
          envFrom:
            - secretRef:
                name: app-secrets
        - name: cloud-sql-proxy
          # same sidecar config
      restartPolicy: Never
  backoffLimit: 1
```

In your deploy script, delete any previous migration job, create the new one, and poll until it completes:

```bash
kubectl delete job db-migrate --ignore-not-found
kubectl apply -f k8s/db-migrate-job.yml
kubectl wait --for=condition=complete job/db-migrate --timeout=1200s
```

Run migrations **before** deploying the new app image. This ensures the database schema is ready when new pods start.

## Asset Pipeline and CDN

Extract compiled assets from the Docker image and sync them to Cloud Storage for CDN serving:

```bash
# Create a temporary container to copy assets out
docker create --name assets myapp:latest
docker cp assets:/rails/public/assets ./tmp/assets
docker rm assets

# Sync to Cloud Storage with immutable cache headers
gsutil -m rsync -r -d \
  -h "Cache-Control:public, max-age=31536000, immutable" \
  ./tmp/assets gs://your-bucket/assets/
```

Configure Rails to serve assets from the CDN:

```ruby
# config/environments/production.rb
config.asset_host = "https://cdn.example.com"
```

This offloads static asset traffic from your Rails pods entirely.

## The Deploy Script

Put it all together in a deploy script:

```bash
#!/bin/bash
set -e

APP_NAME="myapp"
IMAGE="us-west1-docker.pkg.dev/PROJECT/docker/$APP_NAME"
TAG=$(git rev-parse --short HEAD)

# 1. Build and push
docker build -t $IMAGE:$TAG -t $IMAGE:latest .
docker push $IMAGE:$TAG
docker push $IMAGE:latest

# 2. Sync assets to CDN
# (extract from image and gsutil rsync as above)

# 3. Run migrations
kubectl delete job db-migrate --ignore-not-found
kubectl apply -f k8s/db-migrate-job.yml
kubectl wait --for=condition=complete job/db-migrate --timeout=1200s

# 4. Deploy
kubectl set image deployment/app-deployment app=$IMAGE:$TAG
kubectl set image deployment/app-jobs-deployment jobs=$IMAGE:$TAG

# 5. Wait for rollout
kubectl rollout status deployment/app-deployment --timeout=300s

echo "Deploy complete: $TAG"
```

## Image Tagging Strategy

Maintain three tags for rollback capability:

- **Commit SHA** — immutable reference to exactly what's deployed
- **latest** — always points to the current production image
- **rollback** — points to the previous production image

Before deploying, tag the current `latest` as `rollback`. If something goes wrong:

```bash
kubectl rollout undo deployment/app-deployment
```

## Production Rails 8 Configuration

A few Rails settings that matter in production:

```ruby
# config/environments/production.rb

# SSL behind a load balancer
config.force_ssl = true
config.assume_ssl = true  # trust X-Forwarded-Proto from LB

# Logging
config.log_to_stdout = true
config.log_level = :info

# Caching with Solid Cache
config.cache_store = :solid_cache_store

# Background jobs with Solid Queue
config.active_job.queue_adapter = :solid_queue

# Active Storage with Google Cloud Storage
config.active_storage.service = :google
```

The `assume_ssl` setting is important when running behind a load balancer that terminates TLS. Without it, Rails thinks every request is HTTP and triggers redirect loops.

## Wrapping Up the Series

Over these four articles, we've gone from an empty GCP project to a production Rails 8 deployment:

1. [**GCP Foundation**](/blog/deploying-rails-8-gcp-foundation) — project, networking, IAM, Cloud Armor
2. [**Cloud SQL**](/blog/deploying-rails-8-cloud-sql-mysql) — managed MySQL with HA, backups, multi-database
3. [**GKE Cluster**](/blog/deploying-rails-8-gke-cluster) — Kubernetes, ingress, TLS, monitoring
4. **This article** — Docker, deploys, migrations, CDN

This architecture serves FutureFund well — multiple Rails apps running on a single cluster, each with their own database, job workers, and domain routing. It's resilient, manageable by a small team, and cost-effective.

The beauty of this setup is that it scales with you. Need more capacity? Add replicas or upgrade node types. New app? Add a deployment and ingress rule. The foundation stays the same.
