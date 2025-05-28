# Aztec Node Sync Monitor & Telegram Alerts – Setup Guide

This guide helps you set up an automatic sync monitor for your Aztec node that sends alerts via Telegram when sync succeeds or failsbased on a script that checks the sync status of [CerberusNode](https://github.com/cerberus-node)

<img src="https://github.com/user-attachments/assets/5a80b92a-c925-4c10-8925-38bf36cbe67c" width="300">

---

# 🛠 Prerequisites

Ensure the following tools are installed on your system:

- `curl`: For making HTTP requests.
- `jq`: For parsing JSON responses.

Install them using:

```bash
sudo apt-get install curl jq
```

---

## 🤖 Step 1: Create a Telegram Bot

1. Open Telegram and search for `@BotFather`.
2. Run `/newbot` and follow the instructions.
3. Save your **Bot Token** (e.g., `123456789:ABCDefgh`).

---

## 🆔 Step 2: Get Your Telegram Chat ID

1. Send a message like `/start` to your bot.
2. On your server, run the following (replace `<YOUR_BOT_TOKEN>` with your real token):

```bash
curl -s "https://api.telegram.org/bot<YOUR_BOT_TOKEN>/getUpdates"
```

3. In the JSON response, look for:

```json
"chat":{"id":6451081221,...}
```

That number is your **Chat ID**.

---

## 📋 Step 3: Create and Configure the Script

The `sync_status.sh` script monitors your Aztec node's sync status and sends Telegram alerts. Follow these steps to set it up:

1. Create a new file named `sync_status.sh`:

```bash
nano sync_status.sh
```

2. Copy and paste the following script content:

```bash
#!/bin/bash

# Replace with your Telegram Bot Token
TELEGRAM_TOKEN="<BOT_TOKEN>"
# Replace with your Telegram Chat ID
CHAT_ID="<CHAT_ID>"

if [ -z "$TELEGRAM_TOKEN" ] || [ -z "$CHAT_ID" ] || [ "$TELEGRAM_TOKEN" == "<BOT_TOKEN>" ] || [ "$CHAT_ID" == "<CHAT_ID>" ]; then
  echo "Error: Please edit this script and replace <BOT_TOKEN> and <CHAT_ID> with your actual values"
  exit 1
fi

REMOTE_RPC="https://aztec-rpc.cerberusnode.com"
LOCAL_RPC="http://localhost:8080"

# Check local node sync status
LOCAL_RESPONSE=$(curl -s -m 5 -X POST -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"node_getL2Tips","params":[],"id":1}' "$LOCAL_RPC")

if [ -z "$LOCAL_RESPONSE" ] || [[ "$LOCAL_RESPONSE" == *"error"* ]]; then
  TEXT="🚫 Error: Unable to reach the local Aztec node or received an invalid response."
  curl -s -X POST "https://api.telegram.org/bot$TELEGRAM_TOKEN/sendMessage" \
    -d chat_id="$CHAT_ID" \
    -d text="$TEXT" >/dev/null
  exit 0
fi

LOCAL_BLOCK=$(echo "$LOCAL_RESPONSE" | jq -r ".result.proven.number")

# Check remote node status
REMOTE_RESPONSE=$(curl -s -m 5 -X POST -H 'Content-Type: application/json' \
  -d '{"jsonrpc":"2.0","method":"node_getL2Tips","params":[],"id":1}' "$REMOTE_RPC")

if [ -z "$REMOTE_RESPONSE" ] || [[ "$REMOTE_RESPONSE" == *"error"* ]]; then
  TEXT="⚠️ Warning: Remote Aztec RPC is unreachable or returned an error."
  curl -s -X POST "https://api.telegram.org/bot$TELEGRAM_TOKEN/sendMessage" \
    -d chat_id="$CHAT_ID" \
    -d text="$TEXT" >/dev/null
  exit 0
fi

REMOTE_BLOCK=$(echo "$REMOTE_RESPONSE" | jq -r ".result.proven.number")

# Check sync status
if [ "$LOCAL_BLOCK" == "$REMOTE_BLOCK" ]; then
  TEXT="✅ Sync complete!%0A🔸 Local block: $LOCAL_BLOCK%0A🔸 Remote block: $REMOTE_BLOCK%0A📦 Your Aztec node is fully up-to-date."
else
  TEXT="⏳ Sync in progress...%0A📍 Local block: $LOCAL_BLOCK%0A🌐 Remote block: $REMOTE_BLOCK%0A⚡ Your Aztec node is still catching up. Please investigate if the delay persists."
fi

curl -s -X POST "https://api.telegram.org/bot$TELEGRAM_TOKEN/sendMessage" \
  -d chat_id="$CHAT_ID" \
  -d text="$TEXT" >/dev/null
```

3. **Important**: Edit these two lines with your actual values:
   - Replace `<BOT_TOKEN>` with your Telegram Bot Token from Step 1
   - Replace `<CHAT_ID>` with your Telegram Chat ID from Step 2
- 🔁 You can also change the default REMOTE_RPC value:
   - The default is https://aztec-rpc.cerberusnode.com, which is currently stable.
   - Change it if you encounter connection errors, stale block numbers, or slow responses.


4. Save the file and exit:
   - Press `Ctrl + O` to save
   - Press `Enter` to confirm
   - Press `Ctrl + X` to exit

5. Make the script executable:

```bash
chmod +x sync_status.sh
```

6. Run the script manually to test:

```bash
./sync_status.sh
```

---

## ⏱ Step 5: Automate with Cron

### 1. Open your crontab:

```bash
crontab -e
```

### 2. Select an editor

```bash
Select 1 (/bin/nano)
```

### 3. Add one of the following lines depending on your preferred schedule:

| Schedule Description       | Cron Expression                                                                     |
| -------------------------- | ----------------------------------------------------------------------------------- |
| Every 5 minutes            | `*/5 * * * * /root/sync_status.sh`                                              |
| Every 15 minutes           | `*/15 * * * * /root/sync_status.sh`                                             |
| Twice an hour (0, 30 min)  | `0,30 * * * * /root/sync_status.sh`                                             |
| Every hour (at minute 0)   | `0 * * * * /root/sync_status.sh`                                                |
| Daily at 2:00 AM           | `0 2 * * * /root/sync_status.sh`                                                |

✨ **To save and exit** when editing crontab:

* Press `Ctrl + O`, `Enter`, then `Ctrl + X`.


---

## 🏁 You're Done!

Your Aztec node is now being monitored!
You’ll receive Telegram alerts based on its sync status — whether it's in sync or falls out of sync.
