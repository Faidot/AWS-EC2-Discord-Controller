# Discord EC2 Controller — Final Setup README

Control AWS EC2 instances (start / stop / reboot / detailed status) from Discord slash commands. Supports a single instance, multiple instances in one account, and instances spread across multiple regions — all through **one bot and one Lambda function**.

---

## Architecture

```
Discord Slash Command (/ec2 action:status instance:gitlab)
        │  HTTPS POST (signed with Ed25519)
        ▼
API Gateway (HTTP API)  →  POST /discord
        │
        ▼
Lambda (Python 3.12)  →  reads INSTANCE_MAP env var  →  boto3 EC2 client per region
        │
        ▼
EC2 StartInstances / StopInstances / RebootInstances / DescribeInstances

EventBridge Scheduler ── cron ── StartInstances / StopInstances (auto on/off)
```

One `INSTANCE_MAP` environment variable maps nicknames → `{instance_id, region}`. Add more instances or regions by adding entries — no new Lambda, no new bot, no new API.

---

## Prerequisites

- AWS account with permissions for IAM, Lambda, API Gateway, EventBridge
- One or more EC2 instances already launched — note each **Instance ID** and its **Region** (not Availability Zone)
- A Discord account and a server you manage
- AWS CloudShell (built into the console — do all packaging here, not locally, to avoid OS/version mismatches)

---

## Step 1 — IAM Role for Lambda

1. **IAM → Roles → Create role → AWS service → Lambda**
2. Attach an inline policy (JSON tab):
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
        "ec2:DescribeInstances",
        "ec2:DescribeInstanceStatus"
      ],
      "Resource": "*"
    }
  ]
}
```
3. Also attach the managed policy **`AWSLambdaBasicExecutionRole`** (for CloudWatch logging)
4. Name it `discord-ec2-controller-role` → Create

---

## Step 2 — Discord Application & Bot

1. **discord.com/developers/applications → New Application**
2. **General Information** → copy and save: `Application ID`, `Public Key`
3. **Bot** tab → Reset Token → copy and save the `Bot Token`
4. **OAuth2 → URL Generator** → check `applications.commands` **and** `bot` scopes, permission `Send Messages` → open the generated URL → authorize it on your server
5. Enable **Developer Mode** (Discord Settings → Advanced)
6. Right-click your username → **Copy User ID** → save as your allowed user
7. Right-click your server icon → **Copy Server ID** → save as `GUILD_ID`

---

## Step 3 — Build the deployment package (in AWS CloudShell)

Always build in CloudShell, never on your local Windows/Mac machine — compiled dependencies (`pynacl`, `cffi`) must match Lambda's Linux + Python version exactly.

```bash
mkdir -p ~/ec2-bot && cd ~/ec2-bot

pip install pynacl cffi \
  --platform manylinux2014_x86_64 \
  --target . \
  --only-binary=:all: \
  --python-version 3.12 \
  --implementation cp
```

Create the function file with a heredoc (safer than pasting into any browser editor — avoids smart-quote/indentation corruption):

```bash
cat > lambda_function.py << 'PYEOF'
import json
import os
import boto3
from datetime import datetime, timezone
from nacl.signing import VerifyKey
from nacl.exceptions import BadSignatureError

PUBLIC_KEY = os.environ["DISCORD_PUBLIC_KEY"]
ALLOWED_USER_IDS = set(os.environ.get("ALLOWED_USER_IDS", "").split(","))
INSTANCE_MAP = json.loads(os.environ["INSTANCE_MAP"])

INSTANCE_PRICING = {
    "t3.micro": 0.0104,
    "t3.small": 0.0208,
    "t3.medium": 0.0416,
    "t3.large": 0.0832,
    "t3.xlarge": 0.1664,
    "t2.micro": 0.0116,
    "t2.medium": 0.0464,
    "m5.large": 0.096,
    "m5.xlarge": 0.192,
}


def verify_signature(event):
    signature = event["headers"].get("x-signature-ed25519")
    timestamp = event["headers"].get("x-signature-timestamp")
    body = event["body"]
    verify_key = VerifyKey(bytes.fromhex(PUBLIC_KEY))
    verify_key.verify(f"{timestamp}{body}".encode(), bytes.fromhex(signature))


def ec2_client_for(region):
    return boto3.client("ec2", region_name=region)


def lambda_handler(event, context):
    try:
        verify_signature(event)
    except (BadSignatureError, Exception):
        return {"statusCode": 401, "body": "invalid request signature"}

    body = json.loads(event["body"])

    if body["type"] == 1:
        return {"statusCode": 200, "body": json.dumps({"type": 1})}

    user_id = body["member"]["user"]["id"]
    if user_id not in ALLOWED_USER_IDS:
        return respond("You are not authorized to run this command.")

    options = {opt["name"]: opt["value"] for opt in body["data"].get("options", [])}
    action = options.get("action")
    nickname = options.get("instance")

    target = INSTANCE_MAP.get(nickname)
    if not target:
        return respond(f"Unknown instance `{nickname}`. Known: {', '.join(INSTANCE_MAP)}")

    client = ec2_client_for(target["region"])
    iid = target["id"]

    if action == "start":
        client.start_instances(InstanceIds=[iid])
        msg = f"Starting **{nickname}** ({target['region']})..."

    elif action == "stop":
        client.stop_instances(InstanceIds=[iid])
        msg = f"Stopping **{nickname}** ({target['region']})..."

    elif action == "reboot":
        client.reboot_instances(InstanceIds=[iid])
        msg = f"Rebooting **{nickname}** ({target['region']})..."

    elif action == "status":
        desc = client.describe_instances(InstanceIds=[iid])
        inst = desc["Reservations"][0]["Instances"][0]

        state = inst["State"]["Name"]
        itype = inst["InstanceType"]
        az = inst["Placement"]["AvailabilityZone"]
        region = target["region"]
        public_ip = inst.get("PublicIpAddress", "N/A")
        private_ip = inst.get("PrivateIpAddress", "N/A")
        name_tag = next(
            (t["Value"] for t in inst.get("Tags", []) if t["Key"] == "Name"),
            nickname
        )
        hourly_rate = INSTANCE_PRICING.get(itype, 0.0)

        if state == "running":
            launch_time = inst["LaunchTime"]
            now = datetime.now(timezone.utc)
            hours = (now - launch_time).total_seconds() / 3600
            h, m = int(hours), int((hours - int(hours)) * 60)
            uptime_str = f"{h}h {m}m"
            session_cost = hours * hourly_rate
            cost_line = f"Current Session: ${session_cost:.4f} (running {hours:.1f} hrs)"
        else:
            uptime_str = "-"
            cost_line = "Instance is stopped - no active session cost"

        daily = hourly_rate * 24
        monthly = hourly_rate * 24 * 30

        msg = (
            "```\n"
            f"EC2 Status Report\n"
            f"--------------------\n"
            f"Name:         {name_tag}\n"
            f"Instance ID:  {iid}\n"
            f"Type:         {itype}\n"
            f"State:        {state.upper()}\n"
            f"Region/AZ:    {region} ({az})\n"
            f"--------------------\n"
            f"Public IP:    {public_ip}\n"
            f"Private IP:   {private_ip}\n"
            f"Uptime:       {uptime_str}\n"
            f"--------------------\n"
            f"Cost:         ~${hourly_rate:.4f}/hr | ~${daily:.2f}/day | ~${monthly:.2f}/month\n"
            f"{cost_line}\n"
            "```"
        )

    else:
        msg = "Unknown action."

    return respond(msg)


def respond(text):
    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps({"type": 4, "data": {"content": text}})
    }
PYEOF
```

**Validate syntax before zipping — do this every time you edit the file:**
```bash
python3 -c "import ast; ast.parse(open('lambda_function.py').read()); print('SYNTAX OK')"
```

**Confirm the file sits at zip root, and dependencies match Python 3.12:**
```bash
ls | grep lambda_function.py
ls | grep cffi_backend        # must show cpython-312, not 313
```

**Zip and confirm structure:**
```bash
rm -f function.zip
zip -r function.zip .
unzip -l function.zip | grep -E "lambda_function.py|cffi_backend"
```
`lambda_function.py` must appear with **no folder prefix** (e.g. not `ec2-bot/lambda_function.py`).

---

## Step 4 — Create the Lambda function

1. **Lambda → Create function**: name `discord-ec2-controller`, Runtime **Python 3.12**, Architecture **x86_64**
2. Under execution role, select **Use an existing role** → `discord-ec2-controller-role`
3. **Code tab → Upload from → .zip file** → upload `function.zip`
4. **Configuration → Environment variables**, add all three:

| Key | Example value |
|---|---|
| `DISCORD_PUBLIC_KEY` | your Discord app's Public Key |
| `ALLOWED_USER_IDS` | your Discord User ID (comma-separate for multiple people) |
| `INSTANCE_MAP` | `{"gitlab":{"id":"i-01a2aefc2ba19130b","region":"ap-south-1"}}` |

   - `INSTANCE_MAP` must be **valid single-line JSON**: double quotes only, no trailing commas
   - The `region` value must be a **region** (`ap-south-1`), never an **Availability Zone** (`ap-south-1b` is wrong)
   - Add more instances as more keys in the same object, e.g. `{"web":{"id":"i-...","region":"us-east-1"},"db":{"id":"i-...","region":"eu-west-1"}}`

5. **Configuration → General configuration**: Timeout `10 sec`, Memory `128 MB`

**Confirm what Lambda actually has stored (don't trust the console alone):**
```bash
aws lambda get-function-configuration --function-name discord-ec2-controller --query 'Environment'
```

**If you ever need to fix env vars via CLI directly:**
```bash
aws lambda update-function-configuration \
  --function-name discord-ec2-controller \
  --environment '{"Variables":{"DISCORD_PUBLIC_KEY":"...","ALLOWED_USER_IDS":"...","INSTANCE_MAP":"{\"gitlab\":{\"id\":\"i-...\",\"region\":\"ap-south-1\"}}"}}'
```

---

## Step 5 — API Gateway

1. **API Gateway → Create API → HTTP API → Build**
2. Add integration → Lambda → `discord-ec2-controller`
3. Route: `POST /discord`
4. Create → go to **Stages** → confirm `$default` stage exists with **Auto-deploy: true**
5. Copy the Invoke URL, final endpoint is:
   `https://<api-id>.execute-api.<region>.amazonaws.com/discord`

**Confirm Lambda has permission to be invoked by API Gateway:**
```bash
aws lambda get-policy --function-name discord-ec2-controller
```
Should show a statement with `"Service":"apigateway.amazonaws.com"` scoped to your API's `/discord` route. If this errors `ResourceNotFoundException`, the permission is missing — re-add the Lambda integration in API Gateway.

**Sanity-check the whole chain before touching Discord:**
```bash
curl -i -X POST https://<api-id>.execute-api.<region>.amazonaws.com/discord \
  -H "Content-Type: application/json" \
  -d '{"type":1}'
```
Expected: `HTTP/2 401` with body `invalid request signature`. That confirms API Gateway → Lambda routing works (the 401 is correct — curl can't fake Discord's real signature).

---

## Step 6 — Register the slash command

On your local PC, `register.py`:
```python
import requests

APP_ID = "YOUR_APPLICATION_ID"
BOT_TOKEN = "YOUR_BOT_TOKEN"
GUILD_ID = "YOUR_SERVER_ID"

url = f"https://discord.com/api/v10/applications/{APP_ID}/guilds/{GUILD_ID}/commands"
headers = {"Authorization": f"Bot {BOT_TOKEN}"}

command = {
    "name": "ec2",
    "description": "Control an EC2 instance",
    "options": [
        {
            "type": 3, "name": "action", "description": "What to do", "required": True,
            "choices": [
                {"name": "start", "value": "start"},
                {"name": "stop", "value": "stop"},
                {"name": "reboot", "value": "reboot"},
                {"name": "status", "value": "status"},
            ],
        },
        {
            "type": 3, "name": "instance", "description": "Which instance", "required": True,
            "choices": [
                {"name": "gitlab", "value": "gitlab"},
                # add one entry per key in INSTANCE_MAP
            ],
        },
    ],
}

r = requests.post(url, headers=headers, json=command)
print(r.status_code, r.text)
```
```bash
pip install requests
python register.py
```
Expect `201`.

---

## Step 7 — Connect Discord to your endpoint

1. Discord Developer Portal → your app → **General Information**
2. Paste `https://<api-id>.execute-api.<region>.amazonaws.com/discord` into **Interactions Endpoint URL**
3. Save — should turn green

---

## Step 8 — Scheduling automatic start/stop (optional)

Per instance, per region, in **EventBridge → Schedules**:
1. Create schedule, cron in **UTC**, e.g. `0 9 ? * MON-FRI *` for 9 AM start
2. Target: **AWS API → EC2 → StartInstances** (or `StopInstances`), Region = the instance's own region
3. Input: `{"InstanceIds": ["i-xxxxxxxxxxxx"]}`

Since EventBridge Scheduler is regional, create the schedule **in the same region as the target instance**.

---

## Every bug hit during setup — and the fix

| Symptom | Root cause | Fix |
|---|---|---|
| `403 Missing Access` when running `register.py` | Bot invited without `applications.commands` scope, or wrong Guild/App ID | Re-run OAuth2 URL Generator with both `applications.commands` + `bot` scopes, re-authorize on the correct server |
| `No module named 'lambda_function'` | Zip was built with the folder nested inside (`ec2-bot/lambda_function.py`) instead of at zip root | `cd` into the folder containing the file, then `zip -r function.zip .` (the `.` matters) |
| `No module named '_cffi_backend'` | Package installed for the wrong Python version (e.g. cpython-313 wheel on a Python 3.12 Lambda) | Reinstall with `--python-version 3.12 --implementation cp --platform manylinux2014_x86_64 --only-binary=:all:` |
| `KeyError: 'INSTANCE_MAP'` | Environment variable named `INSTANCE_ID` instead of `INSTANCE_MAP`, or never saved | Verify with `aws lambda get-function-configuration --query 'Environment'`; fix name/value via console or `update-function-configuration` |
| Region value like `ap-south-1b` rejected by boto3 | Availability Zone used instead of Region in `INSTANCE_MAP` | Use `ap-south-1`, not `ap-south-1b` |
| `{"statusCode": 401, "body": "invalid request signature"}` on manual Lambda test | Expected behavior — Lambda's Test button can't produce a real Discord Ed25519 signature | Not a bug; verify with a real Discord PING or `curl` against the API Gateway URL instead |
| Discord: "The application did not respond" | API Gateway → Lambda permission or route misconfigured | Check `aws lambda get-policy`, confirm route `POST /discord`, confirm `$default` stage exists with auto-deploy |
| "The specified interactions endpoint url could not be verified" | `lambda_function.py` had a `SyntaxError` (broke Lambda's init phase entirely) | Validate with `python3 -c "import ast; ast.parse(open('lambda_function.py').read())"` before zipping/deploying, every time |
| Emoji turned into `�` in the source file | Pasting through browser console editor / Windows terminal mangled UTF-8 encoding | Build the file in CloudShell with a `cat << 'PYEOF'` heredoc instead of pasting into the Lambda console editor |
| Dict/code block landed mid-function, breaking `if/elif` chain | Manual copy-paste inserted a block in the wrong place | Always replace the **entire file** in one shot from a known-good, syntax-validated source rather than patching fragments |

---

## Quick end-to-end verification checklist

```bash
# 1. Syntax valid
python3 -c "import ast; ast.parse(open('lambda_function.py').read()); print('SYNTAX OK')"

# 2. Zip structured correctly
unzip -l function.zip | grep -E "lambda_function.py|cffi_backend"

# 3. Env vars present and correct
aws lambda get-function-configuration --function-name discord-ec2-controller --query 'Environment'

# 4. Lambda has no crash on load
aws lambda invoke --function-name discord-ec2-controller --payload '{}' /tmp/out.json --cli-binary-format raw-in-base64-out && cat /tmp/out.json

# 5. API Gateway can invoke Lambda
aws lambda get-policy --function-name discord-ec2-controller

# 6. Full chain reachable
curl -i -X POST https://<api-id>.execute-api.<region>.amazonaws.com/discord -H "Content-Type: application/json" -d '{"type":1}'

# 7. Live logs while testing from Discord
aws logs tail /aws/lambda/discord-ec2-controller --since 5m
```

If all seven pass, `/ec2 action:status instance:<name>` in Discord will return the full status report.

---

## Scaling notes

- **Single instance:** one `INSTANCE_MAP` entry, one choice in `register.py`.
- **Multiple instances, same account:** add more entries to `INSTANCE_MAP` and matching choices in `register.py`. Same Lambda, same API Gateway, same bot.
- **Multi-region:** each `INSTANCE_MAP` entry carries its own `region`; the Lambda creates a fresh `boto3.client("ec2", region_name=...)` per request, so one Lambda (deployed in any single region) can control instances anywhere. Only EventBridge Scheduler needs a schedule created *in* each target region, since it is a regional service.
- **Cost:** effectively $0/month for the bot infrastructure itself (Lambda + API Gateway free tier) — you only pay for the EC2 instances you actually run.
