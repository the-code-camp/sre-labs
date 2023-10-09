## Setting up the Alertmanager

Alertmanager does not come with Prometheus by default so we need to install it.

Let's download and run the Alertmanager with a configuration that sends all alerts to the `#alerts` channel in our Mattermost setup. We will use Alertmanager's Slack notification integration for this, as it is compatible with Mattermost's.

Run this command to download Alertmanager v0.24.0 for Linux (see [the download page](https://prometheus.io/download/#alertmanager) for other versions):

```bash
wget https://github.com/prometheus/alertmanager/releases/download/v0.24.0/alertmanager-0.24.0.linux-amd64.tar.gz
```

Then, extract the tarball.  You can run this from the Desktop folder:

```bash
tar xvfz alertmanager-0.24.0.linux-amd64.tar.gz
```

Then, change into the extracted directory:

```bash
cd alertmanager-0.24.0.linux-amd64
```

Create (or replace) the file `alertmanager.yml` with the following content. You will need to replace the <webhook URL> placeholder in the configuration below with the webhook URL that Mattermost gave you earlier and you should have noted down:

```yml
route:
  group_by: ['alertname', 'job']

  group_wait: 30s
  group_interval: 5m
  repeat_interval: 3h

  receiver: slack

receivers:
- name: slack
  slack_configs:
  - api_url: '<webhook URL>' # <––– REPLACE THIS!
    channel: '#alerts'
    # Also send notifications for resolved alerts.
    send_resolved: true
    # A message title that shows a summary of the alerts.
    title: |-
      [{{ .Status | toUpper }}{{ if eq .Status "firing" }}:{{ .Alerts.Firing | len }}{{ end }}] {{ .CommonLabels.alertname }} for {{ .CommonLabels.job }}
      {{- if gt (len .CommonLabels) (len .GroupLabels) -}}
        {{" "}}(
        {{- with .CommonLabels.Remove .GroupLabels.Names }}
          {{- range $index, $label := .SortedPairs -}}
            {{ if $index }}, {{ end }}
            {{- $label.Name }}="{{ $label.Value -}}"
          {{- end }}
        {{- end -}}
        )
      {{- end }}
    # The message body that shows full label and annotation details for all alerts.
    text: >-
      {{ with index .Alerts 0 -}}
        :chart_with_upwards_trend: *<{{ .GeneratorURL }}|Graph>*
      {{ end }}

      *Alert details*:

      {{ range .Alerts -}}
        *Alert:* {{ .Annotations.title }}{{ if .Labels.severity }} - `{{ .Labels.severity }}`{{ end }}
      *Description:* {{ .Annotations.description }}
      *Details:*
        {{ range .Labels.SortedPairs }} • *{{ .Name }}:* `{{ .Value }}`
        {{ end }}
      {{ end }}

# Inhibit notifications for individual down instances
# when all instances are down for the same job.
inhibit_rules:
- source_matchers:
  - alertname="DemoServiceAllInstancesDown"
  target_matchers:
  - alertname="DemoServiceInstanceDown"
  equal:
  - job
```

The configuration uses heavy templating in the title and text fields of the Slack / Mattermost notification to show a summary of the received alerts in the title, as well as the full label and annotation details, along with a URL pointing back to the generating Prometheus server, in the message body. It also sets up an inhibit rule to not send notifications for the `DemoServiceInstanceDown` alert when the `DemoServiceAllInstancesDown` alert is simultaneously firing for the same job.

Go ahead and run the Alertmanager:

```bash
./alertmanager
```

The Alertmanager should start up and output some logs...

By default, the Alertmanager will load its configuration file from the file `alertmanager.yml` in the current directory. If you need to override this, you can use the command-line flag `--config.file=<filename>`. It will also store its notification log and the currently configured silences in a `data/` subdirectory. You can override this using the `--storage.path` flag.

The Alertmanager listens on port 9093 by default (this can be adjusted using the `--web.listen-address` flag). Head to http://localhost:9093/ to view its web-based user interface.

## Pointing Prometheus to Alertmanager

Next, we need to tell Prometheus where to find our Alertmanager, so it can send firing alerts to it.

Add the following to the scrape_configs section in your `prometheus.yml` Prometheus configuration file (which should live at `/etc/prometheus/prometheus.yml`):

```yaml
alerting:
  alertmanagers:
  - static_configs:
    - targets: ['localhost:9093']
```
Since we have edited the config file we need to restart the Prometheus server -  Reload the Prometheus configuration by sending a HUP signal to the Prometheus process:

```bash
killall -HUP prometheus
```

Head to your Prometheus server's status page at http://localhost:9090/status and verify that the "Alertmanagers" heading lists the configured Alertmanager endpoint.
