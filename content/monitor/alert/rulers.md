+++
title = 'Rulers'
date = 2024-03-26T11:27:25+08:00
draft = true
+++

访问：[告警规则集合链接](https://samber.github.io/awesome-prometheus-alerts/)

#### Node Exporter

---

```yaml
# 主机和硬件相关的告警规则
groups:
# 磁盘使用率告警
  - name: 磁盘使用率告警
    rules:
      - alert: 磁盘使用率告警
        expr: floor(((node_filesystem_size_bytes{heihutao!~"heihutao-tw-dev|heihutao-tw-test|heihutao-tw-dp"} - node_filesystem_free_bytes) * 100) / (node_filesystem_avail_bytes + node_filesystem_size_bytes - node_filesystem_free_bytes)) > 85
        for: 1m
        labels:
          severity: 严重
        annotations:
          summary: "磁盘使用率超过85% (实例 {{ $labels.instance }})"
          description: "设备 {{ $labels.device }} 挂载在 {{ $labels.mountpoint }} 的磁盘使用率超过了 85%。(当前值: {{ $value }}%)"

# 内存使用率告警
  - name: 内存告警规则
    rules:
      - alert: "内存使用率告警"
        expr: floor((node_memory_MemTotal_bytes{heihutao!~"heihutao-tw-dev|heihutao-tw-test|heihutao-tw-dp"} - (node_memory_MemFree_bytes + node_memory_Buffers_bytes + node_memory_Cached_bytes)) / node_memory_MemTotal_bytes * 100) > 85
        for: 3m
        labels:
          severity: 严重
        annotations:
          summary: "内存使用率超过85% (实例 {{ $labels.instance }})"
          description: "内存资源利用率大于85%! (当前值: {{ $value }}%)"

# CPU使用率告警
  - name: CPU报警规则
    rules:
      - alert: CPU使用率告警
        expr: floor(100 * (1 - avg(irate(node_cpu_seconds_total{mode="idle",heihutao!~"heihutao-tw-dev|heihutao-tw-test|heihutao-tw-dp"}[3m])) by (instance, datacenter))) > 85
        for: 3m
        labels:
          severity: 严重
        annotations:
          summary: "CPU报警使用率超过85% (实例  {{ $labels.instance }})"
          description: "CPU使用率超过了85%! (当前值: {{ $value }}%)"

#磁盘几乎耗尽了可用 inode（剩余 < 10%）
  - name: 磁盘inode告警
    rules:
      - alert: "磁盘inode告警"
        expr: (node_filesystem_files_free{fstype!="msdosfs"} / node_filesystem_files{fstype!="msdosfs"} * 100 < 10 and ON (instance, device, mountpoint) node_filesystem_readonly == 0) * on(instance) group_left (nodename) node_uname_info{nodename=~".+"}
        for: 2m
        labels:
          severity: warning
        annotations:
            summary: "磁盘inode告警 (实例 {{ $labels.instance }})"
            description: "设备 {{ $labels.device }} 挂载在 {{ $labels.mountpoint }} 的磁盘几乎耗尽了可用 inode（剩余 < 10%）。(当前值: {{ $value }}%)"

# 主机离线
  - name: 主机离线
    rules:
      - alert: "主机离线"
        expr: up == 0
        for: 1m
        labels:
          severity: 严重
        annotations:
          summary: "主机离线 (实例 {{ $labels.instance }})"
          description: "主机 {{ $labels.instance }} 离线了"
```





#### mongodb Exporter

```yaml
groups:
  - name: MongoDB 宕机
    rules:
      - alert: MongoDB 宕机
        expr: mongodb_up == 0
        for: 1m
        labels:
          severity: 严重
        annotations:
          summary: "MongoDB 宕机 (实例 {{ $labels.instance }})"
          description: "MongoDB 实例 {{ $labels.instance }} 宕机。"
```



#### ETCD Exporter

```yaml
groups:
#etcd集群没有领导者
- name: etcd 集群没有领导者
  rules:
    - alert: etcd 集群没有领导者
      expr: etcd_server_has_leader == 0
      for: 1m
      labels:
        severity: 严重
      annotations:
        summary: "etcd 集群没有领导者 (实例 {{ $labels.instance }})"
        description: "etcd 集群没有领导者。"

# 在 Etcd 中检测到超过 5% 的 HTTP 故障
- name: Etcd 中检测到超过 5% 的 HTTP 故障
  rules:
    - alert: Etcd 中检测到超过 5% 的 HTTP 故障
      expr: sum(rate(etcd_http_failed_total[1m])) BY (method) / sum(rate(etcd_http_received_total[1m])) BY (method) > 0.05
      for: 1m
      labels:
        severity: 严重
      annotations:
        summary: "Etcd 中检测到超过 5% 的 HTTP 故障 (实例 {{ $labels.instance }})"
        description: "在 Etcd 中检测到超过 5% 的 HTTP 故障。"

# Etcd成员通信变慢，第99个百分位数超过0.15秒
- name: Etcd成员通信变慢
  rules:
    - alert: Etcd成员通信变慢
      expr: histogram_quantile(0.99, rate(etcd_network_peer_round_trip_time_seconds_bucket[1m])) > 0.15
      for: 1m
      labels:
        severity: 严重
      annotations:
        summary: "Etcd成员通信变慢 (实例 {{ $labels.instance }})"
        description: "Etcd成员通信变慢，第99个百分位数超过0.15秒。"

# Etcd 服务器过去一小时收到超过 5 个失败的提案
- name: Etcd 服务器过去一小时收到超过 5 个失败的提案
  rules:
    - alert: Etcd 服务器过去一小时收到超过 5 个失败的提案
      expr: increase(etcd_server_proposals_failed_total[1h]) > 5
      for: 1m
      labels:
        severity: 严重
      annotations:
        summary: "Etcd 服务器过去一小时收到超过 5 个失败的提案 (实例 {{ $labels.instance }})"
        description: "Etcd 服务器过去一小时收到超过 5 个失败的提案。"

# Etcd WAL fsync 持续时间增加，第 99 个百分位数超过 0.5 秒[复制]
- name: Etcd WAL fsync 持续时间增加
  rules:
    - alert: Etcd WAL fsync 持续时间增加
      expr: histogram_quantile(0.99, rate(etcd_disk_wal_fsync_duration_seconds_bucket[1m])) > 0.5
      for: 1m
      labels:
        severity: 严重
      annotations:
        summary: "Etcd WAL fsync 持续时间增加 (实例 {{ $labels.instance }})"
        description: "Etcd WAL fsync 持续时间增加，第 99 个百分位数超过 0.5 秒。"

# Etcd 提交持续时间增加，第 99 个百分位超过 0.25 秒
- name: Etcd 提交持续时间增加
  rules:
    - alert: Etcd 提交持续时间增加
      expr: histogram_quantile(0.99, rate(etcd_disk_backend_commit_duration_seconds_bucket[1m])) > 0.25
      for: 1m
      labels:
        severity: 严重
      annotations:
        summary: "Etcd 提交持续时间增加 (实例 {{ $labels.instance }})"
        description: "Etcd 提交持续时间增加，第 99 个百分位超过 0.25 秒。"

# HTTP 请求变慢，第 99 个百分位数超过 0.15 秒
- name: HTTP 请求变慢
  rules:
  - alert: HTTP 请求变慢
    expr: histogram_quantile(0.99, rate(etcd_http_successful_duration_seconds_bucket[1m])) > 0.15
    for: 1m
    labels:
      severity: 严重
    annotations:
      summary: "HTTP 请求变慢 (实例 {{ $labels.instance }})"
      description: "HTTP 请求变慢，第 99 个百分位数超过 0.15 秒。"

# Etcd 中检测到超过 1% 的 HTTP 故障
- name: Etcd 中检测到超过 1% 的 HTTP 故障
  rules:
  - alert: Etcd 中检测到超过 1% 的 HTTP 故障
    expr: sum(rate(etcd_http_failed_total[1m])) BY (method) / sum(rate(etcd_http_received_total[1m])) BY (method) > 0.01
    for: 1m
    labels:
      severity: 严重
    annotations:
      summary: "Etcd 中检测到超过 1% 的 HTTP 故障 (实例 {{ $labels.instance }})"
      description: "在 Etcd 中检测到超过 1% 的 HTTP 故障。"

# Etcd 中检测到超过 5% 的 GRPC 请求失败
- name: Etcd 中检测到超过 5% 的 GRPC 请求失败
  rules:
    - alert: Etcd 中检测到超过 5% 的 GRPC 请求失败
      expr: sum(rate(grpc_server_handled_total{grpc_code!="OK"}[1m])) BY (grpc_service, grpc_method) / sum(rate(grpc_server_handled_total[1m])) BY (grpc_service, grpc_method) > 0.05
      for: 1m
      labels:
        severity: 严重
      annotations:
        summary: "Etcd 中检测到超过 5% 的 GRPC 请求失败 (实例 {{ $labels.instance }})"
        description: "在 Etcd 中检测到超过 5% 的 GRPC 请求失败。"

# Etcd 中检测到超过 1% 的 GRPC 请求失败
- name: Etcd 中检测到超过 1% 的 GRPC 请求失败
  rules:
    - alert: Etcd 中检测到超过 1% 的 GRPC 请求失败
      expr: sum(rate(grpc_server_handled_total{grpc_code!="OK"}[1m])) BY (grpc_service, grpc_method) / sum(rate(grpc_server_handled_total[1m])) BY (grpc_service, grpc_method) > 0.01
      for: 1m
      labels:
        severity: 严重
      annotations:
        summary: "Etcd 中检测到超过 1% 的 GRPC 请求失败 (实例 {{ $labels.instance }})"
        description: "在 Etcd 中检测到超过 1% 的 GRPC 请求失败。"

# Etcd 领导者在 10 分钟内更换了 2 次以上
- name: Etcd 领导者在 10 分钟内更换了 2 次以上
  rules:
    - alert: Etcd 领导者在 10 分钟内更换了 2 次以上
      expr: increase(etcd_server_leader_changes_seen_total[10m]) > 2
      for: 1m
      labels:
        severity: 严重
      annotations:
        summary: "Etcd 领导者在 10 分钟内更换了 2 次以上 (实例 {{ $labels.instance }})"
        description: "Etcd 领导者在 10 分钟内更换了 2 次以上。"

```



#### Elasticsearch

```yaml
groups:
#  Elasticsearch 堆使用率过高
- name: Elasticsearch 堆使用率过高
  rules:
  - alert: Elasticsearch 堆使用率过高
    expr: (elasticsearch_jvm_memory_used_bytes{area="heap"} / elasticsearch_jvm_memory_max_bytes{area="heap"}) * 100 > 85
    for: 1m
    labels:
      severity: 严重
    annotations:
      summary: "Elasticsearch 堆使用率过高 (实例 {{ $labels.instance }})"
      description: "Elasticsearch 堆使用率超过了 85%。(当前值: {{ $value }}%)"

#  Elasticsearch 磁盘空间不足
- name: Elasticsearch 磁盘空间不足
  rules:
  - alert: Elasticsearch 磁盘空间不足
    expr: elasticsearch_filesystem_data_available_bytes / elasticsearch_filesystem_data_size_bytes * 100 < 10
    for: 1m
    labels:
      severity: 严重
    annotations:
      summary: "Elasticsearch 磁盘空间不足 (实例 {{ $labels.instance }})"
      description: "Elasticsearch 磁盘空间不足。(当前值: {{ $value }}%)"

#  elasticsearch_cluster_health_status{color="red"}
- name: Elasticsearch 集群健康状态
  rules:
  - alert: Elasticsearch 集群健康状态
    expr: elasticsearch_cluster_health_status{color="red"}
    for: 1m
    labels:
      severity: 严重
    annotations:
      summary: "Elasticsearch 集群健康状态 (实例 {{ $labels.instance }})"
      description: "Elasticsearch 集群健康状态为 red。"
```



#### Kubernetes

```yaml
# Kubernetes： kube-state-metrics
groups:
  - name: 节点已长时间未就绪
    rules:
      - alert: 节点已长时间未就绪
        expr: kube_node_status_condition{condition="Ready",status="false"} == 1
        for: 5m
        labels:
          severity: 严重
        annotations:
          summary: "节点已长时间未就绪 (实例 {{ $labels.node }})"
          description: "节点 {{ $labels.node }} 已长时间未就绪。"

  - name: Kubernetes Pod OOMKilled
    rules:
      - alert: Kubernetes 容器OOMKilled
        expr: sum(kube_pod_container_status_terminated_reason{reason="Error"}) by (namespace, pod, container,datacenter,instance) > 0
        for: 5m
        labels:
          severity: 严重
        annotations:
          summary: "Kubernetes 容器OOMKilled (实例 {{ $labels.instance }})"
          description: "Kubernetes 容器 {{ $labels.container }} 在 {{ $labels.namespace }} 命名空间中发生了 OOMKilled 事件。"

  - name: Kubernetes Pod JobFailed
    rules:
      - alert: Kubernetes 容器JobFailed
        expr: sum(kube_pod_container_status_terminated_reason{reason="Error"}) by (namespace, pod, container,datacenter,instance) > 0
        for: 5m
        labels:
          severity: 严重
        annotations:
          summary: "Kubernetes 容器JobFailed (实例 {{ $labels.instance }})"
          description: "Kubernetes 容器 {{ $labels.container }} 在 {{ $labels.namespace }} 命名空间中发生了 JobFailed 事件。"

  - name: Kubernetes Pod 状态异常
    rules:
      - alert: Kubernetes Pod 状态异常
        expr: sum(kube_pod_status_phase{phase=~"Pending|Failed|Unknown"}) by (namespace, pod, container,datacenter,instance) > 0
        for: 5m
        labels:
          severity: 严重
        annotations:
          summary: "Kubernetes Pod 状态异常 (实例 {{ $labels.instance }})"
          description: "Kubernetes Pod {{ $labels.pod }} 在 {{ $labels.namespace }} 命名空间中出现了异常状态。"

  - name: Kubernetes pod 崩溃循环
    rules:
      - alert: Kubernetes pod 崩溃循环
        expr: increase(kube_pod_container_status_restarts_total[1m]) > 3
        for: 5m
        labels:
          severity: 严重
        annotations:
            summary: "Kubernetes pod 崩溃循环 (实例 {{ $labels.instance }})"
            description: "Kubernetes pod {{ $labels.pod }} 在 {{ $labels.namespace }} 命名空间中发生了崩溃循环。"

```

