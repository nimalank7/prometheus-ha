# Thanos in Kubernetes
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

### Create an S3 storage for Thanos

Deploy our test Minio instance:

```
kubectl apply -f minio.yaml
```

This will create an S3 storage for testing purposes and a `Job` that will create a bucket for our Thanos data.

### Create Thanos storage secret

Secret that Thanos sidecars will use to connect to Minio to store data in S3.

```
kubectl apply -f thanos-secret.yaml
```

### Deploy Prometheus Instances with Thanos sidecar:

Prometheus configurations have a Thanos sidecar configured. Apply two Prometheus instances that are manually setup to target their own sets of `ServiceMonitor`'s to scrape:

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

### Access Prometheus instances

Port-forward to our Prometheus instances and see the scraped targets respectively

```
kubectl port-forward prometheus-prometheus-00-0 9090
kubectl port-forward prometheus-prometheus-01-0 9091:9090
```

Thanos sidecar container is now running inside the Prometheus pod:

```
kubectl logs prometheus-prometheus-00-0 -c thanos-sidecar
```

### Deploying the Thanos Query Service

Thanos sidecars are now enabled on our `Prometheus` instances, which means the sidecars are shipping metrics data from each instance to our `minio` instance.

Deploy the query service:

```
kubectl apply -f thanos-query.yaml
```

### Test in Grafana

We can now verify that we can query Thanos in Grafana by using `port-forward` to access Grafana.

```
kubectl -n monitoring port-forward svc/grafana 3000:80
```

1. Navigate to `localhost:3000`
2. Login using username: `admin` and password: `admin`
3. Create a new Prometheus Data Source and name it **ThanosQuery**
4. Point the URL to `http://thanos-query.default.svc.cluster.local:19090`
5. Metrics from all applications should now appear in the Explore window