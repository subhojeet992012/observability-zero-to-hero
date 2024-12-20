```markdown
# Observability Setup

## Run Elasticsearch as a Docker Container

```bash
docker run -d \
  --name elasticsearch \
  -e "discovery.type=single-node" \
  -e "xpack.security.enabled=false" \
  -e "network.host=0.0.0.0" \
  -p 9200:9200 \
  -p 9300:9300 \
  docker.elastic.co/elasticsearch/elasticsearch:8.16.1
```

---

## Run Kibana as a Docker Container

```bash
docker run -d \
  --name kibana \
  --network bridge \
  -e "ELASTICSEARCH_HOSTS=http://192.168.32.137:9200/" \
  -p 5601:5601 \
  docker.elastic.co/kibana/kibana:8.16.1
```

To test the output using Filebeat, run:
```bash
docker exec -it filebeat filebeat test output
```

---

## Run Filebeat

### Create `filebeat.yml`

```yaml
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /hostfs/var/log/auth.log
      - /hostfs/var/log/syslog
      - /hostfs/var/log/kern.log
      - /hostfs/var/log/lastlog
      - /hostfs/var/log/nginx/access.log
      - /hostfs/var/log/nginx/error.log
    scan_frequency: 10s

filebeat.config.modules:
  path: ${path.config}/modules.d/*.yml
  reload.enabled: false

output.elasticsearch:
  hosts: ["http://192.168.32.137:9200"]  # Update with your Elasticsearch host

setup.kibana:
  host: "http://192.168.32.137:5601"    # Update with your Kibana host
```

### Run Filebeat as a Docker Container

```bash
docker run -d \
  --name=filebeat \
  --user=root \
  --volume="$(pwd)/filebeat.yml:/usr/share/filebeat/filebeat.yml:ro" \
  --volume="/var/log:/hostfs/var/log:ro" \
  --volume="/var/lib/docker/containers:/hostfs/var/lib/docker/containers:ro" \
  --net=host \
  docker.elastic.co/beats/filebeat:8.10.2
```

---

## Install Metricbeat

### Create `metricbeat.yml` and change the permission to root for the file 'chown root:root metricbeat.yml'

```yaml
metricbeat.modules:
  - module: system
    period: 10s
    metricsets:
      - cpu
      - memory
      - network
      - diskio
      - filesystem
      - uptime
      - process
      - socket_summary
    processors:
      - drop_event.when.regexp:
          system.process.name: '.*docker.*'

output.elasticsearch:
  hosts: ["http://192.168.32.137:9200"]  # Replace with your Elasticsearch host

setup.kibana:
  host: "http://192.168.32.137:5601"  # Replace with your Kibana host

logging:
  level: info
  to_files: true
  files:
    path: /var/log/metricbeat
    name: metricbeat
    keepfiles: 7
    permissions: 0644
```

### Run Metricbeat as a Docker Container by using Root user

```bash
docker run -d \
  --name=metricbeat \
  --user=root \
  --volume="$(pwd)/metricbeat.yml:/usr/share/metricbeat/metricbeat.yml:ro" \
  --volume="/var/lib/docker:/var/lib/docker:ro" \
  --volume="/proc:/host/proc:ro" \
  --volume="/sys:/host/sys:ro" \
  --volume="/var/run/docker.sock:/var/run/docker.sock:ro" \
  --net=host \
  docker.elastic.co/beats/metricbeat:8.7.0
```

---

## Create Heartbeat Monitor

### Create `heartbeat.yml`

```yaml
heartbeat.monitors:
  - type: http
    urls: ["http://192.168.32.137"]  # Nginx should be running on this URL (adjust if needed)
    schedule: "@every 2s"  # Poll the service every 2 seconds
    timeout: 10s
    check.response.status: 200
    name: nginx
    tags: ["nginx", "web_server"]

output.elasticsearch:
  hosts: ["http://192.168.32.137:9200"]  # Update with your Elasticsearch host

setup.kibana:
  host: "http://192.168.32.137:5601"    # Update with your Kibana host
```

### Run Heartbeat as a Docker Container

```bash
docker run -d \
  --name=heartbeat \
  --user=root \
  --volume="$(pwd)/heartbeat.yml:/usr/share/heartbeat/heartbeat.yml:ro" \
  --volume="/var/log:/hostfs/var/log:ro" \
  --volume="/var/lib/docker/containers:/hostfs/var/lib/docker/containers:ro" \
  --net=host \
  docker.elastic.co/beats/heartbeat:8.10.2
```
```
