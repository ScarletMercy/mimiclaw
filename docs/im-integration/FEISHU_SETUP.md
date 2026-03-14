# Feishu / Lark Bot Configuration Guide

This guide walks through setting up a Feishu (or Lark) bot to work with MimiClaw, turning your ESP32-S3 into a Feishu-connected AI assistant.

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Step 1: Create a Feishu App](#step-1-create-a-feishu-app)
- [Step 2: Configure App Permissions](#step-2-configure-app-permissions)
- [Step 3: Enable Event Subscription](#step-3-enable-event-subscription)
- [Step 4: Configure MimiClaw](#step-4-configure-mimiclaw)
- [Step 5: Publish and Test](#step-5-publish-and-test)
- [Architecture](#architecture)
- [CLI Commands](#cli-commands)
- [Troubleshooting](#troubleshooting)
- [References](#references)

## Overview

MimiClaw supports Feishu as a messaging channel alongside Telegram and WebSocket. The Feishu integration uses:

- **WebSocket long connection** — ESP32 establishes a persistent WebSocket connection to receive messages in real-time (no public IP required)
- **Send API** — MimiClaw sends replies via Feishu's REST API (`/im/v1/messages`)
- **Tenant access token** — automatic token management with background refresh
- **Rate limiting** — automatic QPS throttling (5 QPS per user/group, per API limits)
- **Message deduplication** — UUID-based deduplication to prevent duplicate sends

Both **direct messages (P2P)** and **group chats** are supported.

> **Note**: MimiClaw uses WebSocket mode instead of Webhook mode. This means you don't need to configure port forwarding or public IP - the ESP32 actively connects to Feishu servers.

## Prerequisites

- A Feishu account (sign up at [feishu.cn](https://www.feishu.cn)) or a Lark account ([larksuite.com](https://www.larksuite.com))
- Admin access to create apps on [Feishu Open Platform](https://open.feishu.cn/) (or [Lark Developer](https://open.larksuite.com/))
- MimiClaw flashed on an ESP32-S3 with network access
- The ESP32 must be reachable from the internet (for WebSocket connection only)

## Step 1: Create a Feishu App

1. Go to [Feishu Open Platform](https://open.feishu.cn/) and sign in
2. Click **Create Custom App** (or "Create App" on Lark)
3. Fill in app details:
   - **App Name**: Choose a name (e.g., "MimiClaw Bot")
   - **App Description**: Brief description of your bot
   - **App Icon**: Upload an icon (optional)
4. After creation, you will see your **App ID** and **App Secret** on the app's **Credentials & Basic Info** page

> **Important:** Save **App ID** (`cli_xxxxxxxxxxxxxx`) and **App Secret** (`xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx`). You will need these to configure MimiClaw.

## Step 2: Configure App Permissions

In your app's settings, go to **Permissions & Scopes** and add these required permissions:

| Permission | Scope ID | Description |
|-----------|----------|-------------|
| Read/Send messages | `im:message` | Receive and send messages |
| Send messages as bot | `im:message:send_as_bot` | Send messages as bot identity |

To add permissions:

1. Navigate to **Permissions & Scopes** in the left sidebar
2. Search for each scope ID listed above
3. Click **Add** next to each permission
4. The permissions will take effect after you publish or update the app version

> **Note:** On Lark (international version), permission names may differ slightly, but scope IDs are the same.

## Step 3: Enable Event Subscription

WebSocket mode requires event subscription to be enabled in your Feishu app.

### Subscribe to Events

Add the following event:

| Event | Event ID | Description |
|-------|----------|-------------|
| Receive messages | `im.message.receive_v1` | Triggered when users send messages to bot |

To add events:

1. In **Event Subscriptions** page, click **Add Event**
2. Search for `im.message.receive_v1`
3. Select it and click **Confirm**

> **Note**: No URL configuration is needed for WebSocket mode. MimiClaw automatically establishes a secure connection to Feishu servers.

## Step 4: Configure MimiClaw

You need to provide **App ID** and **App Secret** to MimiClaw.

### Option 1: Build-time Configuration

1. Copy the secrets template if you haven't already:

```bash
cp main/mimi_secrets.h.example main/mimi_secrets.h
```

2. Edit `main/mimi_secrets.h`:

```c
#define MIMI_SECRET_FEISHU_APP_ID     "cli_xxxxxxxxxxxxxx"
#define MIMI_SECRET_FEISHU_APP_SECRET "xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx"
```

3. Rebuild and flash:

```bash
idf.py fullclean && idf.py build
idf.py -p PORT flash monitor
```

### Option 2: Runtime Configuration via Serial CLI

Connect to the UART (COM) port and run:

```
mimi> set_feishu_creds cli_xxxxxxxxxxxxxx xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
```

This saves credentials to NVS flash immediately — no rebuild needed.

### Verify Configuration

```
mimi> config_show
```

You should see `feishu_app_id: cli_****` and `feishu_app_secret: ****` in the output.

## Step 5: Publish and Test

### Enable Bot

1. In your app settings, go to **Bot** in the left sidebar
2. Toggle **Enable Bot** to ON

### Publish App

1. Go to **App Release** (or "Version Management & Release")
2. Click **Create Version**
3. Set a version number and description
4. Click **Submit for Review** (for enterprise apps) or **Publish** (for personal testing apps)

> **Tip:** For testing, you can use the app in "development mode" without publishing — just add yourself as a test user.

### Test Bot

1. Open Feishu and search for your bot by name
2. Send a message to the bot in a direct chat
3. Check the ESP32 serial output — you should see the message being received and processed
4. The bot should reply through Feishu

For group chats:

1. Add the bot to a group
2. Mention the bot with `@BotName` followed by your message
3. The bot will process and reply in the group

## Architecture

```
Feishu Cloud
     |
     |  WebSocket Event Push
     |  (im.message.receive_v1)
     v
[ESP32 WebSocket Client]
     |
     |  message_bus_push_inbound()
     v
[Message Bus] ──> [Agent Loop] ──> [Message Bus]
                    (Claude/GPT)         |
                                         |  outbound dispatch
                                         v
                               [feishu_send_message()]
                                         |
                                         |  POST /im/v1/messages
                                         |  (rate-limited, UUID-deduped)
                                         v
                                    Feishu API
```

### Key Components

| Component | Description |
|-----------|-------------|
| **WebSocket Client** | Persistent secure connection to Feishu event servers, handles protobuf frame parsing |
| **Token Manager** | Manages tenant access tokens, auto-refreshes before expiry |
| **Message Sender** | Sends text messages via Feishu REST API with auto-chunking and rate limiting |
| **Deduplication** | Prevents processing duplicate events from Feishu retries (64-item cache) |
| **Rate Limiter** | Automatic QPS throttling (5 QPS per user/group, per API limits) |
| **UUID Generator** | Generates unique UUIDs for each message to prevent duplicate sends |

### Configuration Constants

These can be found in `main/mimi_config.h`:

| Constant | Default | Description |
|----------|---------|-------------|
| `MIMI_FEISHU_MAX_MSG_LEN` | 4096 | Max message length per chunk |
| `MIMI_FEISHU_POLL_STACK` | 12 KB | WebSocket task stack size |
| `MIMI_FEISHU_POLL_PRIO` | 5 | WebSocket task priority |
| `MIMI_FEISHU_POLL_CORE` | 0 | WebSocket task core (Core 0 for I/O) |

## CLI Commands

| Command | Description |
|---------|-------------|
| `set_feishu_creds <app_id> <app_secret>` | Save Feishu credentials to NVS |
| `feishu_send <open_id|chat_id> "<text>"` | Send test message to Feishu |
| `config_show` | Show all configuration (including Feishu, masked) |
| `config_reset` | Clear all NVS config, revert to build-time defaults |

## Troubleshooting

### WebSocket connection fails

- Ensure ESP32 is running and connected to WiFi: `wifi_status`
- Check that ESP32 can reach `https://open.feishu.cn` (try web search if possible)
- Look at ESP32 serial output for WebSocket connection logs
- Ensure Feishu bot is enabled in app settings

### Bot doesn't respond to messages

1. **Check credentials**: `config_show` should show Feishu app_id and app_secret
2. **Check event subscription**: Ensure `im.message.receive_v1` is subscribed in Feishu app settings
3. **Check permissions**: Both `im:message` and `im:message:send_as_bot` must be granted
4. **Check serial output**: Look for message processing logs on ESP32

### "Tenant access token" errors

- Verify your App ID and App Secret are correct
- The token auto-refreshes every 2 hours — if you just set credentials, wait a moment for first token fetch
- Ensure ESP32 can reach `https://open.feishu.cn` (check proxy settings if needed)

### Rate limiting errors

- If you see error code 230020, the rate limit (5 QPS) has been reached
- MimiClaw automatically backs off and retries, but you may need to reduce message frequency
- Consider increasing the cooldown period for high-volume applications

### Messages are truncated

Feishu has a 4096-character limit per message. MimiClaw automatically chunks long messages, but if you see issues, check serial output for chunking errors.

### Bot works in DM but not in groups

- Ensure the bot is added to the group
- Users must `@mention` the bot in group chats for it to receive messages
- Check that group messaging permissions are enabled in the Feishu app settings

### Event subscription shows errors in Feishu console

- Feishu retries failed events up to 5 times with exponential backoff
- MimiClaw deduplicates retried events, so duplicate processing is not a concern
- If events consistently fail, check ESP32's network connectivity

## References

- [Feishu Open Platform Documentation](https://open.feishu.cn/document/home/index)
- [Feishu Bot Development Guide](https://open.feishu.cn/document/client-docs/bot-v3/bot-overview)
- [Feishu Message API](https://open.feishu.cn/document/server-docs/im-v1/message/create)
- [Feishu Event Subscription Guide](https://open.feishu.cn/document/server-docs/event-subscription/event-subscription-guide)
- [Lark Developer Documentation](https://open.larksuite.com/document/home/index) (international version)
