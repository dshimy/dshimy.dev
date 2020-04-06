---
title: "Google Cloud SQL with Rails 6 and GKE/K8S"
date: 2020-03-05T08:00:00-07:00
draft: false
tags: ["k8s", "rails", "gke", "cloud sql", "mysql"]
summary: "Learn how to connect your Rails 6 application to Google Cloud SQL (MySQL) in GKE/K8S."
---
Google Cloud provides a fully managed database service, [Cloud SQL](https://cloud.google.com/sql), running either MySQL or Postgres.  The only reason not to use Cloud SQL is the cost.  Google will charge you twice as much as you running the database in your own compute instance.  For that cost, you gain a highly available master-master database with virtually instant failover.  In most cases, you won't be able to beat Google on the high availability as they have access to networking infrastructure that you won't have.

We'll take a look at:

- Starting our instance and creating a database
- Installing the Cloud SQL Proxy
- Connecting out Rails 6 application to the database

## Getting up and running

Let's create a cloud SQL instance with a failover.  With this configuration:

- There is a failover server acting as a master-master
- You will be billed for two instances
- Times are in UTC
- Backups start at midnight PT
- Maintenance starts Sunday at 1:00AM PT

```sh
gcloud sql instances create db01 \
  --gce-zone us-west1-a \
  --tier db-n1-standard-1 \
  --database-version MYSQL_5_7 \
  --enable-bin-log \
  --maintenance-window-hour 9 \
  --maintenance-window-day SUN \
  --maintenance-release-channel production \
  --backup-start-time="08:00" \
  --database-flags=slow_query_log=on,log_output=FILE,log_queries_not_using_indexes=on,long_query_time=1 \
  --failover-replica-name db01-failover
```

After a few minutes that instance should be up and running.  You can check the status with:

```sh
gcloud sql instances list
```

With our instance running, let's create a database for our application:

```sh
gcloud sql databases create app_production \
  -i db01 \
  --charset utf8mb4 \
  --collation utf8mb4_general_ci
```

## Creating database users

We are going to create two sets of credentials.  The first is for you.  At some point you are going to connect to the production database and you will need a user.

```sh
gcloud sql users create USERNAME --host=% --instance=db01 --password=PASSWORD
```

Let's create another user for our application to connect to the instance and store the credentials as a secret in GKE where we can get to them later.

```sh
export DB_PASSWORD=`openssl rand -base64 32`
gcloud sql users create webuser --host=% --instance=db01 --password=${DBPASSWORD}
kubectl create secret generic cloudsql-db-credentials --from-literal=username=webuser --from-literal=password=${DBPASSWORD}
```

## Connecting to the database locally

Connecting to a Cloud SQL database requires a little more work than just supplying a host, username, and password.  You need to run something called Cloud SQL Proxy.  You connect to the proxy, not the database itself.  To test the new database, we will connect from out local computer.

```sh
cloud_sql_proxy -instances=gcp-project:us-west1:db01=tcp:3309
```

Since most will be running MySQL locally, we will change the port from `3306` to `3309`. Be sure to replace `gcp-project` with your GCP project name.  The command will output something like this:

```sh
2020/04/05 17:06:14 Rlimits for file descriptors set to {&{8500 9223372036854775807}}
2020/04/05 17:06:17 Listening on 127.0.0.1:3309 for gcp-project:us-west1:db01
2020/04/05 17:06:17 Ready for new connections
```

Using another terminal window, connect to the database using the username and password from the previous section:

```sh
mysql --host=127.0.0.1 --port=3309 --user=USERNAME --password
```

## Connecting from GKE

Connecting from our application requires running a sidecar.  This sidecar is a Cloud SQL Proxy container and our application will connect to the proxy to access the database similarly to our local access in the previous section.

### Passing credentials

When we created the database user for our application, we stored the credentials as a secret in GKE.  We will pass those in via K8S:

```yml
- env:
  - name: MYSQL_PASSWORD
    valueFrom:
      secretKeyRef:
        key: password
        name: cloudsql-db-credentials
  - name: MYSQL_USERNAME
    valueFrom:
      secretKeyRef:
        key: username
        name: cloudsql-db-credentials
```

### Configuring the application

Make a small tweak to out `config/database.yml` file to read the credentials from the `ENV`:

```yml
production:
  adapter:  mysql2
  encoding: utf8mb4
  pool:     <%= ENV.fetch("RAILS_MAX_THREADS") { 20 } %>
  username: <%= ENV["MYSQL_USERNAME"] %>
  password: <%= ENV["MYSQL_PASSWORD"] %>
  host:     127.0.0.1
  database: app_production
```

### Google Cloud Proxy sidecar

Lastly, add the following sidecar configuration for the Google Cloud Proxy:

```yml
- name: cloudsql-proxy
  image: gcr.io/cloudsql-docker/gce-proxy:1.16
  command:
  - /cloud_sql_proxy
  - -instances=$(CLOUD_SQL_INSTANCE)=tcp:3306
  - -credential_file=/secrets/cloudsql/credentials.json
  - term_timeout=10s
  lifecycle:
    preStop:
      exec:
        command:
        - sleep
        - "20"
  volumeMounts:
  - mountPath: /secrets/cloudsql
    name: cloudsql-instance-credentials
    readOnly: true
```

When a new version of your application is deployed, you cannot control the order the containers are shutdown.  If the proxy is shutdown before the application, users will see database connection errors.  The `lifecycle` section keeps the proxy around for 20 seconds which is usually enough time to shutdown the application.  We also add `-term_timeout=10s` to the command to give the proxy time to allow current connections to close.  With the `sleep` command, I am not convinced the term timeout is needed.

## References

- [Google Cloud SQL](https://cloud.google.com/sql)
- [Cloud SQL Proxy](https://cloud.google.com/sql/docs/mysql/sql-proxy)
- [Cloud SQL Proxy Client](https://github.com/GoogleCloudPlatform/cloudsql-proxy)
- [Configuring MySQL Database Flags](https://cloud.google.com/sql/docs/mysql/flags)
