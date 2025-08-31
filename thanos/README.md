# Thanos

This example illustrates Thanos using Prometheus automatic sharding.

## Start up the application

1. Build the applications using docker compose

```
docker compose -f docker-compose.yaml build 
```

2. Start up Prometheus, Thanos and the applications

```
docker compose up -d
```

3. Create a new Prometheus Data Source and name it **ThanosQuery**
4. Point the URL to `http://thanos-query:19090`
5. Metrics from all applications should now appear in the Explore window