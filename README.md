# 🤖 EC2 Discord Controller + EventBridge Scheduler

Control your AWS EC2 instance via Discord slash commands, with automatic start/stop scheduling using EventBridge. Perfect for dev servers that only need to run Mon–Fri, 9am–9pm.

---

## 📋 Table of Contents

- [Architecture](#architecture)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [Part 1 — EventBridge Scheduler](#part-1--eventbridge-scheduler)
- [Part 2 — Discord Bot Setup](#part-2--discord-bot-setup)
- [Part 3 — AWS Lambda Function](#part-3--aws-lambda-function)
- [Part 4 — API Gateway](#part-4--api-gateway)
- [Part 5 — Register Slash Commands](#part-5--register-slash-commands)
- [Part 6 — Connect Discord to API Gateway](#part-6--connect-discord-to-api-gateway)
- [Commands Reference](#commands-reference)
- [Cost Estimate](#cost-estimate)
- [Troubleshooting](#troubleshooting)

---

## Architecture

```
Discord Slash Command
        │
        ▼
  API Gateway (HTTPS)
        │
        ▼
  Lambda Function (Python 3.12)
        │
        ▼
  EC2 Start / Stop / Reboot / Status

EventBridge Scheduler
  → 9:00 AM Mon–Fri  →  EC2 Start
  → 9:00 PM Mon–Fri  →  EC2 Stop
```

---

## Features

| Feature | Details |
|---------|---------|
| ⏰ Auto Start | Every Mon–Fri at 9:00 AM |
| 🛑 Auto Stop | Every Mon–Fri at 9:00 PM |
| 📵 Weekend Off | Saturday & Sunday completely off |
| 🚨 Emergency Control | Start/Stop/Reboot via Discord anytime |
| 📊 Status Report | IP address, uptime, instance type, cost |
| 💰 Cost Tracking | Current session cost + monthly estimate |
| 🔒 Authorization | Only allowed Discord User IDs can control |

---

## Prerequisites

- AWS Account with EC2 instance running
- Discord account + Server
- Python 3.x installed on your PC
- AWS CloudShell access

---

## Part 1 — EventBridge Scheduler

### Create Start Schedule (9 AM Mon–Fri)

1. Go to **AWS Console → EventBridge → Schedules**
2. Click **Create Schedule**
3. Fill in:
   - **Name:** `ec2-start-9am`
   - **Schedule type:** Recurring schedule
   - **Cron expression:** `0 9 ? * MON-FRI *`
   - **Flexible time window:** Off
4. **Target:** AWS API → EC2 → `StartInstances`
5. Add Instance ID in JSON input:
```json
{
  "InstanceIds": ["i-xxxxxxxxxxxxxxxxx"]
}
```
6. Click **Create**

### Create Stop Schedule (9 PM Mon–Fri)

Repeat above with:
- **Name:** `ec2-stop-9pm`
- **Cron expression:** `0 21 ? * MON-FRI *`
- **Target:** EC2 → `StopInstances`

### Cron Reference

| Schedule | Cron Expression |
|----------|----------------|
| Mon–Fri 9 AM | `0 9 ? * MON-FRI *` |
| Mon–Fri 9 PM | `0 21 ? * MON-FRI *` |
| Mon–Sat 9 AM | `0 9 ? * MON-SAT *` |
| Every day 9 AM | `0 9 ? * * *` |

> ⚠️ Times are in UTC. India (IST) = UTC+5:30, so 9 AM IST = 3:30 AM UTC

---

## Part 2 — Discord Bot Setup

### Create Discord Application

1. Go to [discord.com/developers/applications](https://discord.com/developers/applications)
2. Click **New Application** → Name: `EC2 Controller`
3. Go to **General Information** → Copy and save:
   - `Application ID`
   - `Public Key`
4. Go to **Bot** → Click **Reset Token** → Copy `Bot Token`
5. Go to **Bot** → Enable:
   - ✅ Server Members Intent
   - ✅ Message Content Intent

### Add Bot to Your Server

1. Go to **OAuth2 → URL Generator**
2. Check: `applications.commands` + `bot`
3. Bot Permissions: `Send Messages`
4. Copy URL → Open in browser → Select server → **Authorize**

### Get Your Discord User ID

1. Discord → **Settings → Advanced → Enable Developer Mode**
2. Right-click your username → **Copy User ID**

### Get Your Server ID

1. Right-click your server name → **Copy Server ID**

---

## Part 3 — AWS Lambda Function

### Create Function

1. Go to **AWS Lambda → Create Function**
2. Settings:
   - **Name:** `discord-ec2-controller`
   - **Runtime:** Python 3.12
   - **Architecture:** x86_64
3. Click **Create Function**

### Build Deployment Package (in AWS CloudShell)

Open AWS CloudShell (`>_` icon in AWS Console top right) and run:

```bash
# Create project folder
mkdir discord-bot && cd discord-bot

# Install pynacl with correct Linux binary
pip install pynacl \
  --platform manylinux2014_x86_64 \
  --implementation cp \
  --python-version 312 \
  --only-binary=:all: \
  -t .

# Install cffi
pip install cffi \
  --platform manylinux2014_x86_64 \
  --implementation cp \
  --python-version 312 \
  --only-binary=:all: \
  -t .

# Create Lambda function file
cat > lambda_function.py << 'EOF'
import json
import boto3
import os
import nacl.signing
from nacl.exceptions import BadSignatureError
from datetime import datetime, timezone

EC2_REGION         = os.environ['EC2_REGION']
INSTANCE_ID        = os.environ['INSTANCE_ID']
DISCORD_PUBLIC_KEY = os.environ['DISCORD_PUBLIC_KEY']
ALLOWED_USER_IDS   = os.environ['ALLOWED_USER_IDS'].split(',')

ec2 = boto3.client('ec2', region_name=EC2_REGION)

INSTANCE_COSTS = {
    't2.micro':   0.0116, 't2.small':  0.023,  't2.medium': 0.0464,
    't2.large':   0.0928, 't3.micro':  0.0104, 't3.small':  0.0208,
    't3.medium':  0.0416, 't3.large':  0.0832, 't3.xlarge': 0.1664,
    't3a.micro':  0.0094, 't3a.small': 0.0188, 't3a.medium':0.0376,
    'm5.large':   0.096,  'm5.xlarge': 0.192,  'c5.large':  0.085,
    'c5.xlarge':  0.17,   'r5.large':  0.126,  'r5.xlarge': 0.252,
}

def verify_signature(event):
    try:
        body      = event['body']
        timestamp = event['headers'].get('x-signature-timestamp', '')
        signature = event['headers'].get('x-signature-ed25519', '')
        verify_key = nacl.signing.VerifyKey(bytes.fromhex(DISCORD_PUBLIC_KEY))
        verify_key.verify(f"{timestamp}{body}".encode(), bytes.fromhex(signature))
        return True
    except (BadSignatureError, ValueError, Exception):
        return False

def get_status():
    resp     = ec2.describe_instances(InstanceIds=[INSTANCE_ID])
    instance = resp['Reservations'][0]['Instances'][0]

    state         = instance['State']['Name']
    instance_type = instance.get('InstanceType', 'Unknown')
    public_ip     = instance.get('PublicIpAddress', 'No IP (instance stopped)')
    private_ip    = instance.get('PrivateIpAddress', 'N/A')
    launch_time   = instance.get('LaunchTime', None)
    az            = instance.get('Placement', {}).get('AvailabilityZone', 'N/A')

    name = 'N/A'
    for tag in instance.get('Tags', []):
        if tag['Key'] == 'Name':
            name = tag['Value']

    uptime_str = 'N/A'
    if launch_time and state == 'running':
        now    = datetime.now(timezone.utc)
        diff   = now - launch_time
        hours  = int(diff.total_seconds() // 3600)
        mins   = int((diff.total_seconds() % 3600) // 60)
        uptime_str = f"{hours}h {mins}m"

    hourly = INSTANCE_COSTS.get(instance_type, None)
    if hourly:
        daily   = hourly * 10
        monthly = daily * 22
        if launch_time and state == 'running':
            now          = datetime.now(timezone.utc)
            diff         = now - launch_time
            hours_run    = diff.total_seconds() / 3600
            current_cost = hourly * hours_run
            cost_str = (
                f"~${hourly}/hr | ~${daily:.2f}/day | ~${monthly:.2f}/month\n"
                f"💸 **Current Session:** ${current_cost:.4f} "
                f"(running {hours_run:.1f} hrs)"
            )
        else:
            cost_str = f"~${hourly}/hr | ~${daily:.2f}/day | ~${monthly:.2f}/month"
    else:
        cost_str = "N/A"

    emoji = {'running':'🟢','stopped':'🔴','stopping':'🟡','pending':'🟡'}.get(state,'⚪')

    return f"""
{emoji} **EC2 Status Report**
━━━━━━━━━━━━━━━━━━━━
🏷️ **Name:**         {name}
🆔 **Instance ID:**  {INSTANCE_ID}
💻 **Type:**         {instance_type}
📍 **State:**        {state.upper()}
🌍 **Region/AZ:**    {az}
━━━━━━━━━━━━━━━━━━━━
🌐 **Public IP:**    {public_ip}
🔒 **Private IP:**   {private_ip}
⏱️ **Uptime:**       {uptime_str}
━━━━━━━━━━━━━━━━━━━━
💰 **Cost:**         {cost_str}
━━━━━━━━━━━━━━━━━━━━
"""

def start_instance():
    ec2.start_instances(InstanceIds=[INSTANCE_ID])
    return "✅ EC2 is **starting**... wait ~30 seconds\nUse `/status-ec2` to check when it's ready"

def stop_instance():
    ec2.stop_instances(InstanceIds=[INSTANCE_ID])
    return "🛑 EC2 is **stopping**...\nPublic IP will be released"

def reboot_instance():
    ec2.reboot_instances(InstanceIds=[INSTANCE_ID])
    return "🔄 EC2 is **rebooting**... wait ~60 seconds\nUse `/status-ec2` to check when it's back online"

def respond(message):
    return {
        'statusCode': 200,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps({'type': 4, 'data': {'content': message}})
    }

def lambda_handler(event, context):
    if not verify_signature(event):
        return {'statusCode': 401, 'body': json.dumps('Invalid signature')}

    body = json.loads(event['body'])

    if body.get('type') == 1:
        return {'statusCode': 200, 'body': json.dumps({'type': 1})}

    if body.get('type') == 2:
        user_id = body['member']['user']['id']
        command = body['data']['name']

        if user_id not in ALLOWED_USER_IDS:
            return respond("❌ You are **not authorized**!")

        if command == 'start-ec2':
            return respond(start_instance())
        elif command == 'stop-ec2':
            return respond(stop_instance())
        elif command == 'reboot-ec2':
            return respond(reboot_instance())
        elif command == 'status-ec2':
            return respond(get_status())

    return {'statusCode': 400, 'body': 'Bad Request'}
EOF

# Zip everything
zip -r function.zip .
ls -lh function.zip
```

Download the zip:
1. CloudShell → **Actions → Download File**
2. Type: `discord-bot/function.zip`
3. Click **Download**

### Upload to Lambda

1. Lambda → your function → **Code** tab
2. **Upload from → .zip file**
3. Upload `function.zip` → **Save**

### Environment Variables

Go to **Configuration → Environment Variables → Edit → Add:**

| Key | Value |
|-----|-------|
| `EC2_REGION` | e.g. `ap-south-1` |
| `INSTANCE_ID` | e.g. `i-0abc1234567890` |
| `DISCORD_PUBLIC_KEY` | From Discord Developer Portal |
| `ALLOWED_USER_IDS` | Your Discord User ID (comma separated for multiple) |

### IAM Permissions

1. Lambda → **Configuration → Permissions → click Role name**
2. **Add Permissions → Create Inline Policy**
3. Paste JSON:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:StartInstances",
        "ec2:StopInstances",
        "ec2:RebootInstances",
        "ec2:DescribeInstances"
      ],
      "Resource": "*"
    }
  ]
}
```

4. Name it `EC2DiscordControl` → **Create**

### Lambda Settings

Go to **Configuration → General Configuration → Edit:**
- **Timeout:** 10 seconds
- **Memory:** 128 MB (default is fine)

---

## Part 4 — API Gateway

1. Go to **API Gateway → Create API → HTTP API**
2. **Add Integration → Lambda** → select `discord-ec2-controller`
3. **API name:** `discord-ec2-api`
4. **Route:** Method `POST` → Path `/discord`
5. Click **Next → Next → Create**
6. Go to **Stages → $default** → copy **Invoke URL**

Your URL will look like:
```
https://xxxxxxxxxx.execute-api.ap-south-1.amazonaws.com
```

Your full endpoint:
```
https://xxxxxxxxxx.execute-api.ap-south-1.amazonaws.com/discord
```

---

## Part 5 — Register Slash Commands

Create `register.py` on your PC:

```python
import requests

BOT_TOKEN      = "YOUR_BOT_TOKEN"
APPLICATION_ID = "YOUR_APPLICATION_ID"
GUILD_ID       = "YOUR_SERVER_ID"

url = f"https://discord.com/api/v10/applications/{APPLICATION_ID}/guilds/{GUILD_ID}/commands"
headers = {"Authorization": f"Bot {BOT_TOKEN}"}

commands = [
    {"name": "start-ec2",  "description": "Start the EC2 instance"},
    {"name": "stop-ec2",   "description": "Stop the EC2 instance"},
    {"name": "reboot-ec2", "description": "Reboot the EC2 instance"},
    {"name": "status-ec2", "description": "Check EC2 status, IP and cost"},
]

for cmd in commands:
    r = requests.post(url, headers=headers, json=cmd)
    print(f"{cmd['name']}: {r.status_code}")
```

Run:
```bash
pip install requests
python register.py
```

Expected output:
```
start-ec2:  201
stop-ec2:   201
reboot-ec2: 201
status-ec2: 201
```

---

## Part 6 — Connect Discord to API Gateway

1. Go to **Discord Developer Portal → your app**
2. Go to **General Information**
3. Paste in **Interactions Endpoint URL:**
```
https://xxxxxxxxxx.execute-api.ap-south-1.amazonaws.com/discord
```
4. Click **Save Changes** → should show ✅

---

## Commands Reference

| Command | Description |
|---------|-------------|
| `/start-ec2` | Start the EC2 instance |
| `/stop-ec2` | Stop the EC2 instance |
| `/reboot-ec2` | Reboot the EC2 instance |
| `/status-ec2` | Full status: IP, uptime, cost |

### Example `/status-ec2` Output

```
🟢 EC2 Status Report
━━━━━━━━━━━━━━━━━━━━
🏷️ Name:         MyServer
🆔 Instance ID:  i-0abc1234567890
💻 Type:         t3.medium
📍 State:        RUNNING
🌍 Region/AZ:    ap-south-1a
━━━━━━━━━━━━━━━━━━━━
🌐 Public IP:    13.235.xx.xx
🔒 Private IP:   172.31.xx.xx
⏱️ Uptime:       3h 24m
━━━━━━━━━━━━━━━━━━━━
💰 Cost: ~$0.04/hr | ~$0.42/day | ~$9.15/month
💸 Current Session: $0.1386 (running 3.4 hrs)
━━━━━━━━━━━━━━━━━━━━
```

---

## Schedule Summary

```
Monday    → Auto ON 9:00 AM ✅  Auto OFF 9:00 PM ✅
Tuesday   → Auto ON 9:00 AM ✅  Auto OFF 9:00 PM ✅
Wednesday → Auto ON 9:00 AM ✅  Auto OFF 9:00 PM ✅
Thursday  → Auto ON 9:00 AM ✅  Auto OFF 9:00 PM ✅
Friday    → Auto ON 9:00 AM ✅  Auto OFF 9:00 PM ✅
Saturday  → ❌ OFF all day
Sunday    → ❌ OFF all day

Emergency anytime → Discord commands 🚨
```

---

## Cost Estimate

| Service | Free Tier | Monthly Cost |
|---------|-----------|-------------|
| Lambda | 1M requests free | $0.00 |
| API Gateway | 1M requests free | $0.00 |
| EventBridge | 14M invocations free | $0.00 |
| EC2 | Depends on instance type | Varies |

> With 100 Discord commands/month you use 0.01% of the free tier.

---

## Troubleshooting

### ❌ `invalid ELF header` on Lambda
**Cause:** pynacl built on macOS, not Linux.  
**Fix:** Rebuild using AWS CloudShell with `--platform manylinux2014_x86_64` flag.

### ❌ `No module named '_cffi_backend'`
**Cause:** cffi C extension missing.  
**Fix:** Install cffi with the same `--platform` flags in CloudShell.

### ❌ Discord: `interactions endpoint url could not be verified`
**Cause 1:** URL missing `/discord` at the end.  
**Fix:** Make sure URL ends with `/discord`

**Cause 2:** Lambda import error causing 500 response.  
**Fix:** Check CloudWatch logs → fix the import error first.

### ❌ `You are not authorized!` in Discord
**Cause:** Discord User ID in `ALLOWED_USER_IDS` env var is wrong.  
**Fix:** Temporarily update the error message to show the actual user ID:
```python
return respond(f"❌ Not authorized! Your ID: `{user_id}`")
```
Copy the ID shown → paste into `ALLOWED_USER_IDS` → revert code.

### ❌ `KeyError: 'body'`
**Cause:** Test event format is wrong.  
**Fix:** Use this test event format:
```json
{
  "version": "2.0",
  "headers": {
    "x-signature-ed25519": "aabbccdd",
    "x-signature-timestamp": "123456"
  },
  "body": "{\"type\": 1}",
  "isBase64Encoded": false
}
```
Expected result: `{"statusCode": 401}` ✅

---

## Values Reference

| Value | Where to Find |
|-------|--------------|
| `Application ID` | Discord → App → General Information |
| `Public Key` | Discord → App → General Information |
| `Bot Token` | Discord → App → Bot → Reset Token |
| `Discord User ID` | Discord → Right-click username → Copy User ID |
| `Server ID` | Discord → Right-click server → Copy Server ID |
| `Instance ID` | AWS → EC2 → Instances → your instance |
| `EC2 Region` | AWS → top right corner |
| `API Gateway URL` | API Gateway → your API → Stages → $default → Invoke URL |

---

## Tech Stack

- **AWS EventBridge** — Scheduled cron jobs
- **AWS Lambda** — Serverless function (Python 3.12)
- **AWS API Gateway** — HTTP endpoint for Discord
- **Discord API** — Slash commands & bot
- **pynacl** — Ed25519 signature verification

---

*Built with ❤️ — Total monthly cost for Discord commands: $0.00*
