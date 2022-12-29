alertmanager的配置文件名为config.yml

```
global:
  resolve_timeout: 5m
route:
  group_by: ['instance']
  group_wait: 10s
  group_interval: 10s
  repeat_interval: 120m
  receiver: 'PrometheusAlert'
receivers:
- name: 'PrometheusAlert'
  webhook_configs:
  - url: 'http://192.168.1.215:8080/prometheusalert?type=dd&tpl=prometheus-dd'
```

prometheus配置有rules.yml prometheus.yml file_sd/

rules.yml

```
groups:

- name: Domain
  rules:
  - alert: 站点可用性
    expr: probe_success == 0
    for: 1m
    labels:
      alertype: domain
      severity: critical
    annotations:
      description: "{{ $labels.env }}_{{ $labels.name }}({{ $labels.project }})：站点无法访问\n> {{ $labels.instance }}"

  - alert: 站点15m可用性低于80%
    expr: sum_over_time(probe_success[15m])/count_over_time(probe_success[15m]) * 100 < 80
    for: 1m
    labels:
      alertype: domain
      severity: warning
    annotations:
      description: "{{ $labels.env }}_{{ $labels.name }}({{ $labels.project }})：站点15m可用性：{{ $value | humanize }}%\n> {{ $labels.instance }}"

  - alert: 站点状态异常
    expr: (probe_success == 0 and probe_http_status_code > 499) or probe_http_status_code == 0
    for: 1m
    labels:
      alertype: domain
      severity: warning
    annotations:
      description: "{{ $labels.env }}_{{ $labels.name }}({{ $labels.project }})：站点状态异常：{{ $value }}\n> {{ $labels.instance }}"

  - alert: 站点耗时过高
    expr: probe_duration_seconds > 3
    for: 2m
    labels:
      alertype: domain
      severity: warning
    annotations:
      description: "{{ $labels.env }}_{{ $labels.name }}({{ $labels.project }})：当前站点耗时：{{ $value | humanize }}s\n> {{ $labels.instance }}"

  - alert: SSL证书有效期
    expr: (probe_ssl_earliest_cert_expiry-time()) / 3600 / 24 < 15
    for: 2m
    labels:
      alertype: domain
      severity: warning
    annotations:
      description: "{{ $labels.env }}_{{ $labels.name }}({{ $labels.project }})：证书有效期剩余{{ $value | humanize }}天\n> {{ $labels.instance }}"
- name: mem-rule
  rules:
  - alert: "内存报警"
    expr: (node_memory_MemTotal_bytes - (node_memory_MemFree_bytes+node_memory_Buffers_bytes+node_memory_Cached_bytes )) / node_memory_MemTotal_bytes * 101 > 90
    for: 30s
    labels:
      status: Warning
      severity: warning
    annotations:
      summary: "服务名:{{$labels.alertname}} 内存报警"
      description: "{{ $labels.alertname }}  内存资源利用率大于 90%"
      value: "{{ $value }}"
- name: disk-rule
  rules:
  - alert: "硬盘报警"
    expr: (node_filesystem_size_bytes{fstype=~"xfs|ext4"} - node_filesystem_avail_bytes) / node_filesystem_size_bytes * 100 > 95
    for: 30s
    labels:
      severity: middle
      status: Warning
    annotations:
      summary: "服务名:{{$labels.alertname}} 硬盘报警"
      description: "{{ $labels.alertname }}  硬盘资源利用率大于 95%"
      value: "{{ $value }}"
- name: cpu-rule
  rules:
  - alert: "cpu告警"
    expr: 100 - (avg by(instance) (rate(node_cpu_seconds_total{mode="idle"}[2m])) * 100) > 90
    for: 1m
    labels:
      serverity: high
    annotations:
      summary: "服务名:{{$labels.alertname}} cpu报警"
      description: "{{ $labels.alertname }}  cpu资源利用率大于 90%"
- name: node-rule
  rules:
  - alert: "node-down"
    expr: up{job="node_exporter"} == 0
    for: 1m
    labels:
      serverity: high
      team: node
    annotations:
      summary: "服务名:{{$labels.alertname}} node报警"
      description: "{{ $labels.alertname }} Instance  has been down for more than 1 minutes"
  - alert: K8S节点POD磁盘使用率
    expr: 100 - (node_filesystem_avail_bytes/node_filesystem_size_bytes{mountpoint=~"/data/var/lib/docker/containers/.*/mounts/.*"} * 100) > 85
    for: 5m
    labels:
      alertype: system
      severity: warning
    annotations:
      description: "{{ $labels.name }}_{{ $labels.mountpoint }}：磁盘使用率{{ $value | humanize }}%\n> {{ $labels.group }}-{{ $labels.instance }}"

  - alert: NFS磁盘使用率
    expr: 100 - (node_filesystem_avail_bytes/node_filesystem_size_bytes{fstype="nfs"} * 100) > 90
    for: 5m
    labels:
      alertype: system
      severity: warning
    annotations:
      description: "{{ $labels.name }}_{{ $labels.mountpoint }}：磁盘使用率{{ $value | humanize }}%\n> {{ $labels.group }}-{{ $labels.instance }}"

  - alert: 磁盘读写容量
    expr: (irate(node_disk_read_bytes_total[5m]) ) /1024 /1024  > 80 or (irate(node_disk_written_bytes_total[5m]) ) /1024 /1024 > 80
    for: 8m
    labels:
      alertype: disk
      severity: warning
    annotations:
      description: "{{ $labels.name }}_{{ $labels.device }}：当前IO为{{ $value | humanize }}MB/s\n> {{ $labels.group }}-{{ $labels.instance }}"

  - alert: 网络流入（下载）数据过多
    expr: sum by(device,instance, name, group, account) (irate(node_network_receive_bytes_total{device!~'tap.*|veth.*|br.*|docker.*|virbr.*|lo.*|cni.*'}[5m])) / 1024 / 1024 > 70
    for: 5m
    labels:
      alertype: network
      severity: warning
    annotations:
      description: "{{ $labels.name }}：流入数据为{{ $value | humanize }}MB/s\n> {{ $labels.group }}-{{ $labels.instance }}"

  - alert: 网络流出（上传）数据过多
    expr: sum by(device,instance, name, group, account) (irate(node_network_transmit_bytes_total{device!~'tap.*|veth.*|br.*|docker.*|virbr.*|lo.*|cni.*'}[5m])) / 1024 / 1024 > 70
    for: 5m
    labels:
      alertype: network
      severity: warning
    annotations:
      description: "{{ $labels.name }}：流出数据为{{ $value | humanize }}MB/s\n> {{ $labels.group }}-{{ $labels.instance }}"
  - alert: PodDown # 15s 采集 + 15s 扫描规则，规则是1分钟前存在 pod 为 not ready 的 pod，15s 扫描一次的间隔，至少能扫描 3次，所以一定会发送
    expr: min_over_time(kube_pod_container_status_ready{pod!~".*job.*|backup.*|minio-backup.*|clean-docker|dynamic-index-patterns|node-shell.*|.*jenkins.*|elasticsearch-logging-curator-elasticsearch-curator.*|jaeger-es-index-cleaner.*|ks-alerting-migration.*|fluentbit-operator-migrator.*"} [1m])== 0   
    for: 1m # 持续多久确认报警信息
    annotations:
      summary: 'Container: {{ $labels.container }} down'
      description: 'Cluster: {{ $labels.cluster }} Namespace: {{ $labels.namespace }}, Pod: {{ $labels.pod }} 服务不正常'
  - alert: Pod的cpu告警 
    expr: 100* (sum(rate(container_cpu_usage_seconds_total[5m])) by (cluster,namespace,pod,container) /sum(kube_pod_container_resource_limits_cpu_cores) by (cluster,namespace,pod,container))>90
    for: 1m # 持续多久确认报警信息
    annotations:
      summary: 'Container: {{ $labels.container }} '
      description: 'Cluster: {{ $labels.cluster }} Namespace: {{ $labels.namespace }}, Pod: {{ $labels.pod }} 的cpu使用率超过90%'
  - alert: Pod的memory告警 
    expr: 100 * (sum by(cluster, namespace, pod,container)(container_memory_working_set_bytes) / sum by(cluster, namespace, pod,container) (kube_pod_container_resource_limits_memory_bytes)) >90
    for: 1m # 持续多久确认报警信息
    annotations:
      summary: 'Container: {{ $labels.container }} '
      description: 'Cluster: {{ $labels.cluster }} Namespace: {{ $labels.namespace }}, Pod: {{ $labels.pod }} 的memory使用率超过90%'
```

prometheus.yml

```
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
  - static_configs:
    - targets:
      - 192.168.1.215:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  - "rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  - job_name: zijianzhuji
    scrape_interval: 15s
    scrape_timeout: 5s
    consul_sd_configs:
      - server: '192.168.1.215:8500'
        token: '9ff8c572-a78c-d246-7196-38f9716e997a'
        refresh_interval: 30s
        services: []
        tags: ['linux']
    relabel_configs:
      - source_labels: ['__meta_consul_service_metadata_vendor']
        target_label: vendor
      - source_labels: ['__meta_consul_service_metadata_region']
        target_label: region
      - source_labels: ['__meta_consul_service_metadata_group']
        target_label: group
      - source_labels: ['__meta_consul_service_metadata_account']
        target_label: account
      - source_labels: ['__meta_consul_service_metadata_name']
        target_label: name
      - source_labels: ['__meta_consul_service_metadata_iid']
        target_label: iid
      - source_labels: ['__meta_consul_service_metadata_exp']
        target_label: exp
      - source_labels: ['__meta_consul_service_metadata_instance']
        target_label: instance
      - source_labels: [instance]
        target_label: __address__
  - job_name: 'blackbox_exporter'
    metrics_path: /probe
    consul_sd_configs:
      - server: '192.168.1.215:8500'
        token: '9ff8c572-a78c-d246-7196-38f9716e997a'
        services: ['blackbox_exporter']
    relabel_configs:
      - source_labels: ["__meta_consul_service_metadata_instance"]
        target_label: __param_target
      - source_labels: [__meta_consul_service_metadata_module]
        target_label: __param_module
      - source_labels: [__meta_consul_service_metadata_module]
        target_label: module
      - source_labels: ["__meta_consul_service_metadata_company"]
        target_label: company
      - source_labels: ["__meta_consul_service_metadata_env"]
        target_label: env
      - source_labels: ["__meta_consul_service_metadata_name"]
        target_label: name
      - source_labels: ["__meta_consul_service_metadata_project"]
        target_label: project
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: 192.168.1.215:9115

  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
    - targets: ['localhost:9090']
  - job_name: 'federate'
    scrape_interval: 15s
    honor_labels: true
    metrics_path: '/federate'
    params:
      'match[]':
        - '{job="node-exporter-hwcloud"}'
        - '{job="node-exporter"}'
        - '{job="kube-state-metrics"}'
        - '{cluster=~".*"}'
        - '{container=~".*"}'
    file_sd_configs:
    - files:
      - file_sd/federate.yaml
      refresh_interval: 2m 
  - job_name: 'linux'
    file_sd_configs:
    - files:
      - file_sd/nodes-*.yaml
      refresh_interval: 2m 
  - file_sd_configs:
    - files:
      - file_sd/mysql.yml
    job_name: MySQL
    metrics_path: /metrics
    relabel_configs:
    - source_labels: [__address__]
      regex: (.*)
      target_label: __address__
      replacement: $1
```

federate.yaml

```
- targets:
  - '119.3.83.171:30269'
  - '139.198.122.166:31127'
```





prometheus crd中可以添加

```
externalLabels:
  cluster: 青云qke
remoteWrite:
  - url: 'http://122.112.254.39:19291/api/v1/receive'    
```

