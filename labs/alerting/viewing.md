## Viewing Alerts in Alertmanager

The Alertmanager's web UI is a great place to see all the alerts that are currently firing in the Prometheus servers connected to that particular Alertmanager. Head to http://localhost:9093/ to see all firing alerts that it has currently received from Prometheus. It should look something like this (depending on which alerts are currently firing):

![](/imgs/6.png)

The input field in the `"Filter"` tab lets you enter PromQL-style label matchers to only display alerts with specific label values. For example, you could enter `alertname="DemoServiceHighErrorRate"` to only show alerts with the name `DemoServiceHighErrorRate`:

![](/imgs/7.png)

The `"Group"` tab allows you to configure how to group the displayed alerts. By default, alerts are grouped by their `group_by` labels of the route they belong to. Using the `"Enable custom grouping"` checkbox, you can specify a manual list of label names to group the alerts by.

You can also filter which receivers to show alerts for, as well as whether to also include silenced or inhibited alerts in the display.

On an individual alert, you can click on the `"Info"` field to show the full annotation details:

![](/imgs/8.png)

You can also click on the `"Source"` link to show the underlying alerting PromQL expression in the Prometheus server that generated the alert:

![](/imgs/9.png)

During an outage, inspecting the original alerting rule expression like this is often a good starting point for further debugging. You could then proceed to refine the alerting expression into something that helps you pinpoint the problem better (e.g. by changing aggregation levels or removing filters).


## Viewing Notifications in Mattermost

By now, some alert notifications for the `DemoServiceHighErrorRate` and `DemoServiceHighLatency` alerts should have arrived in the `#alerts` channel on Mattermost (or Slack) as well:

![](/imgs/10.png)

You can see how the templated title and body texts include a lot of the label and annotation data, as well as a link back to the alert-generating Prometheus server. Notifications for firing alerts are shown in red, while resolved ones are shown with a green color.

In the notification example above, you can see that the demo service is currently experiencing a high error rate of `0`.`5%` for `GET` requests on the `/api/foo` path and even higher error rates for two other path and method combinations. As a next step, you would likely consult your service's request logs, traces, or similar signal type to determine the exact nature of the problem.


## Setting Silences

Sometimes you already know that you don't want to receive any more alerts of a particular kind for a given period of time. This could be because you are performing planned maintenance for a system or service, or because you are already aware of an ongoing incident and getting more alerts for it would be distracting. The Alertmanager web UI lets you create a temporary silence for this kind of situation. Silences mute a set of alerts based on a set of label matchers and a start and end time. After the end time is reached, the Alertmanager automatically expires the silence, which avoids accidentally muting alerts forever.

Let's imagine that we are aware of high error rates in our demo service and are already looking into the problem. To avoid receiving further notifications for this issue while we are fixing things, let's create a one-hour silence for the DemoServiceHighErrorRate alert.

Head to http://localhost:9093/#/silences and click on "New Silence":

![](/imgs/11.png)

First we need to configure when our silence should start and when it should expire. By default, the `"Start"` field will contain the current time, and the `"End"` field will show a time in the future that changes depending on the `"Duration"` field (`2h` by default). Change the `"Duration"` field to `1h` for a one-hour silence and note how the `"End"` field automatically updates as well:

![](/imgs/12.png)

Next, we need to provide a list of label matchers to tell Alertmanager which alerts to silence. We need to be careful here to avoid silencing unrelated alerts! To be safe, let's create a matcher both for the name of the alert (`alertname="DemoServiceHighErrorRate"` label) as well as the job name (`job="demo"`):

![](/imgs/13.png)

You can enter anything you like in the `"Creator"` and `"Comment"` fields. These fields remind yourself or tell a colleague who created the silence and why.

Click on `"Preview Alerts"` to see a list of currently firing alerts known to the `Alertmanager` that would be muted by this silence. 

Previewing the to-be-silenced alerts can help reassure you that your label matchers are both sufficient and specific enough to silence the intended alerts, but not unrelated ones.

Finally, click on `"Create"` to create the silence. You can now also confirm that you won't get any further notifications in Mattermost for the `DemoServiceHighErrorRate` alerts for one hour.
