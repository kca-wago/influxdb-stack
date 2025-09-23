# Content <!-- omit in toc -->
- [Create TIG-Stag (Telegraf - InfluxDB - Grafana)](#create-tig-stag-telegraf---influxdb---grafana)
  - [Prepare File System](#prepare-file-system)
    - [Prepare Telegraf default configuration](#prepare-telegraf-default-configuration)
    - [Prepare InfluxDB3 token](#prepare-influxdb3-token)
    - [Verify Telegraf](#verify-telegraf)



# Create TIG-Stag (Telegraf - InfluxDB - Grafana)

## Prepare File System

Create a folder with the following files:

```ini
# Copyright 2025 InfluxData
# Author: Suyash Joshi (sjoshi@influxdata.com)

# InfluxDB Configuration
INFLUXDB_HTTP_PORT=8181             # for influxdb3 enterprise database, change this to port 8182
INFLUXDB_HOST=influxdb3-core        # for influxdb3 enterprise database, change this to "influxdb3-enterprise"
INFLUXDB_TOKEN=
INFLUXDB_BUCKET=local_system        # Your Database name
INFLUXDB_ORG=local_org              
INFLUXDB_NODE_ID=node0            

# Grafana Configuration
GRAFANA_PORT=3000
GRAFANA_ADMIN_USER=admin
GRAFANA_ADMIN_PASSWORD=admin

# Telegraf Configuration
TELEGRAF_COLLECTION_INTERVAL=5s
DOCKER_SOCKET=/var/run/docker.sock
```

```yaml
services:
  influxdb3-core:
    container_name: influxdb3-core
    image: influxdb:3-core
    ports:
      - 8181:8181
    command:
      - influxdb3
      - serve
      - --node-id=${INFLUXDB_NODE_ID}
      - --object-store=file
      - --data-dir=/var/lib/influxdb3
    volumes:
      - influxdb_data:/var/lib/influxdb3
    healthcheck:
      test: ["CMD-SHELL", "curl -f -H 'Authorization: Bearer ${INFLUXDB_TOKEN}' http://localhost:8181/health || exit 1"]
      interval: 30s
      timeout: 10s
      retries: 3
    restart: unless-stopped

  telegraf:
    container_name: telegraf
    image: telegraf
    depends_on:
      influxdb3-core:
        condition: service_healthy
    # Optionally, you can switch to depend on influxdb3-enterprise if using that
    environment:
      - INFLUXDB_HOST=${INFLUXDB_HOST}
      - INFLUXDB_TOKEN=${INFLUXDB_TOKEN}
      - INFLUXDB_ORG=${INFLUXDB_ORG}
      - INFLUXDB_BUCKET=${INFLUXDB_BUCKET}
      - TELEGRAF_COLLECTION_INTERVAL=${TELEGRAF_COLLECTION_INTERVAL}
      - HOSTNAME=telegraf
    volumes:
      - ./telegraf/telegraf.conf:/etc/telegraf/telegraf.conf
    restart: unless-stopped

  grafana:
    image: grafana/grafana
    ports:
      - "${GRAFANA_PORT}:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      - GF_SECURITY_ADMIN_USER=${GRAFANA_ADMIN_USER}
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_ADMIN_PASSWORD}
    depends_on:
      - influxdb3-core
    # Or switch to influxdb3-enterprise as needed
    restart: unless-stopped

volumes:
  influxdb_data:
  grafana_data:
```

Create the sub-folder *./telegraf* by
```sh
mkdir -p ./telegraf
```

### Prepare Telegraf default configuration

Create a default configuration:

```sh
docker run --rm telegraf telegraf config > ./telegraf/telegraf.conf
```

### Prepare InfluxDB3 token

```sh
docker compose up -d influxdb3-core influxdb3 create token --admin
```

Copy the created API token to the *.env* file.

``ìni
INFLUXDB_TOKEN=apiv3….
```

## Start remaining services

```sh
docker compose down
docker compose up -d
```

### Verify Telegraf

Show telegraf logs by

```sh
docker compose logs telegraf
```

Verify telegraf has created tables in InfluxDB

```sh
docker compose exec influxdb3-core influxdb3 query \
  "SHOW TABLES" --database local_system --token YOUR_TOKEN_STRING
```
