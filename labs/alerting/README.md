## Lab Overview

In this lab, we will scrape a demo service and then configure example alerting rules for it. 

We have configured a [demo service (apps.yml)](./apps.yml) in a Docker image  that you can run locally. Prometheus will scrape three instances of our local demo service that exposes synthetic metrics about HTTP API calls, resource usage, and more.

For this lab, you have to first configure the prometheus instance. Set the Prometheus configuration file `./config/prometheus.yml` to:

```yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'demo'
    static_configs:
      - targets:
        - 'demo-service-1:8080'
        - 'demo-service-2:8080'
        - 'demo-service-3:8080'
```

This instructs Prometheus to scrape the three demo service targets every 15 seconds, and it also sets the global rule evaluation interval to the same duration.

After making changes to `./config/prometheus.yml`, start the Prometheus and Applications container:

```sh
# move to the labs folder
cd labs/alerting
#start the apps and prometheus containers
docker compose -f infra.yml -f apps.yml up -d
```

Navigate to your Prometheus server's status page at http://localhost:9090/targets and verify that the three demo service targets are being scraped successfully.

## Configuring Alerting Rules

Let's create a few example alerts for our demo service. These fall into two categories:

Basic scrape health alerts:
- Alert when a single instance cannot be scraped.
- Alert when no instance at all can be scraped.
- Alert when the demo job's up metric is absent.

Alerts for user-visible symptoms:
- Alert for paths with high request latencies.
- Alert for paths with high error rates.

Save the following alerting rules in the file named `./config/demo-alerts.yml` in the same directory as the main Prometheus configuration file:

```yaml
groups:
- name: demo-service-alerts
  rules:
  # Alert (as a warning) when an individual instance cannot be scraped.
  - alert: DemoServiceInstanceDown
    expr: up{job="demo"} == 0
    for: 1m
    labels:
      severity: warning
    annotations:
      title: 'Instance {{ $labels.instance }} of {{ $labels.job }} down'
      description: 'Unable to scrape instance {{ $labels.instance }} of {{ $labels.job }}.'

  # Alert (as a critical alert) when no demo service instances at all can be scraped.
  # Meaning
    # - Sums all targets with job="demo" (ignoring instance labels).
    # - Compares if the total is exactly 0, meaning all targets are down.
  # Will match if:
    # Targets exist but all of them are reporting 0 (i.e., down).
    # Useful for alerting when a whole service is down, not just gone.

  - alert: DemoServiceAllInstancesDown
    expr: sum without(instance) (up{job="demo"}) == 0
    for: 30s
    labels:
      severity: critical
    annotations:
      title: '{{ $labels.job }} all instances down'
      description: 'Unable to scrape instance {{ $labels.instance }} of {{ $labels.job }}.'

  # Alert when no demo service instances are present.
  # Returns 1 (with empty labels) only if there are no time series matching up{job="demo"} at all.
  # Will NOT match if:
   # - Targets exist but their value is 0 (i.e., down).
   # - It only checks if the time series is completely missing.

  #Example: If your job is removed from config or a target is deleted — then absent() kicks in.
  - alert: DemoServiceAbsent
    expr: absent(up{job="demo"})
    for: 1m
    labels:
      severity: critical
    annotations:
      title: '{{ $labels.job }} is absent'
      description: 'No instances of {{ $labels.job }} are being scraped.'

  # Alert on path/method combinations with an error rate >0.5%.\
  # We calculate the per-second rate of requests in the past 1 minute. 
  # (5xx_rate / total_rate) * 100
  - alert: DemoServiceHighErrorRate
    expr: |
      (
        sum without(status, instance) (
          rate(demo_api_request_duration_seconds_count{status=~"5..",job="demo"}[1m])
        )
      /
        sum without(status, instance) (
          rate(demo_api_request_duration_seconds_count{job="demo"}[1m])
        ) * 100 > 0.5
      )
    for: 1m
    labels:
      severity: critical
    annotations:
      title: 'High 5xx rate for {{ $labels.method }} on {{ $labels.path }}'
      description: 'The 5xx error rate for path {{$labels.path}} with method {{ $labels.method }} in {{ $labels.job }} is {{ printf "%.2f" $value }}%.'

  # Alert on path/method combinations with a 99th percentile latency > 200ms.
  # Prometheus histogram-based latency SLO
  # This checks if the 99th percentile latency for a given API is above a threshold.

  # demo_api_request_duration_seconds_bucket
  # This is a histogram metric from a Prometheus client library (e.g., in Go):
  # It captures request durations in buckets.
  # Example buckets might be: 0.1s, 0.2s, 0.5s, 1s, 2s, etc.
  # It has cumulative counts:
    # le="0.1" → requests ≤ 100ms
    # le="1" → requests ≤ 1 second

  # histogram_quantile(0.99, ...)
    # Calculates the 99th percentile request latency (P99).
    # This estimates the latency under which 99% of the requests fall.

  # > 0.2
    # Triggers if the 99th percentile latency > 0.2 seconds (200ms)
    
  - alert: DemoServiceHighLatency
    expr: |
      histogram_quantile(
        0.99,
        sum without(status, instance) (
          rate(demo_api_request_duration_seconds_bucket{job="demo"}[5m])
        )
      ) > 0.2
    for: 1m
    labels:
      severity: critical
    annotations:
      title: 'High latency for {{ $labels.method }} on {{ $labels.path }}'
      description: 'The 99th percentile latency for path {{$labels.path}} with method {{ $labels.method }} in {{ $labels.job }} is {{ printf "%.2f" $value }}s.'

```

We have chosen relatively short for durations of one minute here (or even 30s for `DemoServiceAllInstancesDown`) so that we can quickly get some firing alerts. Note that the `DemoServiceHighErrorRate` and `DemoServiceHighLatency` alerts above are written in such a way that they produce regularly changing firing output alerts as the underlying error rates and latencies change over time.

Let's connect our Prometheus server to the alerts we just created.  Go ahead and load this rule file into Prometheus by adding a `rule_files` configuration directive to the top level of your Prometheus configuration file:

```yml
rule_files:
  - 'demo-alerts.yml'
```

The full Prometheus configuration now looks like this:

```yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - 'demo-alerts.yml'

scrape_configs:
  - job_name: 'demo'
    static_configs:
      - targets:
        - 'demo-service-1:8080'
        - 'demo-service-2:8080'
        - 'demo-service-3:8080'
```

Reload the Prometheus configuration:

```sh
docker compose -f infra.yml restart prometheus
```

## Inspecting Alerts in the UI

Head to http://localhost:9090/alerts to view the `"Alerts"` page on your Prometheus server. It should show you an overview of your configured alerting rules, along with a background color indicating whether a rule currently has no `active alerts (green), pending alerts (orange), or firing alerts (red)`.

![](/imgs/1.png)

Enable the `"Show annotations"` checkbox at the top right to include alert annotation details when expanding an alerting rule.

Clicking on the name of an alerting rule will expand details for that rule. For example, click on the `DemoServiceHighErrorRate` rule to reveal its currently active alerts:

![](/imgs/2.png)

In this example, there are `two pending` and `two firing alerts` (only a subset is shown in the screenshot). You can see the full labels, annotations, the output series' sample values, and the alert state for each active alert in this table.

While Prometheus already shows alert information on this page, it is not sending that information anywhere yet. In the next sections, we will learn about `Alertmanager` and how to connect `Prometheus` to it to send notifications for your alerts.

**Next** [Setup Slack/Mattermost for alerts](./mattermost.md)
