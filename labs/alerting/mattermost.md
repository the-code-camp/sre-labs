# Lab Overview

Our Prometheus server is now evaluating and showing firing alerts based on the alerting rules we configured, but we are not getting any notifications for alerts yet. To send alert notifications, we will need to set up an `Alertmanager` and tell Prometheus to forward its alerts to it. 

The `Alertmanager` will then allow us to `route`, `group`, `throttle`, and otherwise `process` the incoming stream of raw alerts into sensible notifications that you can consume as an operator.

Receiving `Alertmanager` notifications via `Slack` is a very common setup.

However, we want to use some open source software instead to avoid any licensing issues with Slack.

Lucky for us, the open-source Slack clone [Mattermost](https://mattermost.com/) has a Slack-compatible webhook API and is easy to set up as a local preview environment using Docker.

We will create an `#alerts` channel in Mattermost as well as a webhook integration for sending messages to that channel. This will let us send Slack-style notifications to our Mattermost channel so we can see what these notifications look like.

## Running Mattermost in Docker

The easiest way to run Mattermost is via Docker:

```bash
docker compose -f mattermost.yml up -d
```

Give it a few minutes for the database initialization of Mattermost to complete. Then, head to http://localhost:8065/ to see the Mattermost web interface.


## Create a local account in Mattermost

First off, create an account in your local installation with the email `alertmanager@somecompany.com`, the username `alertmanager`, and the password `alertmanager` (these details don't matter and you can choose other values if you like).

Second, create an organization named `"Demo Organization"`

Confirm your Mattermost server's URL. The default of http://localhost:8065 should already be correct here.

Skip the question about how you plan on using Mattermost...as well as the question about tools

To add an incoming webhook for the `#alerts` channel that the Alertmanager can use to send notifications, open the main menu at the top left and select `"Integrations"`:

![](/imgs/3.png)

Select the "incoming webhook" integration type:

![](/imgs/4.png)

Then, click "Add Incoming Webhook"

Finally, set the title of the new webhook to `"Alertmanager"` and set its default channel to `#alerts` - and click Save at the bottom:

![](/imgs/5.png)

After clicking `"Save"`, Mattermost will show you the `URL of the newly created webhook` - Note down this URL, as we will use it in our `Alertmanager` configuration in the next step. You need to update the URL from `localhost:8065` to `mattermost:8065`

![](/imgs/mattermost/6.png)

**Next** [Setting up the Alert manager](./alertmanager.md)