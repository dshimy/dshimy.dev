---
title: 'Deploying Rails 8 on Google Cloud: Cloud SQL for MySQL'
description: 'Part 2 of 4 — Configuring a production-ready Cloud SQL MySQL instance with high availability, backups, and Rails 8 multi-database support.'
pubDate: '2025-01-15'
---

This is the second article in a four-part series on deploying Rails 8 to Google Cloud. In [Part 1](/blog/deploying-rails-8-gcp-foundation), we set up the GCP foundation. Now we'll configure Cloud SQL for MySQL — the managed database that will power your Rails app, Solid Queue, and Solid Cache.

## Why Cloud SQL Over Self-Managed MySQL

You could run MySQL in a container on GKE. Don't. Cloud SQL handles automated backups, point-in-time recovery, high availability failover, patching, and monitoring. That's a lot of operational work you don't have to do. For a small team, this is a no-brainer trade.

The cost premium over self-managed is modest, and the first time Cloud SQL saves you from a failed disk or corrupted database, it pays for itself.

## Creating Your Instance

When creating a Cloud SQL instance, you'll make a few key decisions:

**MySQL version**: MySQL 8.0 is the stable choice. MySQL 8.4 is available but newer — evaluate whether your gems and queries are compatible before jumping to it.

**Machine tier**: Cloud SQL uses custom machine types specified as `db-custom-VCPU-RAM`. For a typical Rails app:

- **Starting out**: `db-custom-1-3840` (1 vCPU, 3.8GB RAM) handles light to moderate traffic
- **Production**: `db-custom-4-8192` (4 vCPU, 8GB RAM) gives breathing room for concurrent connections and complex queries
- Scale up from there based on Query Insights data

**Storage**: Always choose SSD (pd-ssd). Enable auto-resize so you never run out of disk space at 3am. Set a reasonable starting size (100GB is common) and let it grow as needed.

## High Availability

For production, enable **Regional HA**. This creates a synchronous standby instance in a different zone within the same region. If the primary zone goes down, Cloud SQL automatically fails over to the standby.

What your app experiences during failover:

- A brief connection interruption (typically under 30 seconds)
- Existing connections are dropped and must be re-established
- No data loss — replication is synchronous

Rails handles reconnection gracefully via Active Record's connection pool, but make sure your `database.yml` has `reconnect: true` and a reasonable `checkout_timeout`.

## Read Replicas

If you have heavy read traffic or reporting queries that you don't want impacting your primary, create a read replica. A replica runs in a separate zone and serves read-only queries.

In Rails 8, you can direct reads to the replica using the multi-database reading role:

```ruby
# config/database.yml
production:
  primary:
    host: 127.0.0.1
    port: 3306
  primary_replica:
    host: 127.0.0.1
    port: 3307
    replica: true
```

The replica gets its own Cloud SQL Proxy sidecar in your pods, connecting on a different local port.

## Backup and Recovery

Configure automated backups from day one:

- **Daily automated backups** — schedule during your lowest traffic window
- **Retention period** — 21 days gives you three weeks of recovery points
- **Transaction log retention** — 7 days of binary logs enables point-in-time recovery to any second within that window
- **Binary logging** — must be enabled (it is by default with HA)

Test your restore process before you need it. Create a clone from a backup at least once to verify your data is recoverable. A backup you've never tested isn't a backup.

## Security

**Private IP only**: Configure your instance with a private IP via VPC peering. This means your database is only reachable from within your VPC — not from the public internet.

**SSL/TLS enforcement**: Require client certificates for all connections. Cloud SQL can enforce `TRUSTED_CLIENT_CERTIFICATE_REQUIRED` mode, which means even if someone gets network access, they can't connect without a valid certificate.

**Deletion protection**: Always enable this. It prevents accidental deletion of your production database via the console or API. You can disable it temporarily when you actually need to delete an instance.

**Authorized networks**: Even with private IP, you may want emergency access from a specific IP (like your office or a bastion host). Add these as authorized networks sparingly.

## Performance Tuning

**Query Insights**: Enable this on your instance. It gives you a dashboard of slow queries, query plans, and wait events without installing any monitoring agents.

- Set the slow query threshold to 500ms (0.5s) — this catches queries that are slow but not obviously broken
- Enable logging of queries without indexes — these are the queries that will hurt as your data grows
- Record application tags so you can trace slow queries back to specific Rails controllers or jobs

**Connection pooling**: Rails manages its own connection pool. Set `pool` in your `database.yml` to match `RAILS_MAX_THREADS` (typically 3-5 per Puma worker). Don't over-provision — each connection consumes memory on the database.

## Multi-Database Setup for Rails 8

Rails 8 introduced Solid Queue (replacing Sidekiq/Redis for background jobs) and Solid Cache (replacing Redis/Memcached for caching). Both use the database as their backend. Running them against your primary database works, but dedicated databases isolate their write load.

Set up four databases:

- **primary** — your application data
- **queue** — Solid Queue jobs and processes
- **cache** — Solid Cache entries
- **cable** — Action Cable broadcast messages

In `database.yml`:

```yaml
production:
  primary:
    adapter: mysql2
    database: myapp_production
    host: 127.0.0.1
    port: 3306
    username: <%= ENV["MYSQL_USERNAME"] %>
    password: <%= ENV["MYSQL_PASSWORD"] %>
    pool: <%= ENV.fetch("RAILS_MAX_THREADS") { 5 } %>
  cache:
    adapter: mysql2
    database: myapp_production_cache
    host: 127.0.0.1
    port: 3306
    migrations_paths: db/cache_migrate
  queue:
    adapter: mysql2
    database: myapp_production_queue
    host: 127.0.0.1
    port: 3306
    migrations_paths: db/queue_migrate
  cable:
    adapter: mysql2
    database: myapp_production_cable
    host: 127.0.0.1
    port: 3306
    migrations_paths: db/cable_migrate
```

All four databases live on the same Cloud SQL instance — you don't need separate instances. The isolation is logical, not physical, but it prevents Solid Queue's frequent polling writes from contending with your application's transaction locks.

## Maintenance Windows

Schedule maintenance updates during your lowest-traffic period. Cloud SQL periodically applies patches and minor updates. By setting a preferred window (e.g., Saturday mornings), you avoid surprises during peak hours.

## What's Next

Your database is configured with HA, backups, security, and multi-database support for Rails 8. In [Part 3](/blog/deploying-rails-8-gke-cluster), we'll build the GKE cluster that will host your Rails application, configure networking and ingress, set up cert-manager for automatic TLS, and connect to Cloud SQL using the Auth Proxy sidecar.
