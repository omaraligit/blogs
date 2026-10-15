# How to Get Website Downtime Alerts on Discord, Slack, and Telegram

Author: [greencrew](https://greencrew.space/blog)

Tags: Uptime Monitoring, Discord, Slack, Telegram, Alerts, DevOps

Description: To get website downtime alerts on Discord, Slack, or Telegram, create a Discord or Slack incoming webhook (or a Telegram bot token and chat ID), test it with a single curl command, then connect it to an external uptime monitor that checks your site every minute. Includes setup steps, curl tests, and alerting best practices.

Date: 2026-09-30

**Short answer:** create an incoming webhook in Discord (Server Settings, then Integrations, then Webhooks) or Slack (an app with Incoming Webhooks turned on). For Telegram, create a bot with @BotFather and get your chat ID. Send a test message with `curl` to confirm it works. Then connect the webhook to an **external** uptime monitor such as [GreenCrew](https://greencrew.space), which checks your site from outside your network and posts to the channel when it goes down and again when it recovers. This guide walks through each platform, shows the curl commands, and explains the alert settings that separate useful alerts from noise.

## Why Chat Alerts Beat Email for Downtime

Downtime costs money by the minute, and email is where urgent alerts go to wait. Chat apps get push notifications on every device, and they put the alert in front of the whole team at once:

| Channel | Typical time to notice | Team visibility | Best for |
| --- | --- | --- | --- |
| Email | Minutes to hours | One inbox | Daily summaries, reports |
| Slack | Seconds | Whole channel | Engineering teams already on Slack |
| Discord | Seconds | Whole server | Indie hackers, gaming, communities, open source |
| Telegram | Seconds | Personal or group | Solo developers, mobile-first on-call |
| SMS / phone | Seconds | One person | Critical paging, after hours |

Even small outages add up quickly. Here's how much downtime each uptime target allows:

| Uptime target | Downtime per month | Downtime per year |
| --- | --- | --- |
| 99% | ~7.3 hours | ~3.65 days |
| 99.5% | ~3.65 hours | ~1.83 days |
| 99.9% | ~43.8 minutes | ~8.77 hours |
| 99.95% | ~21.9 minutes | ~4.38 hours |
| 99.99% | ~4.4 minutes | ~52.6 minutes |

To meet 99.9%, you need to learn about an outage within a minute or two. If your checks run every 15 minutes, a single outage can use up a third of your monthly budget before anyone knows it started.

## Step 1: Choose What to Monitor

Before you wire up alerts, decide what should trigger them. A useful starting set:

| Monitor type | Example target | Detects |
| --- | --- | --- |
| HTTP(S) status | `https://example.com` | Server down, 5xx errors, DNS failures, TLS errors |
| Health endpoint | `https://example.com/healthz` | App or database failure behind a working proxy |
| Keyword | Homepage must contain `Sign in` | Blank pages or error pages that return 200 |
| API endpoint | `https://api.example.com/v1/status` | Backend outages the homepage hides |

If you run Nginx, our guide on [setting up an Nginx reverse proxy](/blog/how-to-set-up-nginx-reverse-proxy) shows how to add a cheap `/healthz` endpoint for exactly this purpose.

## Step 2a: Set Up Discord Downtime Alerts

Discord webhooks need no bot and no code.

1. **Open channel settings**: right-click your `#alerts` channel and choose **Edit Channel**.
2. **Create the webhook**: go to **Integrations**, then **Webhooks**, then **New Webhook**, and give it a name like "Uptime Monitor".
3. **Copy the URL**: click **Copy Webhook URL**. It looks like `https://discord.com/api/webhooks/<id>/<token>`.
4. **Test it**:

```bash
DISCORD_WEBHOOK="https://discord.com/api/webhooks/XXXX/YYYY"

curl -H "Content-Type: application/json" \
  -d '{"content":"🔴 DOWN: https://example.com returned 502 (test alert)"}' \
  "$DISCORD_WEBHOOK"
```

A message should appear in the channel right away. You need the **Manage Webhooks** permission on the channel to create one.

## Step 2b: Set Up Slack Downtime Alerts

Slack uses incoming webhooks attached to a Slack app.

1. **Create an app**: go to [api.slack.com/apps](https://api.slack.com/apps), click **Create New App**, then **From scratch**, and pick your workspace.
2. **Enable webhooks**: open **Incoming Webhooks** and switch it on.
3. **Add to a channel**: click **Add New Webhook to Workspace** and choose `#alerts`.
4. **Test it**:

```bash
SLACK_WEBHOOK="https://hooks.slack.com/services/T000/B000/XXXX"

curl -X POST -H 'Content-type: application/json' \
  --data '{"text":":red_circle: DOWN: https://example.com returned 502 (test alert)"}' \
  "$SLACK_WEBHOOK"
```

Slack replies with `ok` when the message goes through. Each webhook posts to one channel only, so create a separate webhook for each channel.

## Step 2c: Set Up Telegram Downtime Alerts

Telegram needs a bot token and the ID of the chat that should receive alerts.

1. **Create a bot**: message [@BotFather](https://t.me/BotFather), send `/newbot`, and follow the prompts. Save the token it gives you, which looks like `123456789:AA...`.
2. **Start the chat**: send your bot any message, or add it to a group and send `/start@YourBotName`.
3. **Find the chat ID**:

```bash
TOKEN="123456789:AAxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
curl -s "https://api.telegram.org/bot$TOKEN/getUpdates" | grep -o '"chat":{"id":-\?[0-9]*'
```

Personal chat IDs are positive numbers. Group IDs are negative, and supergroups start with `-100`.

4. **Test it**:

```bash
CHAT_ID="-1001234567890"
curl -s -X POST "https://api.telegram.org/bot$TOKEN/sendMessage" \
  -d chat_id="$CHAT_ID" \
  -d text="🔴 DOWN: https://example.com returned 502 (test alert)"
```

A response containing `"ok":true` means it worked.

## Setup Comparison at a Glance

| | Discord | Slack | Telegram |
| --- | --- | --- | --- |
| What you need | Webhook URL | Webhook URL | Bot token + chat ID |
| Setup time | ~1 minute | ~3 minutes | ~3 minutes |
| Admin rights needed | Manage Webhooks on the channel | Permission to install apps | None |
| Works in groups | Yes | Yes (one channel per webhook) | Yes (add the bot to the group) |
| Mobile push | Yes | Yes | Yes |
| Treat as a secret | Yes | Yes | Yes (the token controls the bot) |

Anyone who has the webhook URL or bot token can post into your channel. Keep them out of Git, and rotate them if they leak.

## Step 3: Connect the Alerts to an External Monitor

You could write a cron script that curls your site and posts to the webhook:

```bash
#!/bin/sh
URL="https://example.com/healthz"
CODE=$(curl -s -o /dev/null -w '%{http_code}' --max-time 10 "$URL")
if [ "$CODE" != "200" ]; then
  curl -s -H "Content-Type: application/json" \
    -d "{\"content\":\"🔴 DOWN: $URL returned $CODE\"}" "$DISCORD_WEBHOOK"
fi
```

It's a fine learning exercise, but it breaks down in production:

| Problem with a DIY script | Why it matters |
| --- | --- |
| Runs on the server it monitors | If the server dies, the monitor dies with it and you get no alert |
| Single location | Can't tell a real outage from a local network blip |
| No state | Alerts every minute during an outage and never sends a "recovered" message |
| No history | No uptime percentage and no incident timeline |
| No status page | Users can't see that you already know about the outage |

An external monitor fixes all five. With [GreenCrew](https://greencrew.space) the setup is:

1. Create an HTTP monitor for `https://example.com` and one for your `/healthz` endpoint.
2. Add your Discord webhook, Slack webhook, or Telegram chat as a notification channel.
3. Publish a **public status page** so customers can check service health themselves instead of flooding your support inbox.

You get one alert when a check fails and one when it recovers, which is far more useful than a script that fires every minute.

## Alerting Best Practices That Prevent Alert Fatigue

Noisy alerts get muted, and muted alerts are no better than none. Follow these rules:

- **Confirm before alerting.** Require a failure to repeat, or to be confirmed from a second location, before you page anyone. One dropped packet shouldn't wake your team.
- **Alert on recovery too.** A "back up" message closes the loop and tells everyone they can stand down.
- **Route by severity.** Production outages go to `#alerts-critical` with mobile push on. Staging goes to a quieter channel.
- **Include context.** The URL, status code, response time, and a link to the status page save a minute of investigation per incident.
- **Monitor your certificates.** An expired certificate is downtime. See [how to install and auto-renew Let's Encrypt on Nginx](/blog/how-to-install-lets-encrypt-ssl-nginx-certbot) to prevent it.
- **Test the alert path monthly.** Webhooks get deleted, bots get removed from groups, and channels get archived. Send a test alert and confirm it arrives.

## Frequently Asked Questions

### Can I send the same downtime alert to Discord, Slack, and Telegram at once?

Yes. Add each one as its own notification channel in your uptime monitor and attach all of them to the monitor. Teams often send alerts to Slack for engineers and Telegram for the on-call phone at the same time.

### How often should an uptime monitor check my website?

Every 1 minute for production sites, where each minute of downtime costs you; every 5 minutes for less important pages. Faster checks shorten time to detection, which matters most when you're working toward a 99.9% or higher uptime target.

### Why did my Discord webhook stop working?

Most often someone deleted the webhook, deleted the channel, or regenerated the URL. Discord then returns `404 Unknown Webhook` or `401`. Create a new webhook and update the URL in your monitor.

### Why doesn't my Telegram bot see messages in a group?

Bots have privacy mode on by default, so in groups they only see commands and mentions. Send `/start@YourBotName` in the group, then call `getUpdates` again to find the group chat ID.

## Conclusion

Setting up downtime alerts on Discord, Slack, or Telegram takes a few minutes: create a webhook or bot, test it with one `curl` command, and connect it to a monitor. The monitor matters most. It has to run outside your infrastructure, confirm failures, report recoveries, and keep a status page up to date. [GreenCrew](https://greencrew.space) handles all of that and sends alerts wherever your team already works, so you hear about the next outage from your monitor instead of your customers.
