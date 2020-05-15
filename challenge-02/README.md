### Challenge 2 [Elastic Stack]:
[![forthebadge](https://forthebadge.com/images/badges/uses-badges.svg)](https://forthebadge.com)

:bangbang: Before starting it will be handy to run the following command to avoid [elasticsearch container memory issues](https://github.com/docker-library/elasticsearch/issues/111)
```bash
sudo sysctl -w vm.max_map_count=262144
```

#### 1- Nginx:
The new configuration file `ch2.conf` generates access and error logs whenever the site is visited, these logs are kept in a volume `logs_v` that is shared with filebeat.

on linux that volume content can be accessed using:
```bash
cat /var/lib/docker/volumes/volume_name/_data/files_name
```

Dockerfile
```Dockerfile
FROM nginx:alpine 
COPY ./logs/ch2.conf /etc/nginx/conf.d/
COPY ./website/index.html /usr/share/nginx/html/
```

ch2.conf
```
# /etc/nginx/conf.d/ch2.conf
server {
    listen 80;
    server_name localhost;

    #log files
    access_log /var/log/nginx/ch2.access.log main;
    error_log /var/log/nginx/ch2.error.log warn;

    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
    }     

    error_page 500 502 503 504 /50x.html;
    location = /50x.html {
        root /usr/share/nginx/html;
    }
}
```
---

#### 2- Filebeat:
Filebeat rule is to ship the logs to logstash

Dockerfile
```Dockerfile
FROM elastic/filebeat:7.7.0

COPY ./filebeat.yml /usr/share/filebeat/ 

USER root
```

filebeat.yml
```yaml
filebeat.config:
  modules:
    path: ${path.config}/modules.d/*.yml
    reload.enabled: false

output.logstash:
  enabled: true
  hosts: ["logstash:5044"]

# Debugging -- check if filebeats can locally store logs -- START

# output.file:
#   path: "/tmp/filebeat"
#   filename: filebeat

# Debugging END

filebeat.inputs:
- type: log 
  paths:
    - /var/log/nginx/*.log
```    

---

#### 3- Logstash:
Logstash receives the logs from filebeat, there is an [nginx module](https://www.elastic.co/guide/en/logstash/current/logstash-config-for-filebeat-modules.html) that helps in configuring logstash in our use case

Dockerfile
```Dockerfile
FROM logstash:7.7.0
COPY ./logstash.yml /usr/share/logstash/config/
COPY ./logstash.conf /usr/share/logstash/pipeline/

USER root
```

logstash.yml ==> /usr/share/logstash/config/logstash.yml
```yaml
http.host: "0.0.0.0"
xpack.monitoring.elasticsearch.hosts: [ "http://es01:9200" ]
```


logstash.conf ==> /usr/share/logstash/pipeline/logstash.conf
```
input {
  beats {
    port => 5044
    host => "0.0.0.0"
  }
}
filter {
  if [fileset][module] == "nginx" {
    if [fileset][name] == "access" {
      grok {
        match => { "message" => ["%{IPORHOST:[nginx][access][remote_ip]} - %{DATA:[nginx][access][user_name]} \[%{HTTPDATE:[nginx][access][time]}\] \"%{WORD:[nginx][access][method]} %{DATA:[nginx][access][url]} HTTP/%{NUMBER:[nginx][access][http_version]}\" %{NUMBER:[nginx][access][response_code]} %{NUMBER:[nginx][access][body_sent][bytes]} \"%{DATA:[nginx][access][referrer]}\" \"%{DATA:[nginx][access][agent]}\""] }
        remove_field => "message"
      }
      mutate {
        add_field => { "read_timestamp" => "%{@timestamp}" }
      }
      date {
        match => [ "[nginx][access][time]", "dd/MMM/YYYY:H:m:s Z" ]
        remove_field => "[nginx][access][time]"
      }
      useragent {
        source => "[nginx][access][agent]"
        target => "[nginx][access][user_agent]"
        remove_field => "[nginx][access][agent]"
      }
      geoip {
        source => "[nginx][access][remote_ip]"
        target => "[nginx][access][geoip]"
      }
    }
    else if [fileset][name] == "error" {
      grok {
        match => { "message" => ["%{DATA:[nginx][error][time]} \[%{DATA:[nginx][error][level]}\] %{NUMBER:[nginx][error][pid]}#%{NUMBER:[nginx][error][tid]}: (\*%{NUMBER:[nginx][error][connection_id]} )?%{GREEDYDATA:[nginx][error][message]}"] }
        remove_field => "message"
      }
      mutate {
        rename => { "@timestamp" => "read_timestamp" }
      }
      date {
        match => [ "[nginx][error][time]", "YYYY/MM/dd H:m:s" ]
        remove_field => "[nginx][error][time]"
      }
    }
  }
}
output {
  elasticsearch {
    hosts => [ "es01:9200" ]
    manage_template => false
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
  }
}
```

---

#### 4- Elasticsearch:

Dockerfile
```Dockerfile
FROM elasticsearch:7.7.0

COPY ./elasticsearch.yml /usr/share/elasticsearch/config/
```

elasticsearch.yml
```yaml
cluster.name: "docker-cluster"
network.host: 0.0.0.0
```

---

#### 5- Kibana:

Dockerfile
```Dockerfile
FROM kibana:7.7.0

COPY ./kibana.yml /usr/share/kibana/config/
```

kibana.yml
```yaml
server.name: kibana
server.host: "0"
elasticsearch.hosts: [ "http://es01:9200" ]
elasticsearch.username: elastic
elasticsearch.password: changeme
monitoring.ui.container.elasticsearch.enabled: true
```

---

#### 6- Docker-compose:
the elasticsearch + kibana part is taken from the [documentation](https://www.elastic.co/guide/en/elastic-stack-get-started/current/get-started-docker.html#get-started-docker) becase i've faced some troubles regarding elasticsearch container exiting automatically, this helped me fix the issue. 

```dockerfile
version: "3.8"
services:
  nginx:
    build: './nginx'
    ports:
    - "8000:80"
    volumes: 
      - logs_v:/var/log/nginx/

  filebeat:
    build: './filebeat'
    ports:
      - 5044:5044
    expose: 
      - 5044
    volumes:
      - type: volume 
        source: logs_v 
        target: /var/log/nginx
        read_only: true
    networks:
      - elastic

  logstash:
    build: './logstash'
    hostname: logstash
    networks:
      - elastic

  es01:
      build: './elasticsearch'
      container_name: es01
      hostname: elasticsearch
      environment:
        - node.name=es01
        - cluster.name=es-docker-cluster
        - discovery.seed_hosts=es02,es03
        - cluster.initial_master_nodes=es01,es02,es03
        - bootstrap.memory_lock=true
        - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
      ulimits:
        memlock:
          soft: -1
          hard: -1
      volumes:
        - data01:/usr/share/elasticsearch/data
      ports:
        - 9200:9200
      networks:
        - elastic

  es02:
    build: './elasticsearch'
    container_name: es02
    environment:
      - node.name=es02
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es03
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - data02:/usr/share/elasticsearch/data
    ports:
      - 9201:9201
    networks:
      - elastic

  es03:
    build: './elasticsearch'
    container_name: es03
    environment:
      - node.name=es03
      - cluster.name=es-docker-cluster
      - discovery.seed_hosts=es01,es02
      - cluster.initial_master_nodes=es01,es02,es03
      - bootstrap.memory_lock=true
      - "ES_JAVA_OPTS=-Xms512m -Xmx512m"
    ulimits:
      memlock:
        soft: -1
        hard: -1
    volumes:
      - data03:/usr/share/elasticsearch/data
    ports:
      - 9202:9202
    networks:
      - elastic

  kib01:
    build: './kibana'
    container_name: kib01
    ports:
      - 5601:5601
    environment:
      ELASTICSEARCH_URL: http://es01:9200
      ELASTICSEARCH_HOSTS: http://es01:9200
    networks:
      - elastic

networks:
  elastic:
    driver: bridge
 
volumes:
  logs_v:
  data01:
    driver: local
  data02:
    driver: local
  data03:
    driver: local
```

---

#### 7-Jenkinsfile:
```jenkinsfile
pipeline{
    agent any 
    stages{
        stage('RUN ELK'){
            steps{
                sh "docker-compose build --no-cache && docker-compose up -d"
            }
        }


    }
}
```