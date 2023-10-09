## Viewing Alerts in Alertmanager

The Alertmanager's web UI is a great place to see all the alerts that are currently firing in the Prometheus servers connected to that particular Alertmanager. Head to http://localhost:9093/ to see all firing alerts that it has currently received from Prometheus. It should look something like this (depending on which alerts are currently firing):


<img width="1035" alt="Screen Shot 2023-02-14 at 12 47 02 PM" src="https://user-images.githubusercontent.com/25653204/218816882-90420b25-a7e9-4c6c-b09b-47f97816e06a.png">


The input field in the `"Filter"` tab lets you enter PromQL-style label matchers to only display alerts with specific label values. For example, you could enter `alertname="DemoServiceHighErrorRate"` to only show alerts with the name `DemoServiceHighErrorRate`:

<img width="1066" alt="Screen Shot 2023-02-14 at 12 50 50 PM" src="https://user-images.githubusercontent.com/25653204/218817728-53ee91a5-ef2c-4be8-b679-3cc48a245734.png">

The `"Group"` tab allows you to configure how to group the displayed alerts. By default, alerts are grouped by their `group_by` labels of the route they belong to. Using the `"Enable custom grouping"` checkbox, you can specify a manual list of label names to group the alerts by.

You can also filter which receivers to show alerts for, as well as whether to also include silenced or inhibited alerts in the display.

On an individual alert, you can click on the `"Info"` field to show the full annotation details:

<img width="1103" alt="Screen Shot 2023-02-14 at 12 56 27 PM" src="https://user-images.githubusercontent.com/25653204/218818861-01bbe288-7dcd-4dbc-bdae-d1ce7dbd7734.png">

You can also click on the `"Source"` link to show the underlying alerting PromQL expression in the Prometheus server that generated the alert:

<img width="1056" alt="Screen Shot 2023-02-14 at 12 57 12 PM" src="https://user-images.githubusercontent.com/25653204/218819022-f3d3d103-b4b8-4a7d-9951-68ce19479ead.png">

During an outage, inspecting the original alerting rule expression like this is often a good starting point for further debugging. You could then proceed to refine the alerting expression into something that helps you pinpoint the problem better (e.g. by changing aggregation levels or removing filters).


## Viewing Notifications in Mattermost

By now, some alert notifications for the `DemoServiceHighErrorRate` and `DemoServiceHighLatency` alerts should have arrived in the `#alerts` channel on Mattermost (or Slack) as well:

<img width="1086" alt="Screen Shot 2023-02-14 at 1 00 15 PM" src="https://user-images.githubusercontent.com/25653204/218819658-f0311a07-852d-4151-bfbf-fb249a1cc8ab.png">

You can see how the templated title and body texts include a lot of the label and annotation data, as well as a link back to the alert-generating Prometheus server. Notifications for firing alerts are shown in red, while resolved ones are shown with a green color.

In the notification example above, you can see that the demo service is currently experiencing a high error rate of `0`.`5%` for `GET` requests on the `/api/foo` path and even higher error rates for two other path and method combinations. As a next step, you would likely consult your service's request logs, traces, or similar signal type to determine the exact nature of the problem.


## Setting Silences

Sometimes you already know that you don't want to receive any more alerts of a particular kind for a given period of time. This could be because you are performing planned maintenance for a system or service, or because you are already aware of an ongoing incident and getting more alerts for it would be distracting. The Alertmanager web UI lets you create a temporary silence for this kind of situation. Silences mute a set of alerts based on a set of label matchers and a start and end time. After the end time is reached, the Alertmanager automatically expires the silence, which avoids accidentally muting alerts forever.

Let's imagine that we are aware of high error rates in our demo service and are already looking into the problem. To avoid receiving further notifications for this issue while we are fixing things, let's create a one-hour silence for the DemoServiceHighErrorRate alert.

Head to http://localhost:9093/#/silences and click on "New Silence":

<img width="1048" alt="Screen Shot 2023-02-14 at 1 06 03 PM" src="https://user-images.githubusercontent.com/25653204/218820852-06c0c256-cabb-4c35-a401-635932222720.png">

First we need to configure when our silence should start and when it should expire. By default, the `"Start"` field will contain the current time, and the `"End"` field will show a time in the future that changes depending on the `"Duration"` field (`2h` by default). Change the `"Duration"` field to `1h` for a one-hour silence and note how the `"End"` field automatically updates as well:

<img width="1083" alt="Screen Shot 2023-02-14 at 1 07 30 PM" src="https://user-images.githubusercontent.com/25653204/218821143-8f9a2e15-24e2-4114-8736-50bc397cc3de.png">

Next, we need to provide a list of label matchers to tell Alertmanager which alerts to silence. We need to be careful here to avoid silencing unrelated alerts! To be safe, let's create a matcher both for the name of the alert (`alertname="DemoServiceHighErrorRate"` label) as well as the job name (`job="demo"`):

<img width="1093" alt="Screen Shot 2023-02-14 at 1 10 59 PM" src="https://user-images.githubusercontent.com/25653204/218821803-83ea77dd-0b92-4d5b-9920-65e776e43fe5.png">

You can enter anything you like in the `"Creator"` and `"Comment"` fields. These fields remind yourself or tell a colleague who created the silence and why.

Click on `"Preview Alerts"` to see a list of currently firing alerts known to the `Alertmanager` that would be muted by this silence. 

Previewing the to-be-silenced alerts can help reassure you that your label matchers are both sufficient and specific enough to silence the intended alerts, but not unrelated ones.

Finally, click on `"Create"` to create the silence. You can now also confirm that you won't get any further notifications in Mattermost for the `DemoServiceHighErrorRate` alerts for one hour.
