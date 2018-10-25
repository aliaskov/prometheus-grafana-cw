A monitoring solution for Docker hosts and containers with Prometheus, Grafana, cAdvisor, NodeExporter and alerting with AlertManager. Ping (icmp, ssh, tcp) and pushgataway included.

A monitoring solution for Docker hosts and containers with [Prometheus](https://prometheus.io/), [Grafana](http://grafana.org/), [cAdvisor](https://github.com/google/cadvisor),
[NodeExporter](https://github.com/prometheus/node_exporter) and alerting with [AlertManager](https://github.com/prometheus/alertmanager), [CloudWatchExporter](https://github.com/prometheus/node_exporter),
[PushGateway](https://github.com/prometheus/pushgateway), [BlackboxExporter](https://github.com/prometheus/blackbox_exporter)


## Install

```
ADMIN_USER=admin ADMIN_PASSWORD=admin docker-compose up -d
```

Prerequisites:

* Docker Engine >= 1.13
* Docker Compose >= 1.11

Containers:

* Prometheus (metrics database) `http://<host-ip>:9090`
* AlertManager (alerts management) `http://<host-ip>:9093`
* Grafana (visualize metrics) `http://<host-ip>:3000`
* NodeExporter (host metrics collector) :9100
* cAdvisor (containers metrics collector) :8080
* CloudWatchExporter (export Cloudwatch metrics to Prometheus) :9106
* PushGateway (Receive metrics from short-live jobs) :9091
* BlackboxExporter (ping everything!)

## Setup Prometheus jobs

Prometheus is configured to check for data from exporters.
nodeexporter - host info
cadvisor - docker Containers
prometheus - DB itself
cloudwatch_exporter - Cloudwatch metrics API calls
pushgateway - Receive metrics from short-live jobs

![Prometheus_job_targets](https://github.com/aliaskov/prometheus-grafana-cw/blob/master/screens/Prometheus_job_targets.png)

After some time, prometheus would have some data:

![prometheus_ui](https://github.com/aliaskov/prometheus-grafana-cw/blob/master/screens/prometheus_ui.png)

No we can connect Grafana and customize it.
## Setup Grafana

Navigate to `http://<host-ip>:3000` and login with user ***admin*** password ***admin***. You can change the credentials in the compose file or by supplying the `ADMIN_USER` and `ADMIN_PASSWORD` environment variables on compose up.

Grafana is preconfigured with dashboards and Prometheus as the default data source:

* Name: Prometheus
* Type: Prometheus
* Url: http://prometheus:9090
* Access: proxy

and

* Name: Cloudwatch
* Type: Cloudwatch Exporter
* authType : arn - IAM ROLE
* defaultRegion : eu-central-1

The cloudwatch:ListMetrics and cloudwatch:GetMetricStatistics IAM permissions are required.

***Docker Host Dashboard***

![Host](https://raw.githubusercontent.com/aliaskov/prometheus-grafana-cw/master/screens/Grafana_Docker_Host.png)

The Docker Host Dashboard shows key metrics for monitoring the resource usage of your server:

* Server uptime, CPU idle percent, number of CPU cores, available memory, swap and storage
* System load average graph, running and blocked by IO processes graph, interrupts graph
* CPU usage graph by mode (guest, idle, iowait, irq, nice, softirq, steal, system, user)
* Memory usage graph by distribution (used, free, buffers, cached)
* IO usage graph (read Bps, read Bps and IO time)
* Network usage graph by device (inbound Bps, Outbound Bps)
* Swap usage and activity graphs

For storage and particularly Free Storage graph, you have to specify the fstype in grafana graph request.
You can find it in `grafana/dashboards/docker_host.json`, at line 480 :

      "expr": "sum(node_filesystem_free{fstype=\"btrfs\"})",

I work on BTRFS, so i need to change `aufs` to `btrfs`.

You can find right value for your system in Prometheus `http://<host-ip>:9090` launching this request :

      node_filesystem_free

***Docker Containers Dashboard***

![Containers](https://github.com/aliaskov/prometheus-grafana-cw/blob/master/screens/Grafana_Docker_Containers.png)

The Docker Containers Dashboard shows key metrics for monitoring running containers:

* Total containers CPU load, memory and storage usage
* Running containers graph, system load graph, IO usage graph
* Container CPU usage graph
* Container memory usage graph
* Container cached memory usage graph
* Container network inbound usage graph
* Container network outbound usage graph

Note that this dashboard doesn't show the containers that are part of the monitoring stack.

***Monitor Services Dashboard***

![Monitor Services](https://github.com/aliaskov/prometheus-grafana-cw/blob/master/screens/Grafana_Prometheus.png)

The Monitor Services Dashboard shows key metrics for monitoring the containers that make up the monitoring stack:

* Prometheus container uptime, monitoring stack total memory usage, Prometheus local storage memory chunks and series
* Container CPU usage graph
* Container memory usage graph
* Prometheus chunks to persist and persistence urgency graphs
* Prometheus chunks ops and checkpoint duration graphs
* Prometheus samples ingested rate, target scrapes and scrape duration graphs
* Prometheus HTTP requests graph
* Prometheus alerts graph

I've set the Prometheus retention period to 200h and the heap size to 1GB, you can change these values in the compose file.

```yaml
  prometheus:
    image: prom/prometheus
    command:
      - '-storage.local.target-heap-size=1073741824'
      - '-storage.local.retention=200h'
```

Make sure you set the heap size to a maximum of 50% of the total physical memory.

## ping infrastructure
![BlackboxExporter](https://github.com/aliaskov/prometheus-grafana-cw/blob/master/screens/blackboxexporter_ping.png)

## Define alerts

You can modify the alert rules and reload them by making a HTTP POST call to Prometheus:

```
curl -X POST http://admin:admin@<host-ip>:9090/-/reload
```

***Monitoring services alerts***

Trigger an alert if any of the monitoring targets (node-exporter and cAdvisor) are down for more than 30 seconds:

```yaml
ALERT monitor_service_down
  IF up == 0
  FOR 30s
  LABELS { severity = "critical" }
  ANNOTATIONS {
      summary = "Monitor service non-operational",
      description = "{{ $labels.instance }} service is down.",
  }
```

***Docker Host alerts***

Trigger an alert if the Docker host CPU is under hight load for more than 30 seconds:

```yaml
ALERT high_cpu_load
  IF node_load1 > 1.5
  FOR 30s
  LABELS { severity = "warning" }
  ANNOTATIONS {
      summary = "Server under high load",
      description = "Docker host is under high load, the avg load 1m is at {{ $value}}. Reported by instance {{ $labels.instance }} of job {{ $labels.job }}.",
  }
```

Modify the load threshold based on your CPU cores.

Trigger an alert if the Docker host memory is almost full:

```yaml
ALERT high_memory_load
  IF (sum(node_memory_MemTotal) - sum(node_memory_MemFree + node_memory_Buffers + node_memory_Cached) ) / sum(node_memory_MemTotal) * 100 > 85
  FOR 30s
  LABELS { severity = "warning" }
  ANNOTATIONS {
      summary = "Server memory is almost full",
      description = "Docker host memory usage is {{ humanize $value}}%. Reported by instance {{ $labels.instance }} of job {{ $labels.job }}.",
  }
```

Trigger an alert if the Docker host storage is almost full:

```yaml
ALERT hight_storage_load
  IF (node_filesystem_size{fstype="aufs"} - node_filesystem_free{fstype="aufs"}) / node_filesystem_size{fstype="aufs"}  * 100 > 85
  FOR 30s
  LABELS { severity = "warning" }
  ANNOTATIONS {
      summary = "Server storage is almost full",
      description = "Docker host storage usage is {{ humanize $value}}%. Reported by instance {{ $labels.instance }} of job {{ $labels.job }}.",
  }
```

***Docker Containers alerts***

Trigger an alert if a container is down for more than 30 seconds:

```yaml
ALERT jenkins_down
  IF absent(container_memory_usage_bytes{name="jenkins"})
  FOR 30s
  LABELS { severity = "critical" }
  ANNOTATIONS {
    summary= "Jenkins down",
    description= "Jenkins container is down for more than 30 seconds."
  }
```

Trigger an alert if a container is using more than 10% of total CPU cores for more than 30 seconds:

```yaml
 ALERT jenkins_high_cpu
  IF sum(rate(container_cpu_usage_seconds_total{name="jenkins"}[1m])) / count(node_cpu{mode="system"}) * 100 > 10
  FOR 30s
  LABELS { severity = "warning" }
  ANNOTATIONS {
    summary= "Jenkins high CPU usage",
    description= "Jenkins CPU usage is {{ humanize $value}}%."
  }
```

Trigger an alert if a container is using more than 1,2GB of RAM for more than 30 seconds:

```yaml
ALERT jenkins_high_memory
  IF sum(container_memory_usage_bytes{name="jenkins"}) > 1200000000
  FOR 30s
  LABELS { severity = "warning" }
  ANNOTATIONS {
      summary = "Jenkins high memory usage",
      description = "Jenkins memory consumption is at {{ humanize $value}}.",
  }
```

## Setup alerting

The AlertManager service is responsible for handling alerts sent by Prometheus server.
AlertManager can send notifications via email, Pushover, Slack, HipChat or any other system that exposes a webhook interface.
A complete list of integrations can be found [here](https://prometheus.io/docs/alerting/configuration).

You can view and silence notifications by accessing `http://<host-ip>:9093`.

The notification receivers can be configured in [alertmanager/config.yml](https://github.com/stefanprodan/dockprom/blob/master/alertmanager/config.yml) file.

To receive alerts via Slack you need to make a custom integration by choose ***incoming web hooks*** in your Slack team app page.
You can find more details on setting up Slack integration [here](http://www.robustperception.io/using-slack-with-the-alertmanager/).

Copy the Slack Webhook URL into the ***api_url*** field and specify a Slack ***channel***.

```yaml
route:
    receiver: 'slack'

receivers:
    - name: 'slack'
      slack_configs:
          - send_resolved: true
            text: "{{ .CommonAnnotations.description }}"
            username: 'Prometheus'
            channel: '#<channel>'
            api_url: 'https://hooks.slack.com/services/<webhook-id>'
```

![Slack Notifications](https://raw.githubusercontent.com/stefanprodan/dockprom/master/screens/Slack_Notifications.png)
