# 🔧 ELK Stack on Podman — Full Tutorial

This tutorial walks you through running the ELK Stack (Elasticsearch, Logstash, Kibana) on **Podman**, including:

- Testing Logstash (file → file)
- Ingesting logs via Filebeat
- Outputting to Elasticsearch
- Viewing in Kibana

---

## 🧰 Prerequisites

Ensure:
- Podman is installed and running
- Allocate enough memory (≥ 2GB for Elasticsearch):

```bash
podman machine set --memory 4096
podman machine stop
podman machine start
```

---

## 🔹 Step 1: Pull Required Images

```bash
podman pull docker.elastic.co/elasticsearch/elasticsearch:8.13.2
podman pull docker.elastic.co/kibana/kibana:8.13.2
podman pull docker.elastic.co/logstash/logstash:8.13.2
```

---

## 🔹 Step 2: Create a Pod for Networking

```bash
podman pod create --name elk -p 9200:9200 -p 5601:5601 -p 5044:5044
```

---

## 🔹 Step 3: Run Elasticsearch

```bash
podman run -d --rm --name elasticsearch \
  --pod elk \
  -e "discovery.type=single-node" \
  -e "ES_JAVA_OPTS=-Xms1g -Xmx1g" \
  -e "xpack.security.enabled=false" \
  docker.elastic.co/elasticsearch/elasticsearch:8.13.2
```

Check status:

```bash
curl http://localhost:9200
```

Expected output includes `cluster_name`, `version`, etc.

---

## 🔹 Step 4: Run Kibana

```bash
podman run -d --rm --name kibana \
  --pod elk \
  -e "ELASTICSEARCH_HOSTS=http://localhost:9200" \
  docker.elastic.co/kibana/kibana:8.13.2
```

Access UI: http://localhost:5601

---

## 🔹 Step 5: Test Logstash (File → File)

### 1. Create Config File `logstash.conf`

```conf
input {
  file {
    path => "/data/input.log"
    start_position => "beginning"
    sincedb_path => "/dev/null"
  }
}
output {
  file {
    path => "/data/output.log"
  }
}
```

### 2. Prepare Input File

```bash
mkdir -p ./logstash-data
echo "hello logstash" > ./logstash-data/input.log
```

### 3. Run Logstash

```bash
podman run -d --rm --name logstash \
  --pod elk \
  -v ./logstash.conf:/usr/share/logstash/pipeline/logstash.conf:Z \
  -v ./logstash-data:/data:Z \
  docker.elastic.co/logstash/logstash:8.13.2
```

### 4. Verify Output

```bash
cat ./logstash-data/output.log
```

---

## 🔹 Step 6: Filebeat → Logstash → Elasticsearch

### 1. Install and Run Filebeat

Install Filebeat (Mac example):

```bash
brew install filebeat
```

Create config `filebeat.yml`:

```yaml
filebeat.inputs:
  - type: log
    paths:
      - ./filebeat-log/input.log

output.logstash:
  hosts: ["localhost:5044"]
```

Start logging:

```bash
mkdir -p ./filebeat-log
echo "filebeat test log" > ./filebeat-log/input.log
filebeat -e -c ./filebeat.yml
```

### 2. Update `logstash.conf` for Filebeat → Elasticsearch

```conf
input {
  beats {
    port => 5044
  }
}
output {
  elasticsearch {
    hosts => ["http://localhost:9200"]
    index => "filebeat-%{+YYYY.MM.dd}"
  }
}
```

Restart logstash:

```bash
podman restart logstash
```

---

## 🔍 Step 7: View Logs in Kibana

1. Open: http://localhost:5601
2. Go to **Stack Management → Index Patterns**
3. Click **"Create Index Pattern"**
4. Pattern: `filebeat-*`
5. Time Field: `@timestamp`
6. Go to **Discover** to see logs!

---

## 🧪 (Optional) Elasticsearch API Check

```bash
curl http://localhost:9200/filebeat-*/_search?pretty
```

---

## ✅ Summary

| Component       | Role                     | Port         |
|----------------|--------------------------|--------------|
| Elasticsearch  | Log Storage              | 9200         |
| Kibana         | Log Visualization        | 5601         |
| Logstash       | Processing Pipeline      | 5044         |
| Filebeat       | Log File Forwarder       | —            |

---

