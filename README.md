# ELK

## Подготовка. Создание необходимых директорий для ролей. Определение драйверов и сценариев

```shell
netology@netology:~/Projects/elk/roles$ ansible-galaxy role init elasticsearch
- Role elasticsearch was created successfully
netology@netology:~/Projects/elk/roles$ ansible-galaxy role init kibana
- Role kibana was created successfully
netology@netology:~/Projects/elk/roles$ ansible-galaxy role init logstash
- Role logstash was created successfully
netology@netology:~/Projects/elk/roles$ ansible-galaxy role init filebeat
- Role filebeat was created successfully
```

### Elasticsearch

```shell
netology@netology:~/Projects/elk/roles$ cd elasticsearch/
netology@netology:~/Projects/elk/roles/elasticsearch$ molecule init scenario --driver-name docker 
INFO     Initializing new scenario default...
INFO     Initialized scenario in /home/netology/Projects/elk/roles/elasticsearch/molecule/default successfully.
```
--------------------------------------------------------------

* `molecule.yml`:

```yaml
---
dependency:
  name: galaxy
driver:
  name: docker
lint: |
  ansible-lint .
  yamllint .
platforms:
  - name: es-hot
    image: docker.elastic.co/elasticsearch/elasticsearch:7.16.3
    pre_build_image: true
    volumes:
      - data01:/usr/share/elasticsearch/data
    # ulimits:
    #   memlock:
    #     soft: "-1"
    #     hard: "-1"
    #   nofile:
    #     soft: "65536"
    #     hard: "65536"
    ports:
      - 9200:9200
    network:
      - elastic
    depends_on:
      - es-warm
    groups:
      - elk
  - name: es-warm
    image: docker.elastic.co/elasticsearch/elasticsearch:7.16.3
    pre_build_image: true
    volumes:
      - data02:/usr/share/elasticsearch/data
    # ulimits:
    #   memlock:
    #     soft: "-1"
    #     hard: "-1"
    #   nofile:
    #     soft: "65536"
    #     hard: "65536"
    network:
      - elastic
    groups:
      - elk
provisioner:
  name: ansible
  log: true
  options:
    vvv: true
    diff: true
  inventory:
    host_vars:
      es-hot:
        elk_id: 0
      es-warm:
        elk_id: 1

verifier:
  name: ansible

```
--------------------------------------------------------

* `install.yml`:

```yaml
---
- name: Check for Python
  raw: test -e /usr/bin/python
  changed_when: false
  failed_when: false
  register: check_python
- name: Install Python
  raw: apt -y update && apt install -y python
  when: check_python.rc != 0
- name: Update repositories cache and install "python" package
  apt:
    update_cache: yes
```
----------------------------------------------------------

* `configure.yml`:

```yaml
---
- name: Configure Elasticsearch
  template:
    src: elasticsearch.yml.j2
    mode: 0644
    dest: /usr/share/elasticsearch/config/elasticsearch.yml
```
----------------------------------------------------------------------

* `elasticsearch.yml.j2`:
```yaml
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
network.host: 0.0.0.0
cluster.initial_master_nodes: es-hot,es-warm
bootstrap.memory_lock: true
cluster.name: es-docker-cluster
"ES_JAVA_OPTS=-Xms512m -Xmx512m"
{% if inventory_hostname == 'es-hot' %}
node.name: es-hot
discovery.seed_hosts: es-warm
{% elif inventory_hostname == 'es-warm' %}
node.name: es-warm
discovery.seed_hosts: es-hot
{% endif %}
```
----------------------------------------------------------------------

* `main.yml`:

```yaml
---
- include_tasks: "install.yml"
- import_tasks: configure.yml
```

* [Лог](logs/01.md) запуска `molecule test`
* [Лог](logs/02.md) запуска `molecule converge`

------------------------------------------------------------------------

### Kibana

```shell
netology@netology:~/Projects/elk/roles/kibana$ molecule init scenario --driver-name docker 
INFO     Initializing new scenario default...
INFO     Initialized scenario in /home/netology/Projects/elk/roles/kibana/molecule/default successfully.
```
* `molecule.yml`:

```yaml
---
dependency:
  name: galaxy
driver:
  name: docker
lint: |
  ansible-lint .
  yamllint .
platforms:
  - name: kibana
    image: docker.elastic.co/kibana/kibana:7.16.3
    pre_build_image: true
    ports:
      - 5601:5601
    network:
      - elastic
    depends_on:
      - es-hot
      - es-warm
    groups:
      - elk
provisioner:
  name: ansible
  log: true
  options:
    vvv: true
    diff: true
  inventory:
    host_vars:
      kibana:
        elk_id: 2
verifier:
  name: ansible
```
---------------------------------------------------------------------------

* `tasks/main.yml`:

```yaml
---
- name: Configure Kibana
  template:
    src: kibana.yml.j2
    mode: 0644
    dest: /usr/share/kibana/config/kibana.yml
```
---------------------------------------------------------------------------

* `kibana.yml.j2`:

```yaml
server.host: "0.0.0.0"
server.shutdownTimeout: "5s"
monitoring.ui.container.elasticsearch.enabled: true
elasticsearch.url: "http://es-hot:9200"
elasticsearch.hosts: ["http://es-hot:9200","http://es-warm:9200"]
```
---------------------------------------------------------------------------

* [Лог](logs/03.md) запуска `molecule test`
* [Лог](logs/04.md) запуска `molecule converge`

------------------------------------------------------------------------

### Logstash

```shell
netology@netology:~/Projects/elk/roles/logstash$ molecule init scenario --driver-name docker 
INFO     Initializing new scenario default...
INFO     Initialized scenario in /home/netology/Projects/elk/roles/logstash/molecule/default successfully.
```

---------------------------------------------------------------------------

* `molecule.yml`:

```yaml
---
dependency:
  name: galaxy
driver:
  name: docker
lint: |
  ansible-lint .
  yamllint .
platforms:
  - name: logstash
    image: docker.elastic.co/logstash/logstash:7.16.3
    pre_build_image: true
    ports:
      - 5046:5046
    network:
      - elastic
    depends_on:
      - es-hot
      - es-warm
    groups:
      - elk
provisioner:
  name: ansible
  log: true
  options:
    vvv: true
    diff: true
  inventory:
    host_vars:
      logstash:
        elk_id: 3
verifier:
  name: ansible
```

---------------------------------------------------------------------------

* `tasks/main.yml`:

```yaml
---
- name: Configure jvm
  template:
    src: jvm.options.j2
    mode: 0755
    dest: /usr/share/logstash/config/jvm.options
- name: Configure Logstash
  template:
    src: logstash.conf.j2
    mode: 0755
    dest: /usr/share/logstash/config/logstash.conf
- name: Configure with yml Logstash
  template:
    src: logstash.yml.j2
    mode: 0755
    dest: /usr/share/logstash/config/logstash.yml
```
---------------------------------------------------------------------------

* `jvm.options.j2`:

```shell
## G1GC Configuration

10-:-XX:+UseG1GC
10-13:-XX:-UseConcMarkSweepGC
10-13:-XX:-UseCMSInitiatingOccupancyOnly
10-:-XX:G1ReservePercent=25
10-:-XX:InitiatingHeapOccupancyPercent=30

## JVM temporary directory
-Djava.io.tmpdir=${ES_TMPDIR}

## heap dumps

# generate a heap dump when an allocation from the Java heap fails
# heap dumps are created in the working directory of the JVM
-XX:+HeapDumpOnOutOfMemoryError

# specify an alternative path for heap dumps; ensure the directory exists and
# has sufficient space
-XX:HeapDumpPath=data

# specify an alternative path for JVM fatal error logs
-XX:ErrorFile=logs/hs_err_pid%p.log

# JDK 9+ GC logging
8-9:-Xmx2g
# 9-:-Xlog:gc*,gc+age=trace,safepoint:file=logs/gc.log:utctime,pid,tags:filecount=32,filesize=64m
```
---------------------------------------------------------------------------

* `logstash.conf.j2`:

```shell
input {
  beats {
    port => 5044
  }
}
filter {
 if [type] == "nginx_access" {
    grok {
        match => { "message" => "%{IPORHOST:remote_ip} - %{DATA:user} \[%{HTTPDATE:access_time}\] \"%{WORD:http_method} %{DATA:url} HTTP/%{NUMBER:http_version}\" %{NUMBER:response_code} %{NUMBER:body_sent_bytes} \"%{DATA:referrer}\" \"%{DATA:agent}\"" }
    }
  }
  date {
        match => [ "timestamp" , "dd/MMM/YYYY:HH:mm:ss Z" ]
  }
  geoip {
     database => "/etc/logstash/GeoLite2-City.mmdb"
     source => "remote_ip"
     target => "geoip"
     add_tag => [ "nginx-geoip" ]
    }
}
output {
  elasticsearch {
    hosts => ["http://es-hot:9200"]
    index => "%{[@metadata][beat]}-%{[@metadata][version]}-%{+YYYY.MM.dd}"
    #user => "elastic"
    #password => "changeme"
  }
}
```
---------------------------------------------------------------------------

* `logstash.yml.j2`:

```yaml
pipeline.id: netology
pipeline.workers: 1
pipeline.batch.size: 1
path.settings: /etc/logstash
path.config: /etc/logstash
http.host: "0.0.0.0"
```
---------------------------------------------------------------------------

* [Лог](logs/05.md) запуска `molecule test`
* [Лог](logs/06.md) запуска `molecule converge`

---------------------------------------------------------------------------

#### Filebeat

```shell
netology@netology:~/Projects/elk/roles/filebeat$ molecule init scenario --driver-name docker
INFO     Initializing new scenario default...
INFO     Initialized scenario in /home/netology/Projects/elk/roles/filebeat/molecule/default successfully.
```

* `molecule.yml`:
```yaml
---
dependency:
  name: galaxy
driver:
  name: docker
lint: |
  ansible-lint .
  yamllint .
platforms:
  - name: filebeat
    image: docker.elastic.co/beats/filebeat:7.16.3
    privileged: true
    user: root
    pre_build_image: true
    volumes:
      - /var/lib/docker:/var/lib/docker
      - /var/run/docker.sock:/var/run/docker.sock
    network:
      - elastic
    depends_on:
      - logstash
    groups:
      - elk
provisioner:
  name: ansible
  log: true
  options:
    vvv: true
    diff: true
  inventory:
    host_vars:
      filebeat:
        elk_id: 4
verifier:
  name: ansible
```

* `tasks/main.yml`:
```yaml
---
- name: Configure Filebeat
  template:
    src: filebeat.yml.j2
    dest: /usr/share/filebeat/filebeat.yml
    mode: 0644
```

* `filebeat.yml.j2`:
```shell
filebeat.inputs:
  - type: container
    paths:
      - '/var/lib/docker/containers/*/*.log'

processors:
  - add_docker_metadata:
      host: "unix:///var/run/docker.sock"

  - decode_json_fields:
      fields: ["message"]
      target: "json"
      overwrite_keys: true

output.logstash:
  hosts: ["logstash:5046"]

#output.console:
#  enabled: true

logging.json: true
logging.metrics.enabled: false
```

* [Лог](logs/07.md) запуска `molecule test`
* [Лог](logs/08.md) запуска `molecule converge`

------------------------------------------------------------------------------------

```shell
netology@netology:~/Projects/elk$ ansible-galaxy collection install community.docker
Starting galaxy collection install process
Process install dependency map
Starting collection install process
Downloading https://galaxy.ansible.com/download/community-docker-2.1.1.tar.gz to /home/netology/.ansible/tmp/ansible-local-13935p61p6vf2/tmprffesofl/community-docker-2.1.1-7t5x5f7o
Installing 'community.docker:2.1.1' to '/home/netology/.ansible/collections/ansible_collections/community/docker'
community.docker:2.1.1 was installed successfully
```