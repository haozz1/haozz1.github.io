---
title: Prometheus
date: 2022-06-01 15:56:09
tags: [Prometheus,Linux]
---

# Prometheus

### Prometheus is an open-source systems monitoring and alerting toolkit originally built at SoundCloud.


***Document of Prometheus*** https://prometheus.io/docs/introduction/overview/

***Download of Prometheus***https://prometheus.io/download/

---

## 1. Prometheus install

下载并解压压缩文件之后,进入到Prometheus目录
查看或修改配置文件
```yml
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first_rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["localhost:9090"]
```
运行以下命令启动Prometheus: 

```shell
./prometheus
```
or
```shell
./prometheus --config.file=prometheus.yml
```

控制台会打印出一些日志 
```shell
ts=2022-06-01T06:23:12.752Z caller=main.go:488 level=info msg="No time or size retention was set so using the default time retention" duration=15d
ts=2022-06-01T06:23:12.752Z caller=main.go:525 level=info msg="Starting Prometheus" version="(version=2.35.0, branch=HEAD, revision=6656cd29fe6ac92bab91ecec0fe162ef0f187654)"
ts=2022-06-01T06:23:12.752Z caller=main.go:530 level=info build_context="(go=go1.18.1, user=root@cf6852b14d68, date=20220421-09:53:42)"
ts=2022-06-01T06:23:12.752Z caller=main.go:531 level=info host_details="(Linux 5.4.0-1078-azure #81~18.04.1-Ubuntu SMP Mon Apr 25 23:16:13 UTC 2022 x86_64 TitanTest02 (none))"
ts=2022-06-01T06:23:12.752Z caller=main.go:532 level=info fd_limits="(soft=1024, hard=1048576)"
ts=2022-06-01T06:23:12.752Z caller=main.go:533 level=info vm_limits="(soft=unlimited, hard=unlimited)"
ts=2022-06-01T06:23:12.754Z caller=web.go:541 level=info component=web msg="Start listening for connections" address=0.0.0.0:9090
ts=2022-06-01T06:23:12.755Z caller=main.go:957 level=info msg="Starting TSDB ..."
```
可以看到Prometheus已经启动并启用了端口9090. 浏览器打开相应的地址就可以看到Prometheus的界面了

{% asset_img p.png %}

---

## 2. Sql Exporter

Prometheus提供了多种Exporter,可以帮助从第三方系统导出现有指标作为Prometheus指标

***Exporters***:https://prometheus.io/docs/instrumenting/exporters/#exporters-and-integrations

这里使用的是非官方的exporter
https://github.com/free/sql_exporter/tree/caf149bcfa2ccacb873e82c7e6bb8014b6a2b81d

和install Prometheus相似,修改sql_exporter.yml和*.collector.yml文件并启动exporter:

sql_exporter.yml:
```yml
global:
  # Subtracted from Prometheus' scrape_timeout to give us some headroom and prevent Prometheus from timing out first.
  scrape_timeout_offset: 500ms
  # Minimum interval between collector runs: by default (0s) collectors are executed on every scrape.
  min_interval: 0s
  # Maximum number of open connections to any one target. Metric queries will run concurrently on multiple connections,
  # as will concurrent scrapes.
  max_connections: 3
  # Maximum number of idle connections to any one target. Unless you use very long collection intervals, this should
  # always be the same as max_connections.
  max_idle_connections: 3

# A SQL Exporter job is the equivalent of a Prometheus job: a set of related DB instances.
jobs:

  # All metrics for the targets defined here get a `job="pricing_db"` label.
  - job_name: mssql_standard

    # Collectors (referenced by name) to execute on all targets in this job.
    collectors: [mssql_standard]

    # Similar to Prometheus static_configs.
    static_configs:
      - targets:
          # Map of instance name (exported as instance label) to DSN
          'data.database.windows.net:1433': 'sqlserver://username:password@data.database.windows.net:1433'
          'data.database.windows.net:1433': 'sqlserver://username:password@data.database.windows.net:1433'
        labels:
          env: 'prod'
      - targets:
          # Map of instance name (exported as instance label) to DSN
          'data.database.windows.net:1433': 'sqlserver://username:password@data-ingestion-tool.database.windows.net:1433'
        labels:
          env: 'test'

# Collector definition files.
collector_files:
  - "*.collector.yml"
```
>static_configs中还使用了已经不建议使用的对多sql server target的监控
>说明:https://github.com/free/sql_exporter/issues/6
---
*.collector.yml:
```yml
# A collector defining standard metrics for Microsoft SQL Server.
#
# It is required that the SQL Server user has the following permissions:
#
#   GRANT VIEW ANY DEFINITION TO
#   GRANT VIEW SERVER STATE TO
#
collector_name: mssql_standard

# Similar to global.min_interval, but applies to the queries defined by this collector only.
#min_interval: 0s

metrics:
  - metric_name: mssql_local_time_seconds
    type: gauge
    help: 'Local time in seconds since epoch (Unix time).'
    values: [unix_time]
    query: |
      SELECT DATEDIFF(second, '19700101', GETUTCDATE()) AS unix_time

  - metric_name: mssql_connections
    type: gauge
    help: 'Number of active connections.'
    key_labels:
      - db
    values: [count]
    query: |
      SELECT DB_NAME(sp.dbid) AS db, COUNT(sp.spid) AS count
      FROM sys.sysprocesses sp
      GROUP BY DB_NAME(sp.dbid)

  #
  # Collected from sys.dm_os_performance_counters
  #
  - metric_name: mssql_deadlocks
    type: counter
    help: 'Number of lock requests that resulted in a deadlock.'
    values: [cntr_value]
    query: |
      SELECT cntr_value
      FROM sys.dm_os_performance_counters WITH (NOLOCK)
      WHERE counter_name = 'Number of Deadlocks/sec' AND instance_name = '_Total'

  - metric_name: mssql_user_errors
    type: counter
    help: 'Number of user errors.'
    values: [cntr_value]
    query: |
      SELECT cntr_value
      FROM sys.dm_os_performance_counters WITH (NOLOCK)
      WHERE counter_name = 'Errors/sec' AND instance_name = 'User Errors'

  - metric_name: mssql_kill_connection_errors
    type: counter
    help: 'Number of severe errors that caused SQL Server to kill the connection.'
    values: [cntr_value]
    query: |
      SELECT cntr_value
      FROM sys.dm_os_performance_counters WITH (NOLOCK)
      WHERE counter_name = 'Errors/sec' AND instance_name = 'Kill Connection Errors'

  - metric_name: mssql_page_life_expectancy_seconds
    type: gauge
    help: 'The minimum number of seconds a page will stay in the buffer pool on this node without references.'
    values: [cntr_value]
    query: |
      SELECT top(1) cntr_value
      FROM sys.dm_os_performance_counters WITH (NOLOCK)
      WHERE counter_name = 'Page life expectancy'

  - metric_name: mssql_batch_requests
    type: counter
    help: 'Number of command batches received.'
    values: [cntr_value]
    query: |
      SELECT cntr_value
      FROM sys.dm_os_performance_counters WITH (NOLOCK)
      WHERE counter_name = 'Batch Requests/sec'

  - metric_name: mssql_log_growths
    type: counter
    help: 'Number of times the transaction log has been expanded, per database.'
    key_labels:
      - db
    values: [cntr_value]
    query: |
      SELECT rtrim(instance_name) AS db, cntr_value
      FROM sys.dm_os_performance_counters WITH (NOLOCK)
      WHERE counter_name = 'Log Growths' AND instance_name <> '_Total'

  #
  # Collected from sys.dm_io_virtual_file_stats
  #
  - metric_name: mssql_io_stall_seconds
    type: counter
    help: 'Stall time in seconds per database and I/O operation.'
    key_labels:
      - db
    value_label: operation
    values:
      - read
      - write
    query_ref: mssql_io_stall
  - metric_name: mssql_io_stall_total_seconds
    type: counter
    help: 'Total stall time in seconds per database.'
    key_labels:
      - db
    values:
      - io_stall
    query_ref: mssql_io_stall

  #
  # Collected from sys.dm_os_process_memory
  #
  - metric_name: mssql_resident_memory_bytes
    type: gauge
    help: 'SQL Server resident memory size (AKA working set).'
    values: [resident_memory_bytes]
    query_ref: mssql_process_memory

  - metric_name: mssql_virtual_memory_bytes
    type: gauge
    help: 'SQL Server committed virtual memory size.'
    values: [virtual_memory_bytes]
    query_ref: mssql_process_memory

  - metric_name: mssql_memory_utilization_percentage
    type: gauge
    help: 'The percentage of committed memory that is in the working set.'
    values: [memory_utilization_percentage]
    query_ref: mssql_process_memory

  - metric_name: mssql_page_fault_count
    type: counter
    help: 'The number of page faults that were incurred by the SQL Server process.'
    values: [page_fault_count]
    query_ref: mssql_process_memory

  #
  # Collected from sys.dm_os_sys_memory
  #
  - metric_name: mssql_os_memory
    type: gauge
    help: 'OS physical memory, used and available.'
    value_label: 'state'
    values: [used, available]
    query: |
      SELECT
        (total_physical_memory_kb - available_physical_memory_kb) * 1024 AS used,
        available_physical_memory_kb * 1024 AS available
      FROM sys.dm_os_sys_memory

  - metric_name: mssql_os_page_file
    type: gauge
    help: 'OS page file, used and available.'
    value_label: 'state'
    values: [used, available]
    query: |
      SELECT
        (total_page_file_kb - available_page_file_kb) * 1024 AS used,
        available_page_file_kb * 1024 AS available
      FROM sys.dm_os_sys_memory

queries:
  # Populates `mssql_io_stall` and `mssql_io_stall_total`
  - query_name: mssql_io_stall
    query: |
      SELECT
        cast(DB_Name(a.database_id) as varchar) AS [db],
        sum(io_stall_read_ms) / 1000.0 AS [read],
        sum(io_stall_write_ms) / 1000.0 AS [write],
        sum(io_stall) / 1000.0 AS io_stall
      FROM
        sys.dm_io_virtual_file_stats(null, null) a
      INNER JOIN sys.master_files b ON a.database_id = b.database_id AND a.file_id = b.file_id
      GROUP BY a.database_id

  # Populates `mssql_resident_memory_bytes`, `mssql_virtual_memory_bytes`, `mssql_memory_utilization_percentage` and
  # `mssql_page_fault_count`.
  - query_name: mssql_process_memory
    query: |
      SELECT
        physical_memory_in_use_kb * 1024 AS resident_memory_bytes,
        virtual_address_space_committed_kb * 1024 AS virtual_memory_bytes,
        memory_utilization_percentage,
        page_fault_count
      FROM sys.dm_os_process_memory
```
> 这里提供的是对SQL Server的一些系统metrics

修改后,启动sql exporter
```shell
./sql_exporter

I0601 06:58:45.515826   32145 main.go:52] Starting SQL exporter (version=0.5, branch=master, revision=fc5ed07ee38c5b90bab285392c43edfe32d271c5) (go=go1.11.3, user=root@f24ba5099571, date=20190114-09:24:06)
I0601 06:58:45.515996   32145 config.go:18] Loading configuration from sql_exporter.yml
I0601 06:58:45.516658   32145 config.go:131] Loaded collector "mssql_standard" from mssql_standard.collector.yml
I0601 06:58:45.516750   32145 main.go:67] Listening on :9399
```
sql exporter启动并使用了9399端口

修改Prometheus.yml,添加sql server target,并重启Prometheus
```yml
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["localhost:9090"]

  - job_name: "sql_exporter"
    honor_timestamps: true
    static_configs:
      - targets: ["localhost:9399"]
```
{% asset_img sqlp.png %}
在Prometheus http://localhost:9090/targets 上就可以看到已经启动的sql exporter,可以在 http://localhost:9399/metrics 看到已经收到的SQL Server metrics
{% asset_img sqlm.png %}

---

## 3. Zookeeper Exporter

zookeeper从3.6版本之后就提供了Prometheus相关的配置

安装zookeeper 3.6或更新的版本,将关于Prometheus相关的配置打开
```yml
# https://prometheus.io Metrics Exporter
metricsProvider.className=org.apache.zookeeper.metrics.prometheus.PrometheusMetricsProvider
#metricsProvider.httpHost=0.0.0.0
metricsProvider.httpPort=7000
metricsProvider.exportJvmInfo=true
```

在Prometheus.yml中添加zookeeper target并重启Prometheus
```yml
- job_name: zookeeper
  honor_timestamps: true
  scrape_interval: 1m
  scrape_timeout: 10s
  metrics_path: /metrics
  scheme: http
  follow_redirects: true
  enable_http2: true
  file_sd_configs:
  - files:
    - targets/zookeeper.json
    refresh_interval: 5m
```
> 此处使用了- files来配置zookeeper,使得对zookeeper的配置与Prometheus.yml解耦

在Prometheus根目录/targets下的zookeeper.json
```json
[
    {
        "labels": {
            "environment": "zookeeper01",
            "job": "zookeeper",
            "alert": "false"
        },
        "targets": [
            "ip1:7000"
        ]
    },
    {
        "labels": {
            "environment": "zookeeper02",
            "job": "zookeeper",
            "alert": "false"
        },
        "targets": [
            "ip2:7000"
        ]
    }
]
```
{% asset_img zkp.png %}
启动zookeeper之后,在 http://localhost:9090/targets 可以看到新的zookeeper target,同样的在 http://ip1:7000/metrics 可以看到zookeeper的metrics
{% asset_img zkm.png %}

---

## 4. Grafana

将收集到的metrics转化为更直观的可视化dashboard.

不在赘述Grafana的安装,只说一下Grafana中dashboard的创建,已zookeeper为例.

Grafana提供了大量的dashboard模板 https://grafana.com/grafana/dashboards/ ,找到zookeeper的dashboard https://grafana.com/grafana/dashboards/10465 并记下 ID `10465`

在Grafana中添加新的datasource
{% asset_img pdata.png %}
{% asset_img pdata2.png %}
{% asset_img pdata3.png %}
import zookeeper dashboard
{% asset_img dash.png %}
{% asset_img dash2.png %}

然后在dashboard中就可以看到关于zookeeper的metrics了
{% asset_img dash3.png %}

为dashboard添加新的filter
在dashboard中点击右上角齿轮dashboard settings,选择Variables添加environment
{% asset_img va.png %}
{% asset_img va2.png %}