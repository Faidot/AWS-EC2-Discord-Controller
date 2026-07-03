# 🤖 AWS EC2 Multi-Instance Discord Controller + EventBridge Scheduler

Control **multiple AWS EC2 instances** (in one AWS account) via Discord slash commands with friendly names, plus automatic start/stop scheduling using EventBridge.

Perfect for teams running several dev/game/build servers that only need to run on a schedule — with emergency manual control from Discord anytime.

---

## 📋 Table of Contents

- [Architecture](#architecture)
- [Features](#features)
- [Prerequisites](#prerequisites)
- [Part 1 — Plan Your Instance Map](#part-1--plan-your-instance-map)
- [Part 2 — EventBridge Scheduler (Multi-Instance)](#part-2--eventbridge-scheduler-multi-instance)
- [Part 3 — Discord Bot Setup](#part-3--discord-bot-setup)
- [Part 4 — AWS Lambda Function (Multi-Instance)](#part-4--aws-lambda-function-multi-instance)
- [Part 5 — API Gateway](#part-5--api-gateway)
- [Part 6 — Register Slash Commands](#part-6--register-slash-commands)
- [Part 7 — Connect Discord to API Gateway](#part-7--connect-discord-to-api-gateway)
- [Commands Reference](#commands-reference)
- [Adding / Removing Instances Later](#adding--removing-instances-later)
- [Cost Estimate](#cost-estimate)
- [Troubleshooting](#troubleshooting)
- [Values Reference](#values-reference)

---

## Architecture

```
Discord Slash Command  (/start-ec2 name:gameserver)
        │
        ▼
  API Gateway (HTTPS POST /discord)
        │
        ▼
  Lambda Function (Python 3.12)
        │  resolves friendly name → instance ID via INSTANCE_MAP
        ▼
  EC2 Start / Stop / Reboot / Status / Details
  (one instance, or ALL instances at once)

EventBridge Scheduler (per group)
  → weekday-group:  Mon–Fri 9 AM start, 9 PM stop
  → always-on-group: no schedule (manual only)
```

**Key difference from single-instance version:** instead of one hardcoded
`INSTANCE_ID`, the bot uses an `INSTANCE_MAP` — a JSON mapping of
**friendly names → instance IDs**. Every Discord command takes a required
`name` option (rendered as a dropdown), so you never type raw instance IDs.

---

## Features

| Feature                  | Details                                                        |
| ------------------------ | -------------------------------------------------------------- |
| 🗺️ Multi-Instance         | Manage unlimited instances via friendly names (`INSTANCE_MAP`) |
| 🎛️ Dropdown Selection     | Discord shows a picker — no typos, no raw IDs                  |
| 🌐 `all` Target           | Start / stop / reboot / status **every** instance in one command |
| ⏰ Group Scheduling       | Different EventBridge schedules per instance group             |
| 📊 Compact Status         | One-line-per-instance fleet overview                           |
| 🔍 Deep Details           | `/details-ec2` → volumes, SGs, AMI, key pair, tags, monitoring |
| 💰 Cost Tracking          | Per-instance hourly / daily / monthly + current session cost   |
| 🔒 Authorization          | Only whitelisted Discord User IDs can run commands             |
| 🛡️ Scoped IAM             | Start/Stop/Reboot restricted to your instance ARNs only        |
| ✅ Signature Verification | Ed25519 verification of every Discord request (pynacl)         |

---

## Prerequisites

- AWS Account with **two or more EC2 instances**
- Discord account + a server where you have admin rights
- Python 3.x installed on your PC (for registering commands)
- AWS CloudShell access (for building the Lambda package)

---

## Part 1 — Plan Your Instance Map

Before touching AWS, decide your friendly names. Rules:

- lowercase, no spaces (use `-` if needed)
- short and memorable — these appear in the Discord dropdown
- `all` is **reserved** — do not name an instance `all`

Example map (you will paste this JSON into a Lambda env var later):

```json
{
  "gameserver": "i-0aaa1111111111111",
  "devbox":     "i-0bbb2222222222222",
  "gitlab":     "i-0ccc3333333333333",
  "buildagent": "i-0ddd4444444444444"
}
```

Also decide your **schedule groups**, e.g.:

| Group        | Instances               | Schedule            |
| ------------ | ----------------------- | ------------------- |
| `weekday`    | gameserver, devbox      | Mon–Fri 9 AM – 9 PM |
| `always-on`  | gitlab                  | none (manual only)  |
| `on-demand`  | buildagent              | none (manual only)  |

---

## Part 2 — EventBridge Scheduler (Multi-Instance)

One schedule can target **multiple instances at once** — just list all IDs
in the JSON input. Create one start + one stop schedule **per group**.

### Create Start Schedule for the `weekday` group

1. Go to **AWS Console → EventBridge → Schedules**
2. Click **Create Schedule**
3. Fill in:
   - **Name:** `ec2-weekday-start-9am`
   - **Schedule type:** Recurring schedule
   - **Cron expression:** `0 9 ? * MON-FRI *`
   - **Flexible time window:** Off
4. **Target:** AWS API → EC2 → `StartInstances`
5. JSON input — **all instance IDs of this group**:

```json
{
  "InstanceIds": [
    "i-0aaa1111111111111",
    "i-0bbb2222222222222"
  ]
}
```

6. Click **Create**

### Create Stop Schedule for the `weekday` group

Repeat with:

- **Name:** `ec2-weekday-stop-9pm`
- **Cron expression:** `0 21 ? * MON-FRI *`
- **Target:** EC2 → `StopInstances`
- Same `InstanceIds` JSON

### Instances with different hours?

Create additional schedule pairs, e.g. `ec2-nightbatch-start-11pm` /
`ec2-nightbatch-stop-5am`, each with its own `InstanceIds` list.
Instances in the `always-on` / `on-demand` groups simply get **no schedule**.

### Cron Reference

| Schedule        | Cron Expression       |
| --------------- | --------------------- |
| Mon–Fri 9 AM    | `0 9 ? * MON-FRI *`   |
| Mon–Fri 9 PM    | `0 21 ? * MON-FRI *`  |
| Mon–Sat 9 AM    | `0 9 ? * MON-SAT *`   |
| Every day 9 AM  | `0 9 ? * * *`         |
| Every day 11 PM | `0 23 ? * * *`        |

> ⚠️ EventBridge cron uses the **timezone you select** in the schedule
> settings. Pick your local timezone (e.g. `Asia/Karachi`) in the
> "Timezone" dropdown instead of doing UTC math in your head.

---

## Part 3 — Discord Bot Setup

### Create Discord Application

1. Go to [discord.com/developers/applications](https://discord.com/developers/applications)
2. Click **New Application** → Name: `EC2 Fleet Controller`
3. **General Information** → copy and save:
   - `Application ID`
   - `Public Key`
4. **Bot** → **Reset Token** → copy `Bot Token` (shown only once!)
5. **Bot** → Enable:
   - ✅ Server Members Intent
   - ✅ Message Content Intent

### Add Bot to Your Server

1. **OAuth2 → URL Generator**
2. Scopes: `applications.commands` + `bot`
3. Bot Permissions: `Send Messages`
4. Copy URL → open in browser → select server → **Authorize**

### Get Your Discord User ID

1. Discord → **Settings → Advanced → Enable Developer Mode**
2. Right-click your username → **Copy User ID**

### Get Your Server ID

1. Right-click your server name → **Copy Server ID**

---

## Part 4 — AWS Lambda Function (Multi-Instance)

### Create Function

1. **AWS Lambda → Create Function**
2. Settings:
   - **Name:** `discord-ec2-fleet-controller`
   - **Runtime:** Python 3.12
   - **Architecture:** x86_64
3. Click **Create Function**

### Build Deployment Package (in AWS CloudShell)

Open AWS CloudShell (`>_` icon, top right of AWS Console) and run:

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
```

Now create the Lambda function file:

```bash
cat > lambda_function.py << 'EOF'
import json
import boto3
import os
import nacl.signing
from nacl.exceptions import BadSignatureError
from datetime import datetime, timezone

# ─────────────────────────── Configuration ───────────────────────────
EC2_REGION         = os.environ['EC2_REGION']
DISCORD_PUBLIC_KEY = os.environ['DISCORD_PUBLIC_KEY']
ALLOWED_USER_IDS   = [u.strip() for u in os.environ['ALLOWED_USER_IDS'].split(',')]

# JSON map: friendly name -> instance ID
# e.g. {"gameserver": "i-0aaa...", "devbox": "i-0bbb..."}
INSTANCE_MAP = json.loads(os.environ['INSTANCE_MAP'])
# reverse map: instance ID -> friendly name (for status output)
ID_TO_NAME   = {v: k for k, v in INSTANCE_MAP.items()}

ec2 = boto3.client('ec2', region_name=EC2_REGION)

DISCORD_MSG_LIMIT = 2000  # Discord hard limit per message

# On-demand hourly prices (USD) — adjust for your region if needed
INSTANCE_COSTS = {
    't2.micro':   0.0116, 't2.small':  0.023,  't2.medium': 0.0464,
    't2.large':   0.0928, 't3.micro':  0.0104, 't3.small':  0.0208,
    't3.medium':  0.0416, 't3.large':  0.0832, 't3.xlarge': 0.1664,
    't3a.micro':  0.0094, 't3a.small': 0.0188, 't3a.medium':0.0376,
    'm5.large':   0.096,  'm5.xlarge': 0.192,  'c5.large':  0.085,
    'c5.xlarge':  0.17,   'r5.large':  0.126,  'r5.xlarge': 0.252,
}

STATE_EMOJI = {
    'running': '🟢', 'stopped': '🔴', 'stopping': '🟡',
    'pending': '🟡', 'shutting-down': '🟠', 'terminated': '⚫',
}

# ─────────────────────────── Security ───────────────────────────
def verify_signature(event):
    try:
        body      = event['body']
        timestamp = event['headers'].get('x-signature-timestamp', '')
        signature = event['headers'].get('x-signature-ed25519', '')
        verify_key = nacl.signing.VerifyKey(bytes.fromhex(DISCORD_PUBLIC_KEY))
        verify_key.verify(f"{timestamp}{body}".encode(), bytes.fromhex(signature))
        return True
    except (BadSignatureError, ValueError, KeyError, Exception):
        return False

# ─────────────────────────── Helpers ───────────────────────────
def resolve_target(options):
    """
    Read the 'name' option from the slash command.
    Returns (list_of_instance_ids, error_message).
    Supports the special value 'all'.
    """
    name = None
    for opt in (options or []):
        if opt.get('name') == 'name':
            name = str(opt.get('value', '')).lower().strip()

    available = ", ".join(sorted(INSTANCE_MAP)) + ", all"

    if not name:
        return None, f"❌ Instance name missing! Available: `{available}`"
    if name == 'all':
        return list(INSTANCE_MAP.values()), None
    if name not in INSTANCE_MAP:
        return None, f"❌ Unknown instance `{name}`. Available: `{available}`"
    return [INSTANCE_MAP[name]], None


def clamp(msg):
    """Never exceed Discord's 2000-char message limit."""
    if len(msg) > DISCORD_MSG_LIMIT:
        return msg[:DISCORD_MSG_LIMIT - 20] + "\n…(truncated)"
    return msg


def friendly(instance_id):
    return ID_TO_NAME.get(instance_id, instance_id)


def uptime_of(instance):
    launch = instance.get('LaunchTime')
    if not launch or instance['State']['Name'] != 'running':
        return None, 'N/A'
    diff  = datetime.now(timezone.utc) - launch
    hours = diff.total_seconds() / 3600
    return hours, f"{int(hours)}h {int((diff.total_seconds() % 3600) // 60)}m"


def cost_line(instance_type, hours_running):
    hourly = INSTANCE_COSTS.get(instance_type)
    if hourly is None:
        return "N/A"
    daily, monthly = hourly * 10, hourly * 10 * 22
    line = f"~${hourly}/hr | ~${daily:.2f}/day | ~${monthly:.2f}/mo"
    if hours_running:
        line += f" | 💸 session ${hourly * hours_running:.4f}"
    return line

# ─────────────────────────── Actions ───────────────────────────
def start_instances(ids):
    ec2.start_instances(InstanceIds=ids)
    names = ", ".join(f"`{friendly(i)}`" for i in ids)
    return f"✅ Starting {names} … wait ~30s, then `/status-ec2` to verify"


def stop_instances(ids):
    ec2.stop_instances(InstanceIds=ids)
    names = ", ".join(f"`{friendly(i)}`" for i in ids)
    return f"🛑 Stopping {names} … public IPs will be released"


def reboot_instances(ids):
    ec2.reboot_instances(InstanceIds=ids)
    names = ", ".join(f"`{friendly(i)}`" for i in ids)
    return f"🔄 Rebooting {names} … wait ~60s, then `/status-ec2`"


def get_status(ids):
    """
    Compact fleet view — one block per instance.
    Works for a single instance or the whole fleet ('all').
    """
    resp = ec2.describe_instances(InstanceIds=ids)
    blocks = []
    for res in resp['Reservations']:
        for inst in res['Instances']:
            iid    = inst['InstanceId']
            state  = inst['State']['Name']
            itype  = inst.get('InstanceType', '?')
            pub_ip = inst.get('PublicIpAddress', '—')
            hrs, up = uptime_of(inst)
            emoji  = STATE_EMOJI.get(state, '⚪')
            blocks.append(
                f"{emoji} **{friendly(iid)}** ({itype}) — {state.upper()}\n"
                f"   🌐 {pub_ip} | ⏱️ {up} | 💰 {cost_line(itype, hrs)}"
            )
    header = f"📊 **EC2 Fleet Status** — {len(blocks)} instance(s)\n━━━━━━━━━━━━━━━━━━━━\n"
    return header + "\n".join(blocks)


def get_details(ids):
    """
    Deep-dive on ONE instance: network, storage, AMI, security, tags.
    Refuses 'all' to keep output readable and under Discord's limit.
    """
    if len(ids) != 1:
        return "🔍 `/details-ec2` works on **one instance at a time** — pick a name, not `all`."

    resp = ec2.describe_instances(InstanceIds=ids)
    inst = resp['Reservations'][0]['Instances'][0]

    iid    = inst['InstanceId']
    state  = inst['State']['Name']
    itype  = inst.get('InstanceType', '?')
    az     = inst.get('Placement', {}).get('AvailabilityZone', 'N/A')
    ami    = inst.get('ImageId', 'N/A')
    key    = inst.get('KeyName', 'N/A')
    vpc    = inst.get('VpcId', 'N/A')
    subnet = inst.get('SubnetId', 'N/A')
    mon    = inst.get('Monitoring', {}).get('State', 'N/A')
    pub_ip = inst.get('PublicIpAddress', '—')
    prv_ip = inst.get('PrivateIpAddress', 'N/A')
    hrs, up = uptime_of(inst)

    sgs = ", ".join(g['GroupName'] for g in inst.get('SecurityGroups', [])) or 'N/A'

    vols = []
    for bdm in inst.get('BlockDeviceMappings', []):
        ebs = bdm.get('Ebs', {})
        vols.append(f"{bdm.get('DeviceName','?')} → {ebs.get('VolumeId','?')}")
    vols_str = "; ".join(vols) or 'N/A'

    tags = ", ".join(f"{t['Key']}={t['Value']}" for t in inst.get('Tags', [])) or 'None'

    emoji = STATE_EMOJI.get(state, '⚪')
    return f"""
{emoji} **Instance Details — {friendly(iid)}**
━━━━━━━━━━━━━━━━━━━━
🆔 **ID:**          {iid}
💻 **Type:**        {itype}
📍 **State:**       {state.upper()}
🌍 **AZ:**          {az}
━━━━━━━━━━━━━━━━━━━━
🌐 **Public IP:**   {pub_ip}
🔒 **Private IP:**  {prv_ip}
🕸️ **VPC/Subnet:**  {vpc} / {subnet}
🛡️ **Sec Groups:**  {sgs}
━━━━━━━━━━━━━━━━━━━━
💿 **AMI:**         {ami}
🔑 **Key Pair:**    {key}
💾 **Volumes:**     {vols_str}
📈 **Monitoring:**  {mon}
🏷️ **Tags:**        {tags}
━━━━━━━━━━━━━━━━━━━━
⏱️ **Uptime:**      {up}
💰 **Cost:**        {cost_line(itype, hrs)}
━━━━━━━━━━━━━━━━━━━━
"""

# ─────────────────────────── Discord plumbing ───────────────────────────
def respond(message):
    return {
        'statusCode': 200,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps({'type': 4, 'data': {'content': clamp(message)}})
    }


COMMANDS = {
    'start-ec2':   start_instances,
    'stop-ec2':    stop_instances,
    'reboot-ec2':  reboot_instances,
    'status-ec2':  get_status,
    'details-ec2': get_details,
}


def lambda_handler(event, context):
    if not verify_signature(event):
        return {'statusCode': 401, 'body': json.dumps('Invalid signature')}

    body = json.loads(event['body'])

    # Discord verification PING
    if body.get('type') == 1:
        return {'statusCode': 200, 'body': json.dumps({'type': 1})}

    # Slash command
    if body.get('type') == 2:
        user_id = body['member']['user']['id']
        command = body['data']['name']
        options = body['data'].get('options', [])

        if user_id not in ALLOWED_USER_IDS:
            return respond("❌ You are **not authorized**!")

        handler = COMMANDS.get(command)
        if handler is None:
            return respond(f"❌ Unknown command `{command}`")

        ids, err = resolve_target(options)
        if err:
            return respond(err)

        try:
            return respond(handler(ids))
        except Exception as e:
            return respond(f"⚠️ AWS error: `{type(e).__name__}: {e}`")

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

**Configuration → Environment Variables → Edit → Add:**

| Key                  | Value                                                                    |
| -------------------- | ------------------------------------------------------------------------ |
| `EC2_REGION`         | e.g. `ap-south-1`                                                        |
| `INSTANCE_MAP`       | `{"gameserver":"i-0aaa...","devbox":"i-0bbb...","gitlab":"i-0ccc..."}`   |
| `DISCORD_PUBLIC_KEY` | From Discord Developer Portal → General Information                      |
| `ALLOWED_USER_IDS`   | Discord User ID(s), comma separated e.g. `1111,2222`                     |

> ⚠️ `INSTANCE_MAP` must be **valid JSON** — double quotes only, no
> trailing commas. Validate at jsonlint.com before pasting.

### IAM Permissions (scoped — production grade)

1. Lambda → **Configuration → Permissions → click Role name**
2. **Add Permissions → Create Inline Policy → JSON:**

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "ControlOnlyOurInstances",
      "Effect": "Allow",
      "Action": [
        "ec2:StartInstances",
        "ec2:StopInstances",
        "ec2:RebootInstances"
      ],
      "Resource": [
        "arn:aws:ec2:ap-south-1:YOUR_ACCOUNT_ID:instance/i-0aaa1111111111111",
        "arn:aws:ec2:ap-south-1:YOUR_ACCOUNT_ID:instance/i-0bbb2222222222222",
        "arn:aws:ec2:ap-south-1:YOUR_ACCOUNT_ID:instance/i-0ccc3333333333333"
      ]
    },
    {
      "Sid": "DescribeNeedsWildcard",
      "Effect": "Allow",
      "Action": ["ec2:DescribeInstances"],
      "Resource": "*"
    }
  ]
}
```

3. Name it `EC2FleetDiscordControl` → **Create**

> ℹ️ `ec2:DescribeInstances` does **not** support resource-level ARNs —
> it must stay `"*"`. Start/Stop/Reboot are locked to your exact
> instances, so even if the bot is compromised it cannot touch anything
> else in the account.
>
> Simpler (but broader) alternative: use `"Resource": "*"` for everything.

### Lambda Settings

**Configuration → General Configuration → Edit:**

- **Timeout:** 10 seconds
- **Memory:** 128 MB (default is fine)

> Discord requires a response within **3 seconds**. Start/Stop/Reboot API
> calls return in well under a second (they're async on AWS's side), so
> this pattern is safe.

---

## Part 5 — API Gateway

1. **API Gateway → Create API → HTTP API**
2. **Add Integration → Lambda** → select `discord-ec2-fleet-controller`
3. **API name:** `discord-ec2-fleet-api`
4. **Route:** Method `POST` → Path `/discord`
5. **Next → Next → Create**
6. **Stages → $default** → copy **Invoke URL**

Your full endpoint:

```
https://xxxxxxxxxx.execute-api.ap-south-1.amazonaws.com/discord
```

---

## Part 6 — Register Slash Commands

Create `register.py` on your PC. **Keep `INSTANCES` in sync with the keys
of your `INSTANCE_MAP` env var** (plus `all`):

```python
import requests

BOT_TOKEN      = "YOUR_BOT_TOKEN"
APPLICATION_ID = "YOUR_APPLICATION_ID"
GUILD_ID       = "YOUR_SERVER_ID"

# Must match INSTANCE_MAP keys + the special 'all' target
INSTANCES = ["gameserver", "devbox", "gitlab", "buildagent", "all"]

url = f"https://discord.com/api/v10/applications/{APPLICATION_ID}/guilds/{GUILD_ID}/commands"
headers = {"Authorization": f"Bot {BOT_TOKEN}"}

def name_option(allow_all=True):
    choices = INSTANCES if allow_all else [i for i in INSTANCES if i != "all"]
    return {
        "name": "name",
        "description": "Which instance",
        "type": 3,  # STRING
        "required": True,
        "choices": [{"name": c, "value": c} for c in choices],
    }

commands = [
    {"name": "start-ec2",   "description": "Start EC2 instance(s)",        "options": [name_option()]},
    {"name": "stop-ec2",    "description": "Stop EC2 instance(s)",         "options": [name_option()]},
    {"name": "reboot-ec2",  "description": "Reboot EC2 instance(s)",       "options": [name_option()]},
    {"name": "status-ec2",  "description": "Fleet status: state/IP/cost",  "options": [name_option()]},
    {"name": "details-ec2", "description": "Deep details of ONE instance", "options": [name_option(allow_all=False)]},
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

Expected output (200 = updated, 201 = created — both fine):

```
start-ec2:   201
stop-ec2:    201
reboot-ec2:  201
status-ec2:  201
details-ec2: 201
```

> ℹ️ Discord allows max **25 choices** per option. If you ever exceed 25
> instances, remove the `choices` list and rely on free-text input —
> the Lambda already validates names server-side.

---

## Part 7 — Connect Discord to API Gateway

1. **Discord Developer Portal → your app → General Information**
2. Paste into **Interactions Endpoint URL:**

```
https://xxxxxxxxxx.execute-api.ap-south-1.amazonaws.com/discord
```

3. **Save Changes** → should show ✅ (Discord sends a PING; the Lambda's
   `type == 1` branch answers it)

---

## Commands Reference

| Command                        | What it does                                        |
| ------------------------------ | --------------------------------------------------- |
| `/start-ec2 name:devbox`       | Start one instance                                  |
| `/start-ec2 name:all`          | Start the entire fleet                              |
| `/stop-ec2 name:gameserver`    | Stop one instance                                   |
| `/stop-ec2 name:all`           | Stop everything (end of day 🌙)                     |
| `/reboot-ec2 name:gitlab`      | Reboot one instance                                 |
| `/status-ec2 name:all`         | Compact fleet overview: state, IP, uptime, cost     |
| `/status-ec2 name:devbox`      | Same compact view, single instance                  |
| `/details-ec2 name:devbox`     | Full deep-dive: VPC, SGs, AMI, volumes, tags, key   |

### Example `/status-ec2 name:all` Output

```
📊 EC2 Fleet Status — 4 instance(s)
━━━━━━━━━━━━━━━━━━━━
🟢 gameserver (t3.medium) — RUNNING
   🌐 13.235.xx.xx | ⏱️ 3h 24m | 💰 ~$0.0416/hr | ~$0.42/day | ~$9.15/mo | 💸 session $0.1414
🔴 devbox (t3.small) — STOPPED
   🌐 — | ⏱️ N/A | 💰 ~$0.0208/hr | ~$0.21/day | ~$4.58/mo
🟢 gitlab (t3.large) — RUNNING
   🌐 3.110.xx.xx | ⏱️ 27h 02m | 💰 ~$0.0832/hr | ~$0.83/day | ~$18.30/mo | 💸 session $2.2481
🔴 buildagent (c5.large) — STOPPED
   🌐 — | ⏱️ N/A | 💰 ~$0.085/hr | ~$0.85/day | ~$18.70/mo
```

### Example `/details-ec2 name:devbox` Output

```
🔴 Instance Details — devbox
━━━━━━━━━━━━━━━━━━━━
🆔 ID:          i-0bbb2222222222222
💻 Type:        t3.small
📍 State:       STOPPED
🌍 AZ:          ap-south-1a
━━━━━━━━━━━━━━━━━━━━
🌐 Public IP:   —
🔒 Private IP:  172.31.xx.xx
🕸️ VPC/Subnet:  vpc-0abc123 / subnet-0def456
🛡️ Sec Groups:  dev-ssh, dev-web
━━━━━━━━━━━━━━━━━━━━
💿 AMI:         ami-0abcd1234efgh5678
🔑 Key Pair:    dev-keypair
💾 Volumes:     /dev/xvda → vol-0aaa111; /dev/sdb → vol-0bbb222
📈 Monitoring:  disabled
🏷️ Tags:        Name=DevBox, Team=Platform, Env=dev
━━━━━━━━━━━━━━━━━━━━
⏱️ Uptime:      N/A
💰 Cost:        ~$0.0208/hr | ~$0.21/day | ~$4.58/mo
━━━━━━━━━━━━━━━━━━━━
```

---

## Adding / Removing Instances Later

Three touch points — no redeploy of code needed:

1. **Lambda env var:** edit `INSTANCE_MAP` JSON (add/remove the entry)
2. **register.py:** update the `INSTANCES` list → run `python register.py`
   again (Discord upserts commands, old ones are overwritten)
3. **IAM policy:** add/remove the instance ARN in the inline policy
4. *(Optional)* **EventBridge:** add/remove the ID from the relevant
   schedule's `InstanceIds` JSON

---

## Cost Estimate

| Service     | Free Tier                | Monthly Cost |
| ----------- | ------------------------ | ------------ |
| Lambda      | 1M requests free         | $0.00        |
| API Gateway | 1M requests free         | $0.00        |
| EventBridge | 14 free schedules + 1M invocations | $0.00 |
| EC2         | Depends on instance type | Varies       |

Even with 4 instances and hundreds of Discord commands per month, the
control plane stays inside the free tier.

---

## Troubleshooting

### ❌ `invalid ELF header` on Lambda

**Cause:** pynacl built on macOS/Windows, not Linux.
**Fix:** Rebuild in AWS CloudShell with `--platform manylinux2014_x86_64`.

### ❌ `No module named '_cffi_backend'`

**Cause:** cffi C extension missing.
**Fix:** Install cffi with the same `--platform` flags in CloudShell.

### ❌ `JSONDecodeError` on Lambda cold start

**Cause:** `INSTANCE_MAP` env var is not valid JSON (single quotes,
trailing comma, or smart quotes from copy-paste).
**Fix:** Validate the JSON, use double quotes only, re-save env vars.

### ❌ Discord: `interactions endpoint url could not be verified`

**Cause 1:** URL missing `/discord` at the end. **Fix:** append it.
**Cause 2:** Lambda import/env-var error → 500. **Fix:** check
CloudWatch logs, fix the error, retry Save.

### ❌ `You are not authorized!` in Discord

**Cause:** wrong Discord User ID in `ALLOWED_USER_IDS`.
**Fix:** temporarily change the message to reveal the caller's ID:

```python
return respond(f"❌ Not authorized! Your ID: `{user_id}`")
```

Copy it → paste into `ALLOWED_USER_IDS` → revert code.

### ❌ `UnauthorizedOperation` when starting/stopping

**Cause:** the instance's ARN is missing from the scoped IAM policy
(common after adding a new instance to `INSTANCE_MAP` but forgetting
IAM). **Fix:** add the new ARN to the inline policy.

### ❌ `/status-ec2 name:all` output cut off

**Cause:** more instances than fit in Discord's 2000-char limit — the
Lambda clamps and appends `…(truncated)`.
**Fix:** query instances individually, or shorten the per-instance block.

### ❌ `KeyError: 'body'` when testing in Lambda console

**Cause:** wrong test event shape. **Fix:** use:

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

Expected result: `{"statusCode": 401}` ✅ (signature correctly rejected)

---

## Values Reference

| Value             | Where to Find                                           |
| ----------------- | ------------------------------------------------------- |
| `Application ID`  | Discord → App → General Information                     |
| `Public Key`      | Discord → App → General Information                     |
| `Bot Token`       | Discord → App → Bot → Reset Token                       |
| `Discord User ID` | Discord → Right-click username → Copy User ID           |
| `Server ID`       | Discord → Right-click server → Copy Server ID           |
| `Instance IDs`    | AWS → EC2 → Instances (one per instance)                |
| `Account ID`      | AWS Console → top-right account menu                    |
| `EC2 Region`      | AWS → top right corner                                  |
| `API Gateway URL` | API Gateway → your API → Stages → $default → Invoke URL |

---

## Tech Stack

- **AWS EventBridge Scheduler** — per-group cron start/stop
- **AWS Lambda** — serverless controller (Python 3.12)
- **AWS API Gateway** — HTTPS endpoint for Discord interactions
- **Discord Interactions API** — slash commands with choice dropdowns
- **pynacl** — Ed25519 signature verification

---

*Built with ❤️ — one bot, entire fleet, $0.00 control-plane cost.*
