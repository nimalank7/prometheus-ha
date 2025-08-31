# Prometheus Sharding in Kubernetes using Prometheus Operator

Walkthrough covers manual and automatic sharding.

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

3. Check monitoring

```
kubectl -n monitoring get pods
kubectl -n monitoring get servicemonitors
```

Default Prometheus

4. Access Prometheus on `localhost:9090`

```
kubectl -n monitoring port-forward svc/prometheus-operated 9090
```

5. Access Grafana on `localhost:3000`

```
kubectl -n monitoring port-forward svc/grafana 3000:80
```

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

## Sharding Strategies

There are two main types of strategies to achieve sharding, Manual and Automated. </br>

**Manual Sharding**

Manual sharding is more robust, and you are in control. </br>
It involves manually creating a `Prometheus` instance or "shard" and then setting `ServiceMonitors` per instance to scrape.

**Automated Sharding**

Automated sharding is an automated sharding strategy where Prometheus will use a hasmhmod technique to automatically select `ServiceMonitors` per shard to scrape.

--- 

## Create a service account

Service account with RBAC permissions needs to be created to allow Prometheus instances to access service monitors and allow scraping of service endpoints:

```
kubectl apply -f serviceaccount.yaml
```

### Manual Sharding

Apply two Prometheus instances that are manually setup to target their own sets of `ServiceMonitor`'s to scrape:

```
kubectl apply -f prometheus-man-00.yaml
kubectl apply -f prometheus-man-01.yaml
```

Use `kubectl get pods` and `kubectl get prometheus` to see the Prometheus instances in the `default` namespace.

### Apply Service Monitors

There are 4 ServiceMonitor which correspond to our applications respectively. Each ServiceMonitor has an extra label 
called `prometheus-shard` which indicates which Prometheus instance should scrape it. Two service monitors will be scraped by instance `prometheus-00` 
and the other two will be scraped by `prometheus-01`.

```
kubectl apply -f servicemonitors.yaml
```

Port-forward to our Prometheus instances and see the scraped targets respectively

```
kubectl port-forward prometheus-prometheus-00-0 9090
kubectl port-forward prometheus-prometheus-01-0 9091:9090
```

### Automated Sharding

In larger clusters, it can become toil to manually set sharding labels on `ServiceMonitor` resources and manually balancing `Prometheus` instances.

The Prometheus-Operator supports automated sharding, which follows the technique outlined in our Sharding Introduction video. </br>

The Prometheus-Operator will automate everything we learned (and performed in docker). </br>
It will create a separate `StatefulSet` for each shard and use the `hashmod` technique, typically on the address field of the service endpoints.

All we need to do is use the `spec.shards` field. </br>
The operator will also set external labels for each replica or shard as `prometheus_replica` which you can view in the Prometheus configuration in the UI. </br>

Ensure that the ServiceMonitors are applied.

Apply our sharded `Prometheus` instance:

```
kubectl apply -f prometheus-auto.yaml
```

Checkout each of the automated Prometheus shards

```
kubectl port-forward prometheus-prometheus-0 9090
kubectl port-forward prometheus-prometheus-shard-1-0 9091:9090
```