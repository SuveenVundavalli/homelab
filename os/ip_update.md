# ☁️ Dynamic DNS with Cloudflare for `bujji.suveen.me`

This guide sets up Dynamic DNS (DDNS) using a bash script to update the A record for `bujji.suveen.me` on Cloudflare whenever your Ubuntu server's **public IP changes**.

---

## 🔐 Prerequisites

- Domain `suveen.me` is managed by [Cloudflare](https://cloudflare.com)
- Subdomain `bujji.suveen.me` already exists as an A record (you can create it initially with a dummy IP)
- Ubuntu server has curl and cron
- You have a **Cloudflare API Token** with the following permissions:

  - Zone > Zone Settings: Read
  - Zone > DNS: Edit

---

## 🔁 Step 1: Get Required Info from Cloudflare

You'll need:

1. **Zone ID** (for `suveen.me`)
2. **Record ID** (for the A record of `bujji.suveen.me`)
3. **API Token**

### a. Get Zone ID

```bash
curl -s -X GET "https://api.cloudflare.com/client/v4/zones" \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  -H "Content-Type: application/json"
```

Find the zone with `"name": "suveen.me"` and copy its `id`.

### b. Get DNS Record ID

```bash
curl -s -X GET "https://api.cloudflare.com/client/v4/zones/YOUR_ZONE_ID/dns_records?type=A&name=bujji.suveen.me" \
  -H "Authorization: Bearer YOUR_API_TOKEN" \
  -H "Content-Type: application/json"
```

Find the `id` of the DNS record.

---

## 🔧 Step 2: Create the DDNS Script

Create `/usr/local/bin/cloudflare-ddns.sh`:

```bash
#!/bin/bash

# Config
API_TOKEN="YOUR_API_TOKEN"
ZONE_NAME="suveen.me"
RECORD_NAME="bujji.suveen.me"
LOG_FILE="/var/log/cloudflare-ddns.log"

# Get current public IP
IP=$(curl -s https://api.ipify.org)

# Get current DNS record IP
DNS_IP=$(dig +short "$RECORD_NAME" @1.1.1.1)

# Skip if IP hasn't changed
if [ "$IP" = "$DNS_IP" ]; then
  echo "[$(date)] IP unchanged: $IP" >> "$LOG_FILE"
  exit 0
fi

# Get Zone ID
ZONE_ID=$(curl -s -X GET "https://api.cloudflare.com/client/v4/zones?name=$ZONE_NAME" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json" \
  | jq -r '.result[0].id')

# Get Record ID
RECORD_ID=$(curl -s -X GET "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records?type=A&name=$RECORD_NAME" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json" \
  | jq -r '.result[0].id')

# Update the record
RESPONSE=$(curl -s -X PUT "https://api.cloudflare.com/client/v4/zones/$ZONE_ID/dns_records/$RECORD_ID" \
  -H "Authorization: Bearer $API_TOKEN" \
  -H "Content-Type: application/json" \
  --data '{
    "type": "A",
    "name": "'"$RECORD_NAME"'",
    "content": "'"$IP"'",
    "ttl": 1,
    "proxied": false
  }')

# Log the result
if echo "$RESPONSE" | grep -q '"success":true'; then
  echo "[$(date)] Updated $RECORD_NAME to $IP" >> "$LOG_FILE"
else
  echo "[$(date)] Failed to update IP: $RESPONSE" >> "$LOG_FILE"
fi
```

Make it executable:

```bash
sudo chmod +x /usr/local/bin/cloudflare-ddns.sh
```

---

## 📂 Step 3: Prepare Logging Directory

```bash
sudo touch /var/log/cloudflare-ddns.log
sudo touch /var/log/suveen-backups/susra-nextcloud.log
sudo chown $(whoami):$(whoami) /var/log/cloudflare-ddns.log
```

---

## 🕒 Step 4: Set up Cron Job

Edit crontab:

```bash
crontab -e
```

Add this line to check for IP changes every 5 minutes:

```cron
*/5 * * * * /usr/local/bin/cloudflare-ddns.sh
```

---

## 🚀 Done!

Your server will now automatically update the A record for `bujji.suveen.me` every 5 minutes **only if** the IP changes.

Logs are stored in:

```
/var/log/cloudflare-ddns.log
```

---

## 🔐 Security Tip

- Store the script config (token, IDs) in a separate config file with restricted permissions
- Or use environment variables loaded via `source` in the script

Let me know if you want to use `systemd` instead of cron or rotate logs with `logrotate`.
