+++
title = 'Cassandra Exporter'
date = 2024-03-25T09:26:56+08:00
draft = true
+++


在cassandra 节点 - data1

1. 下载包

```sh
wget https://repo1.maven.org/maven2/io/prometheus/jmx/jmx_prometheus_javaagent/0.19.0/jmx_prometheus_javaagent-0.19.0.jar
```
放在data1:

```sh
 mv jmx_prometheus_javaagent-0.19.0.jar /usr/share/cassandra/lib/
```



2.1  配置修改1
` /etc/cassandra/conf/cassandra-jmx.yaml`

```sh
lowercaseOutputLabelNames: true
lowercaseOutputName: true
whitelistObjectNames: ["org.apache.cassandra.metrics:*"]
# ColumnFamily is an alias for Table metrics
blacklistObjectNames: ["org.apache.cassandra.metrics:type=ColumnFamily,*"]
rules:
# Generic gauges with 0-2 labels
- pattern: org.apache.cassandra.metrics<type=(\S*)(?:, ((?!scope)\S*)=(\S*))?(?:, scope=(\S*))?, name=(\S*)><>Value
  name: cassandra_$1_$5
  type: GAUGE
  labels:
    "$1": "$4"
    "$2": "$3"

#
# Emulate Prometheus 'Summary' metrics for the exported 'Histogram's.
# TotalLatency is the sum of all latencies since server start
#
- pattern: org.apache.cassandra.metrics<type=(\S*)(?:, ((?!scope)\S*)=(\S*))?(?:, scope=(\S*))?, name=(.+)?(?:Total)(Latency)><>Count
  name: cassandra_$1_$5$6_seconds_sum
  type: UNTYPED
  labels:
    "$1": "$4"
    "$2": "$3"
  # Convert microseconds to seconds
  valueFactor: 0.000001

- pattern: org.apache.cassandra.metrics<type=(\S*)(?:, ((?!scope)\S*)=(\S*))?(?:, scope=(\S*))?, name=((?:.+)?(?:Latency))><>Count
  name: cassandra_$1_$5_seconds_count
  type: UNTYPED
  labels:
    "$1": "$4"
    "$2": "$3"

- pattern: org.apache.cassandra.metrics<type=(\S*)(?:, ((?!scope)\S*)=(\S*))?(?:, scope=(\S*))?, name=(.+)><>Count
  name: cassandra_$1_$5_count
  type: UNTYPED
  labels:
    "$1": "$4"
    "$2": "$3"

- pattern: org.apache.cassandra.metrics<type=(\S*)(?:, ((?!scope)\S*)=(\S*))?(?:, scope=(\S*))?, name=((?:.+)?(?:Latency))><>(\d+)thPercentile
  name: cassandra_$1_$5_seconds
  type: GAUGE
  labels:
    "$1": "$4"
    "$2": "$3"
    quantile: "0.$6"
  # Convert microseconds to seconds
  valueFactor: 0.000001

- pattern: org.apache.cassandra.metrics<type=(\S*)(?:, ((?!scope)\S*)=(\S*))?(?:, scope=(\S*))?, name=(.+)><>(\d+)thPercentile
  name: cassandra_$1_$5
  type: GAUGE
  labels:
    "$1": "$4"
    "$2": "$3"
    quantile: "0.$6"
    
```



2.2 修改配置2

```sh
cat <<EOF >> /etc/cassandra/conf/cassandra-env.sh
# 7070为暴露的数据采集端口
JVM_OPTS="\$JVM_OPTS -javaagent:\$CASSANDRA_HOME/lib/jamm-0.3.0.jar -javaagent:\$CASSANDRA_HOME/lib/jmx_prometheus_javaagent-0.19.0.jar=7070:/etc/cassandra/conf/cassandra-jmx.yaml"
EOF
```

3. 重启服务：

```sh
systemctl restart cassandra
systemctl status cassandra
```

4. 验证：

```sh
 curl 127.0.0.1:7070/metrics
```

接入：
```shell
  - job_name: 'cassandra'
    static_configs:
    - targets: ['cassandra-ip:7070']
```



