---
title: 'Deploying Rails 8 on Google Cloud: GCP Foundation'
description: 'Part 1 of 4 — Setting up your Google Cloud project, networking, IAM, and security foundation for a production Rails 8 application.'
pubDate: '2024-12-15'
---

This is the first article in a four-part series on deploying Rails 8 to Google Cloud. We'll go from zero GCP to a production-ready Rails app running on GKE with Cloud SQL. In this first post, we cover the foundation: project setup, networking, IAM, and security.

## Why Google Cloud for Rails?

Google Cloud offers a tightly integrated ecosystem that works well for Rails deployments. GKE (Google Kubernetes Engine) is arguably the best managed Kubernetes offering. Cloud SQL gives you managed MySQL or PostgreSQL. Artifact Registry stores your Docker images. Cloud Storage handles file uploads via Active Storage. Everything talks to everything else with minimal glue.

If you're a solo developer or small team, GCP's managed services mean less infrastructure to babysit.

## Project and API Setup

Start by creating a dedicated GCP project for production. Keep it separate from development and staging — this makes billing, IAM, and audit logs much cleaner.

Enable the APIs you'll need:

- **Compute Engine API** — for networking and load balancers
- **Kubernetes Engine API** — for GKE
- **Cloud SQL Admin API** — for managed MySQL
- **Artifact Registry API** — for Docker image storage
- **Cloud Storage API** — for Active Storage and asset hosting

You can enable these through the console or with `gcloud services enable`.

## VPC Networking

The default VPC works fine for getting started, but for production you'll want a custom VPC with explicit subnets. This gives you control over IP ranges and avoids surprises.

Create a subnet in your target region with secondary IP ranges for Kubernetes pods and services. GKE uses these secondary ranges for IP aliasing, which gives each pod a routable IP without NAT.

A typical setup:

- **Primary range**: `10.0.0.0/24` for node IPs
- **Pod range**: `10.100.0.0/14` for pod IPs
- **Service range**: `10.104.0.0/20` for cluster service IPs

Enable **Private Google Access** on your subnet so nodes and pods can reach Google APIs without public IPs.

## IAM and Service Accounts

Follow the principle of least privilege. Create dedicated service accounts for specific roles rather than using the default compute service account for everything.

You'll want service accounts for:

- **GKE nodes** — permissions to pull images from Artifact Registry and write logs
- **Cloud SQL access** — used by the Cloud SQL Auth Proxy sidecar in your pods
- **Cloud Storage** — for Active Storage file uploads and asset serving
- **CI/CD** — for your deployment pipeline to push images and apply K8s manifests

For each service account, grant only the specific IAM roles needed. For example, your Cloud SQL service account only needs `roles/cloudsql.client`, not broad project editor access.

## Artifact Registry

Create a Docker repository in Artifact Registry in the same region as your GKE cluster. This minimizes image pull latency and avoids cross-region transfer costs.

```bash
gcloud artifacts repositories create docker \
  --repository-format=docker \
  --location=us-west1 \
  --description="Production Docker images"
```

Configure Docker authentication so your CI pipeline can push images:

```bash
gcloud auth configure-docker us-west1-docker.pkg.dev
```

Your images will be stored at `us-west1-docker.pkg.dev/YOUR_PROJECT/docker/APP_NAME`.

## Static IPs and DNS

Reserve a global static IP address for your load balancer. This IP won't change when you recreate your ingress or cluster.

```bash
gcloud compute addresses create app-static-ip --global
```

Point your domains to this IP in your DNS provider. If you're serving multiple domains from a single cluster (which is common for multi-product companies), they all point to the same IP. The GKE ingress handles routing based on the `Host` header.

## Cloud Armor

Cloud Armor sits in front of your load balancer and provides Layer 7 DDoS protection and WAF capabilities. Even a basic policy is worth setting up from day one.

Create a security policy:

```bash
gcloud compute security-policies create my-security-policy \
  --description="Production WAF policy"
```

Common rules to add:

- **Country blocking** — if your app only serves specific regions, deny traffic from known high-risk countries
- **IP blocking** — block specific IPs that are abusing your service
- **Rate limiting** — protect against brute force attacks
- **Adaptive protection** — Google's ML-based DDoS detection

Attach the policy to your backend services (we'll configure these when setting up GKE ingress in Part 3).

## Cost Management

A few tips to keep GCP costs reasonable:

- **Committed use discounts** — if you know you'll run a cluster for a year or more, commit for 1 or 3 years for significant savings on compute and Cloud SQL
- **Right-size from the start** — you can always scale up; start with the smallest machine types that meet your needs
- **Budget alerts** — set up billing alerts so unexpected spikes don't surprise you
- **Preemptible/Spot VMs** — for non-critical workloads, spot instances are 60-91% cheaper

## What's Next

With your GCP foundation in place — project, networking, IAM, Artifact Registry, static IPs, and Cloud Armor — you're ready to set up your database. In [Part 2](/blog/deploying-rails-8-cloud-sql-mysql), we'll configure Cloud SQL for MySQL with high availability, automated backups, and the multi-database setup that Rails 8 uses for Solid Queue and Solid Cache.
