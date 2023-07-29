# Tiny Links

---

Tiny Links shortens your URL by a mile! Deployed on Kubernetes cluster this scaled up Tiny Link uses Redis for caching and PostgreSQL for storage. It’s entirely built with Nest.js allowing for a scalable, fast, and responsive modular application.

## Setup DB

The directory **dbschema** has a file `quick-links.sql` with the database schema. 
![Quick Links Database Schema](dbschema/quick-links.png)

## Installation

```bash
$ npm install
```

## Running the app

```bash
# Start PostgreSQL and Redis with Docker Compose
$ docker-compose up -d db redis

$ npm run start

# Dev mode
$ npm run start:dev

# Prod mode
$ npm run start:prod
```

## Test

```bash
# Unit testing
$ npm run test

# e2e tests 
$ npm run test:e2e

# Test coverage
$ npm run test:cov
```

## Debug in VSCode

Add below configuration in `launch.json` vscode.

```
{
  "version": "0.2.0",
  "configurations": [
    {
      "type": "node",
      "request": "attach",
      "name": "Attach NestJS WS",
      "port": 9229,
      "restart": true
    }
  ]
}
```

```bash
# unit tests
$ npm run start:debug

## Setup Kubernetes

- Install `kubectl`

```sh
  brew install kubernetes-cli
```

- Install `helm`

```sh
  brew install helm
```

### Add Bitnami Chart Repo

```sh
helm repo add bitnami https://charts.bitnami.com/bitnami

```

### Setup Postgresql Cluster

```sh
helm install postgres bitnami/postgresql-ha \
--set global.postgresql.username="postgres" \
--set global.postgresql.password="password" \
--set global.postgresql.database="quicklink" \
--set global.postgresql.repmgrUsername="repmgr" \
--set global.postgresql.repmgrPassword="password" \
--set global.postgresql.repmgrDatabase="repmgr" \
--set global.pgpool.adminUsername="admin" \
--set global.pgpool.adminPassword="password" \
--set postgresqlImage.debug=true

# PostgreSQL can be accessed through Pgpool via port 5432 on the following DNS name from within your cluster:

postgres-postgresql-ha-pgpool.default.svc.cluster.local

# Pgpool acts as a load balancer for PostgreSQL and forward read/write connections
# to primary node. Read-only connections are forwarded to standby nodes.

# The password for "postgres" run:

export POSTGRES_PASSWORD=$(kubectl get secret --namespace default postgres-postgresql-ha-postgresql -o jsonpath="{.data.password}" | base64 -d)
echo $POSTGRES_PASSWORD

# The password for "repmgr" run:

export REPMGR_PASSWORD=$(kubectl get secret --namespace default postgres-postgresql-ha-postgresql -o jsonpath="{.data.repmgr-password}" | base64 -d)
echo $REPMGR_PASSWORD

# Connect your database run:

kubectl run postgres-postgresql-ha-client --rm --tty -i --restart='Never' --namespace default --image docker.io/bitnami/postgresql-repmgr:15.2.0-debian-11-r0 --env="PGPASSWORD=$POSTGRES_PASSWORD"  \
--command -- psql -h postgres-postgresql-ha-pgpool -p 5432 -U postgres -d quicklink

# Connect your database outside the cluster run:

# Port forward and connect
# Port forward and throw in backgroud process
kubectl port-forward --namespace default svc/postgres-postgresql-ha-pgpool 5433:5432 &
psql -h 127.0.0.1 -p 5432 -U postgres -d quicklink

```

### Setup Redis Cluster

```sh
# Install redis - single mast
helm install redis bitnami/redis




# Redis&reg; can be accessed on the following DNS names within your cluster:

redis-master.default.svc.cluster.local for read/write operations (port 6379)
redis-replicas.default.svc.cluster.local for read-only operations (port 6379)



# Get your password run:

export REDIS_PASSWORD=$(kubectl get secret --namespace default redis -o jsonpath="{.data.redis-password}" | base64 -d)

# Connect to your Redis&reg; server:

# Run a Redis&reg; pod used as a client:

kubectl run --namespace default redis-client --restart='Never'  --env REDIS_PASSWORD=$REDIS_PASSWORD  --image docker.io/bitnami/redis:7.0.8-debian-11-r0 --command -- sleep infinity

To attach to the pod run:

kubectl exec --tty -i redis-client \
--namespace default -- bash

#  Connect using the Redis&reg; CLI:
REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h redis-master
REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h redis-replicas

# Connect to database outside the cluster run:

kubectl port-forward --namespace default svc/redis-master 6379:6379 &
REDISCLI_AUTH="$REDIS_PASSWORD" redis-cli -h 127.0.0.1 -p 6379
```

### Setup Kubernetes Dashboard

- Create a service account
  ```sh
  kubectl apply -f ./kubernetes/service-accounts/k8s-dashboard.service-account.yaml
  ```
- Create cluster-level role binding
  ```sh
  kubectl apply -f ./kubernetes/service-accounts/k8s-cluster-level.role-binding.yaml
  ```
  We can also create a role-binding for a namespace
  ```sh
  kubectl apply -f ./kubernetes/service-accounts/k8s-dashboard.role-binding.yaml
  ```
- Generate access token for a service account

  ```sh
  # Get token for service account run:

  kubectl describe secret $(kubectl get secret | grep dashboard-sa | awk '{print $1}') | grep token:
  #If above doesn't work - dashboard-sa => is service account
  kubectl create token dashboard-sa
  ```

  ***
