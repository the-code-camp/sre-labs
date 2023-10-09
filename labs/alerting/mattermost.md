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
docker run \
  -d \
  --name mattermost-preview \
  --publish 8065:8065 \
  --add-host dockerhost:127.0.0.1 \
  mattermost/mattermost-preview:6.5.0
```

Note that in order to copy and paste while in your virtual machines you will need to use control+shift+c and control+shift+v.

Give it a few minutes for the database initialization of Mattermost to complete. Then, head to http://localhost:8065/ to see the Mattermost web interface.


## Create a local account in Mattermost

First off, create an account in your local installation with the email `alertmanager@somecompany.com`, the username `alertmanager`, and the password `alertmanager` (these details don't matter and you can choose other values if you like).

Second, create an organization named `"Demo Organization"`

Confirm your Mattermost server's URL. The default of http://localhost:8065 should already be correct here.

Skip the question about how you plan on using Mattermost...as well as the question about tools

To add an incoming webhook for the `#alerts` channel that the Alertmanager can use to send notifications, open the main menu at the top left and select `"Integrations"`:

<img width="1032" alt="Screen Shot 2023-02-14 at 8 47 41 AM" src="https://user-images.githubusercontent.com/25653204/218756798-343e025a-2b07-40f6-ae52-4c6beeb26f66.png">

Select the "incoming webhook" integration type:

<img width="1032" alt="Screen Shot 2023-02-14 at 8 48 52 AM" src="https://user-images.githubusercontent.com/25653204/218757116-ccf7f0a8-0c07-40a4-bd99-3d1b15673d2e.png">

Then, click "Add Incoming Webhook"

Finally, set the title of the new webhook to `"Alertmanager"` and set its default channel to `#alerts` - and click Save at the bottom:

<img width="1037" alt="Screen Shot 2023-02-14 at 8 49 44 AM" src="https://user-images.githubusercontent.com/25653204/218757316-045fc571-aa48-45b1-bb5a-9849097263e7.png">

After clicking `"Save"`, Mattermost will show you the `URL of the newly created webhook` - Note down this URL, as we will use it in our `Alertmanager` configuration in the next step.
