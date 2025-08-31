# Prometheus Basic Sharding

This application uses automatic sharding and rebalancing.

## Run the application

1. Build the applications using docker compose

```
docker compose -f docker-compose.yaml build
```

2. Start Prometheus and the applications

```
docker compose up -d
```

Navigate to `localhost:9090` and `localhost:9091` to access both Prometheus instances. 
Applications specific labels are in `__meta_dns_name`

