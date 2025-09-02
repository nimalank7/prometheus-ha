# Prometheus Sharding in Kubernetes using Prometheus Operator
## Install the Prometheus Operator

1. Setup a Kubernetes cluster

```
k3d cluster create monitoring --api-port 6550 -p "8081:80@loadbalancer"
```

2. Install Prometheus Operator via the Helm Chart

```
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
CHART_VERSION=75.4.0
helm install kube-prometheus-stack prometheus-community/kube-prometheus-stack \
  --version ${CHART_VERSION} \
  --namespace monitoring \
  --create-namespace \
  --values values.yaml
```

3. Check CRDs

```
kubectl get crds
```

4. Check monitoring

```
kubectl -n monitoring get pods
kubectl -n monitoring get servicemonitors
```

Prometheus Operator will create a default Prometheus instance in the `monitoring` namespace.

## Run the applications

1. Build the applications using docker compose

```
docker compose -f docker-compose.yaml build 
```

2. Load the images into our cluster

```
k3d image import go-application -c monitoring
k3d image import dotnet-application -c monitoring
k3d image import python-application -c monitoring
k3d image import nodejs-application -c monitoring
```

3. Deploy our microservices:

```
kubectl apply -f ../go-application/deployment.yaml
kubectl apply -f ../dotnet-application/deployment.yaml
kubectl apply -f ../python-application/deployment.yaml
kubectl apply -f ../nodejs-application/deployment.yaml
kubectl get pods
```

## Create a service account

Service account with RBAC permissions needs to be created to allow Prometheus instances to access service monitors and allow scraping of service endpoints:

```
kubectl apply -f serviceaccount.yaml
```

### Apply Service Monitors

There are 4 ServiceMonitor which correspond to our applications respectively. Each ServiceMonitor has an extra label
called `prometheus-shard` which indicates which Prometheus instance should scrape it. Two service monitors will be scraped by instance `prometheus-00`
and the other two will be scraped by `prometheus-01`.

```
kubectl apply -f servicemonitors.yaml
```

### Create the additional scrape configuration

This configuration is included here to illustrate how additional scrape configuration works. 
There is no `catalogue.sock-shop` service.

```
kubectl apply -f additional-scrape-config-secret.yaml
```

### Deploy Prometheus instance:

The alerting rule is simple alert that always fires. To deploy:

```
kubectl apply -f prometheus-rule.yaml
```

Deploy the Prometheus instance:

```
kubectl apply -f prometheus.yaml
```

Use `kubectl get pods` and `kubectl get prometheus` to see the Prometheus instance in the `default` namespace.


Port-forward to our Prometheus instance and see the scraped targets respectively:

```
kubectl port-forward svc/prometheus-operated 9090
```

### AlertManager

There is no webhook service as the configuration is to illustrate how `receiver`s work. To deploy:

```
kubectl apply -f alertmanager-config.yaml
```

The Prometheus Operator creates a `alertmanager-operated` endpoint object which Prometheus uses to match to an 
AlertManager. The `replicas: 3` is to ensure that the cluster status is up. To deploy:

```
kubectl apply -f alertmanager.yaml
```

Port-forward to our Prometheus instance and see the scraped targets respectively:

```
kubectl port-forward svc/alertmanager-operated 9093:9093
```

### Test in Grafana

Port forward to our Grafana instance to see our Prometheus metrics:

```
kubectl -n monitoring port-forward svc/grafana 3000:80
```

1. Navigate to `localhost:3000`
2. Login using username: `admin` and password: `admin`