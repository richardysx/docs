# AmneziaVPN on AWS — Complete Setup Guide

Deploy a self-hosted, censorship-resistant VPN on AWS using AmneziaWG.  
This guide uses an EC2 **Spot instance** (up to 70% cheaper) with an **Auto Scaling Group** for automatic recovery — if AWS interrupts your instance, a new one starts automatically with no changes needed on the client side.

**Estimated cost:** ~$3–6/month (t3.small Spot in Tokyo)  
**Time to complete:** ~30 minutes

---

## Table of Contents

1. [Prerequisites](#1-prerequisites)
2. [AWS Setup](#2-aws-setup)
   - 2.1 [Allocate an Elastic IP](#21-allocate-an-elastic-ip)
   - 2.2 [Create a Security Group](#22-create-a-security-group)
   - 2.3 [Create an SSH Key Pair](#23-create-an-ssh-key-pair)
   - 2.4 [Find the Ubuntu AMI](#24-find-the-ubuntu-ami)
3. [Prepare the Server Startup Script](#3-prepare-the-server-startup-script)
4. [Generate WireGuard Keys](#4-generate-wireguard-keys)
5. [Build the User Data Script](#5-build-the-user-data-script)
6. [Deploy with Auto Scaling Group](#6-deploy-with-auto-scaling-group)
   - 6.1 [Create a Launch Template](#61-create-a-launch-template)
   - 6.2 [Create the Auto Scaling Group](#62-create-the-auto-scaling-group)
7. [Verify the Server](#7-verify-the-server)
8. [Generate Client Configuration Strings](#8-generate-client-configuration-strings)
9. [Install the Amnezia Client](#9-install-the-amnezia-client)
10. [Adding More Clients](#10-adding-more-clients)
11. [Troubleshooting](#11-troubleshooting)
12. [How Automatic Recovery Works](#12-how-automatic-recovery-works)

---

## 1. Prerequisites

- An **AWS account** with billing enabled
- **AWS CLI** installed and configured ([install guide](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2.html))
- **Python 3** installed locally (for generating client configs)
- Basic comfort with a terminal

### Install and configure AWS CLI

```bash
# macOS
brew install awscli

# Linux
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o awscliv2.zip
unzip awscliv2.zip && sudo ./aws/install

# Configure with your IAM credentials
aws configure
# Enter: Access Key ID, Secret Access Key, region (e.g. ap-northeast-1), output format (json)
```

> **Region choice:** This guide uses `ap-northeast-1` (Tokyo). Replace with your preferred region throughout.

---

## 2. AWS Setup

### 2.1 Allocate an Elastic IP

An Elastic IP (EIP) is a **fixed public IP address** that stays the same even when your Spot instance is replaced. Your clients always connect to this IP.

```bash
aws ec2 allocate-address \
  --region ap-northeast-1 \
  --domain vpc \
  --query '{AllocationId:AllocationId, PublicIp:PublicIp}' \
  --output table
```

**Save the output** — you will need `AllocationId` and `PublicIp` throughout this guide.

```
Example output:
----------------------------------------------
| AllocationId           | PublicIp           |
|------------------------|--------------------|
| eipalloc-0abc12345xyz  | 12.34.56.78        |
----------------------------------------------
```

### 2.2 Create a Security Group

```bash
# Get your default VPC ID
VPC_ID=$(aws ec2 describe-vpcs \
  --region ap-northeast-1 \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' --output text)

echo "VPC: $VPC_ID"

# Create security group
SG_ID=$(aws ec2 create-security-group \
  --region ap-northeast-1 \
  --group-name amnezia-sg \
  --description "AmneziaVPN security group" \
  --vpc-id "$VPC_ID" \
  --query 'GroupId' --output text)

echo "Security Group: $SG_ID"

# Allow SSH (for setup and diagnostics)
aws ec2 authorize-security-group-ingress \
  --region ap-northeast-1 \
  --group-id "$SG_ID" \
  --protocol tcp --port 22 --cidr 0.0.0.0/0

# Allow AmneziaWG VPN traffic (UDP — choose any port you like)
aws ec2 authorize-security-group-ingress \
  --region ap-northeast-1 \
  --group-id "$SG_ID" \
  --protocol udp --port 41213 --cidr 0.0.0.0/0
```

> **Port note:** `41213` is an example. You can use any UDP port between 1024–65535. Just be consistent throughout the guide.

### 2.3 Create an SSH Key Pair

```bash
# Generate a key pair and save the private key
aws ec2 create-key-pair \
  --region ap-northeast-1 \
  --key-name amnezia-key \
  --query 'KeyMaterial' \
  --output text > ~/.ssh/amnezia-key.pem

chmod 600 ~/.ssh/amnezia-key.pem
echo "Key saved to ~/.ssh/amnezia-key.pem"
```

### 2.4 Find the Ubuntu AMI

```bash
# Get the latest Ubuntu 24.04 LTS AMI from Canonical
AMI_ID=$(aws ec2 describe-images \
  --region ap-northeast-1 \
  --owners 099720109477 \
  --filters \
    "Name=name,Values=ubuntu/images/hvm-ssd-gp3/ubuntu-noble-24.04-amd64-server-*" \
    "Name=state,Values=available" \
  --query 'sort_by(Images, &CreationDate)[-1].ImageId' \
  --output text)

echo "AMI: $AMI_ID"
```

---

## 3. Prepare the Server Startup Script

The server startup script runs automatically every time a new Spot instance launches. It installs Docker, configures AmneziaWG, and binds the Elastic IP — no manual SSH required.

Before writing the script, you need to **generate your VPN keys**.

---

## 4. Generate WireGuard Keys

AmneziaWG uses WireGuard cryptography. You need:

- **Server key pair** (one per server)
- **Client key pair** (one per device — phone, laptop, etc.)
- **Pre-shared key** (one per client, for extra security)

### Option A: Generate keys on any Linux/macOS machine with `wg` installed

```bash
# Install wireguard-tools
# macOS:  brew install wireguard-tools
# Ubuntu: sudo apt install wireguard-tools

# Server keys
SERVER_PRIV=$(wg genkey)
SERVER_PUB=$(echo "$SERVER_PRIV" | wg pubkey)

# Client 1 (e.g. phone)
C1_PRIV=$(wg genkey)
C1_PUB=$(echo "$C1_PRIV" | wg pubkey)
C1_PSK=$(wg genpsk)

# Client 2 (e.g. laptop)
C2_PRIV=$(wg genkey)
C2_PUB=$(echo "$C2_PRIV" | wg pubkey)
C2_PSK=$(wg genpsk)

# Print all keys
echo "SERVER_PRIV=$SERVER_PRIV"
echo "SERVER_PUB=$SERVER_PUB"
echo "C1_PRIV=$C1_PRIV"
echo "C1_PUB=$C1_PUB"
echo "C1_PSK=$C1_PSK"
echo "C2_PRIV=$C2_PRIV"
echo "C2_PUB=$C2_PUB"
echo "C2_PSK=$C2_PSK"
```

**Save these values** — you need them in all following steps. Never share private keys or PSKs.

### Option B: Generate keys using Python (no extra tools needed)

```python
#!/usr/bin/env python3
# save as gen_keys.py and run: python3 gen_keys.py

import subprocess, sys, os

def gen_keypair():
    priv = subprocess.check_output(['wg', 'genkey']).decode().strip()
    pub  = subprocess.check_output(['wg', 'pubkey'],
                                    input=priv.encode()).decode().strip()
    return priv, pub

def gen_psk():
    return subprocess.check_output(['wg', 'genpsk']).decode().strip()

server_priv, server_pub = gen_keypair()
c1_priv, c1_pub = gen_keypair()
c1_psk = gen_psk()
c2_priv, c2_pub = gen_keypair()
c2_psk = gen_psk()

print(f"SERVER_PRIV={server_priv}")
print(f"SERVER_PUB={server_pub}")
print(f"C1_PRIV={c1_priv}")
print(f"C1_PUB={c1_pub}")
print(f"C1_PSK={c1_psk}")
print(f"C2_PRIV={c2_priv}")
print(f"C2_PUB={c2_pub}")
print(f"C2_PSK={c2_psk}")
```

### Choose obfuscation parameters

AmneziaWG uses randomized parameters to disguise VPN traffic. Choose your own random values (or use the defaults below):

| Parameter | Purpose | Recommended Range | Example Value |
|-----------|---------|-------------------|---------------|
| `Jc` | Junk packet count | 3–10 | `6` |
| `Jmin` | Min junk size (bytes) | 40–100 | `64` |
| `Jmax` | Max junk size (bytes) | 200–400 | `267` |
| `S1` | Init packet junk | 15–50 | `27` |
| `S2` | Response packet junk | 15–50 | `46` |
| `S3` | Cookie reply junk | 10–30 | `18` |
| `S4` | Transport packet junk | 10–30 | `27` |
| `H1`–`H4` | Magic headers | any 32-bit integers | (generate below) |

```bash
# Generate random H1-H4 values
python3 -c "
import random
for i in range(1,5):
    a = random.randint(100000000, 999999999)
    b = random.randint(1000000000, 2100000000)
    print(f'H{i} = {a}-{b}')
"
```

---

## 5. Build the User Data Script

Create the file `userdata.sh` on your local machine. Replace all `<PLACEHOLDER>` values with your actual keys and settings.

```bash
cat > userdata.sh << 'OUTER_EOF'
#!/bin/bash
exec > /var/log/amnezia-restore.log 2>&1
set -x
echo "=== Starting AmneziaVPN setup: $(date) ==="

sleep 15  # Wait for network

export DEBIAN_FRONTEND=noninteractive
apt-get update -qq
apt-get install -y docker.io curl unzip
systemctl enable docker && systemctl start docker

# Install AWS CLI (needed to bind the Elastic IP)
cd /tmp
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip" -s
unzip -q awscliv2.zip && ./aws/install && rm -rf awscliv2.zip aws

# ── Write server AWG configuration ────────────────────────────────────────────
mkdir -p /opt/amnezia/awg
cat > /opt/amnezia/awg/awg0.conf << 'CONF'
[Interface]
PrivateKey = <SERVER_PRIV>
Address = 10.8.0.1/24
ListenPort = <VPN_PORT>
Jc   = <Jc>
Jmin = <Jmin>
Jmax = <Jmax>
S1   = <S1>
S2   = <S2>
S3   = <S3>
S4   = <S4>
H1   = <H1>
H2   = <H2>
H3   = <H3>
H4   = <H4>

[Peer]
#_Name = phone
PublicKey    = <C1_PUB>
PresharedKey = <C1_PSK>
AllowedIPs   = 10.8.0.2/32

[Peer]
#_Name = laptop
PublicKey    = <C2_PUB>
PresharedKey = <C2_PSK>
AllowedIPs   = 10.8.0.3/32
CONF

# ── Build the AmneziaWG Docker image ──────────────────────────────────────────
mkdir -p /opt/amnezia/amnezia-awg2
cat > /opt/amnezia/amnezia-awg2/Dockerfile << 'DOCKERFILE'
FROM amneziavpn/amneziawg-go:latest
RUN apk add --no-cache bash dumb-init && mkdir -p /opt/amnezia
ENTRYPOINT ["/bin/sh"]
DOCKERFILE

docker build --pull -t amnezia-awg2 /opt/amnezia/amnezia-awg2/

# ── Enable IP forwarding ───────────────────────────────────────────────────────
sysctl -w net.ipv4.ip_forward=1
echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf

# ── Create Docker network ──────────────────────────────────────────────────────
docker network create \
  --driver bridge \
  --subnet=172.29.172.0/24 \
  --opt com.docker.network.bridge.name=amn0 \
  amnezia-dns-net 2>/dev/null || true

# ── Start the AmneziaWG container ─────────────────────────────────────────────
docker run -d \
  --log-driver json-file \
  --restart always \
  --privileged \
  --cap-add=NET_ADMIN \
  --cap-add=SYS_MODULE \
  -p <VPN_PORT>:<VPN_PORT>/udp \
  -v /lib/modules:/lib/modules \
  -v /opt/amnezia/awg:/opt/amnezia/awg \
  --sysctl="net.ipv4.conf.all.src_valid_mark=1" \
  --name amnezia-awg2 \
  --entrypoint /bin/sh \
  amnezia-awg2 \
  -c '
    sysctl -w net.ipv4.ip_forward=1 2>/dev/null
    awg-quick down /opt/amnezia/awg/awg0.conf 2>/dev/null
    awg-quick up /opt/amnezia/awg/awg0.conf
    iptables -A FORWARD -i awg0 -j ACCEPT
    iptables -A FORWARD -o awg0 -j ACCEPT
    iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
    tail -f /dev/null
  '

sleep 15

# ── Bind Elastic IP to this instance ──────────────────────────────────────────
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)
REGION=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/region)

export AWS_ACCESS_KEY_ID=<YOUR_ACCESS_KEY_ID>
export AWS_SECRET_ACCESS_KEY=<YOUR_SECRET_ACCESS_KEY>

/usr/local/bin/aws ec2 associate-address \
  --instance-id "$INSTANCE_ID" \
  --allocation-id <EIP_ALLOCATION_ID> \
  --region "$REGION" \
  --allow-reassociation \
  && echo "Elastic IP bound successfully." \
  || echo "WARNING: Elastic IP binding failed."

echo "=== Setup complete: $(date) ==="
docker ps
OUTER_EOF

echo "userdata.sh created. Fill in all <PLACEHOLDER> values before proceeding."
```

**Fill in these placeholders in `userdata.sh`:**

| Placeholder | What to put |
|-------------|-------------|
| `<SERVER_PRIV>` | Server private key from Step 4 |
| `<VPN_PORT>` | Your chosen UDP port (e.g. `41213`) |
| `<Jc>` through `<H4>` | Obfuscation parameters from Step 4 |
| `<C1_PUB>`, `<C1_PSK>` | Client 1 public key and PSK |
| `<C2_PUB>`, `<C2_PSK>` | Client 2 public key and PSK |
| `<YOUR_ACCESS_KEY_ID>` | AWS IAM access key (needs `ec2:AssociateAddress`) |
| `<YOUR_SECRET_ACCESS_KEY>` | AWS IAM secret key |
| `<EIP_ALLOCATION_ID>` | Allocation ID from Step 2.1 |

> **Security note:** Embedding AWS credentials in User Data is acceptable for personal use. For production, use an IAM Instance Role instead and remove the `export AWS_*` lines.

---

## 6. Deploy with Auto Scaling Group

The ASG ensures exactly one Spot instance is always running. When AWS interrupts a Spot instance, the ASG automatically launches a replacement.

### 6.1 Create a Launch Template

```bash
# Base64-encode the user data script
USERDATA_B64=$(base64 -w 0 userdata.sh)   # Linux
# USERDATA_B64=$(base64 -i userdata.sh)   # macOS (remove -w 0)

# Set your values
REGION="ap-northeast-1"
AMI_ID="<AMI_ID_FROM_STEP_2_4>"
SG_ID="<SECURITY_GROUP_ID>"
KEY_NAME="amnezia-key"

aws ec2 create-launch-template \
  --region "$REGION" \
  --launch-template-name "amnezia-spot-lt" \
  --version-description "AmneziaVPN v1" \
  --launch-template-data "{
    \"ImageId\": \"$AMI_ID\",
    \"InstanceType\": \"t3.small\",
    \"KeyName\": \"$KEY_NAME\",
    \"SecurityGroupIds\": [\"$SG_ID\"],
    \"BlockDeviceMappings\": [{
      \"DeviceName\": \"/dev/sda1\",
      \"Ebs\": {
        \"VolumeSize\": 20,
        \"DeleteOnTermination\": true,
        \"VolumeType\": \"gp3\"
      }
    }],
    \"TagSpecifications\": [{
      \"ResourceType\": \"instance\",
      \"Tags\": [{\"Key\": \"Name\", \"Value\": \"amnezia-spot\"}]
    }],
    \"UserData\": \"$USERDATA_B64\",
    \"InstanceMarketOptions\": {
      \"MarketType\": \"spot\",
      \"SpotOptions\": {\"SpotInstanceType\": \"one-time\"}
    }
  }" \
  --query 'LaunchTemplate.{ID:LaunchTemplateId,Name:LaunchTemplateName}' \
  --output table
```

Save the **Launch Template ID** from the output.

### 6.2 Create the Auto Scaling Group

```bash
# Get subnet IDs for all availability zones in your region
SUBNETS=$(aws ec2 describe-subnets \
  --region "$REGION" \
  --filters "Name=defaultForAz,Values=true" \
  --query 'Subnets[*].SubnetId' \
  --output text | tr '\t' ',')

echo "Subnets: $SUBNETS"

LT_ID="<LAUNCH_TEMPLATE_ID>"

aws autoscaling create-auto-scaling-group \
  --region "$REGION" \
  --auto-scaling-group-name "amnezia-spot-asg" \
  --launch-template "LaunchTemplateId=$LT_ID,Version=1" \
  --min-size 1 \
  --max-size 1 \
  --desired-capacity 1 \
  --vpc-zone-identifier "$SUBNETS" \
  --capacity-rebalance \
  --health-check-type EC2 \
  --health-check-grace-period 300

echo "Auto Scaling Group created. Instance will launch in ~1 minute."
```

---

## 7. Verify the Server

Wait **5–8 minutes** for the instance to start and run the setup script, then verify:

```bash
EIP="<YOUR_ELASTIC_IP>"

# 1. Check that the instance picked up the EIP
aws ec2 describe-addresses \
  --region ap-northeast-1 \
  --filters "Name=public-ip,Values=$EIP" \
  --query 'Addresses[0].{IP:PublicIp,Instance:InstanceId}' \
  --output table

# 2. SSH into the instance to check the VPN service
ssh -i ~/.ssh/amnezia-key.pem ubuntu@"$EIP"
```

Once SSH'd in, run:

```bash
# Check the Docker container is running
sudo docker ps

# Check AmneziaWG interface status
sudo docker exec amnezia-awg2 awg show

# View the setup log
sudo tail -50 /var/log/amnezia-restore.log
```

Expected output of `awg show`:
```
interface: awg0
  public key: <your server public key>
  listening port: 41213
  jc: 6  jmin: 64  jmax: 267
  s1: 27  s2: 46  s3: 18  s4: 27
  h1: ...  h2: ...  h3: ...  h4: ...

peer: <client 1 public key>
  allowed ips: 10.8.0.2/32

peer: <client 2 public key>
  allowed ips: 10.8.0.3/32
```

---

## 8. Generate Client Configuration Strings

Amnezia clients connect using a `vpn://` URI that encodes all connection parameters. Run this Python script locally to generate the URIs.

```python
#!/usr/bin/env python3
# save as gen_vpn_uri.py

import json, base64

# ── Fill in your values ────────────────────────────────────────────────────────
SERVER_IP   = "<YOUR_ELASTIC_IP>"
SERVER_PORT = 41213                          # Must match your VPN_PORT
SERVER_PUB  = "<SERVER_PUB>"

# Obfuscation parameters (must match server awg0.conf exactly)
JC, JMIN, JMAX = "6", "64", "267"
S1, S2, S3, S4 = "27", "46", "18", "27"
H1 = "<H1>"
H2 = "<H2>"
H3 = "<H3>"
H4 = "<H4>"

# Client 1 — Phone
C1_PRIV = "<C1_PRIV>"
C1_PUB  = "<C1_PUB>"
C1_PSK  = "<C1_PSK>"
C1_IP   = "10.8.0.2/32"

# Client 2 — Laptop
C2_PRIV = "<C2_PRIV>"
C2_PUB  = "<C2_PUB>"
C2_PSK  = "<C2_PSK>"
C2_IP   = "10.8.0.3/32"
# ──────────────────────────────────────────────────────────────────────────────

def make_uri(client_priv, client_pub, client_ip, psk, label):
    wg_conf = (
        f"[Interface]\n"
        f"PrivateKey = {client_priv}\n"
        f"Address = {client_ip}\n"
        f"DNS = $PRIMARY_DNS\n"
        f"MTU = 1280\n"
        f"Jc = {JC}\nJmin = {JMIN}\nJmax = {JMAX}\n"
        f"S1 = {S1}\nS2 = {S2}\nS3 = {S3}\nS4 = {S4}\n"
        f"H1 = {H1}\nH2 = {H2}\nH3 = {H3}\nH4 = {H4}\n\n"
        f"[Peer]\n"
        f"PublicKey = {SERVER_PUB}\n"
        f"PresharedKey = {psk}\n"
        f"AllowedIPs = 0.0.0.0/0\n"
        f"Endpoint = {SERVER_IP}:{SERVER_PORT}\n"
        f"PersistentKeepalive = 25"
    )

    last_config = json.dumps({
        "config": wg_conf,
        "Jc": JC, "Jmin": JMIN, "Jmax": JMAX,
        "S1": S1, "S2": S2, "S3": S3, "S4": S4,
        "H1": H1, "H2": H2, "H3": H3, "H4": H4,
        "I1": "", "I2": "", "I3": "", "I4": "", "I5": "",
        "mtu": "1280",
        "client_priv_key": client_priv,
        "client_pub_key":  client_pub,
        "server_pub_key":  SERVER_PUB,
        "psk_key":         psk,
        "client_ip":       client_ip,
        "hostName":        SERVER_IP,
        "port":            SERVER_PORT,
        "allowed_ips":     ["0.0.0.0/0", "::/0"],
        "persistent_keep_alive": "25",
        "clientId":        client_pub,
    })

    config = {
        "containers": [{
            "container": "amnezia-awg",
            "awg": {
                "last_config":        last_config,
                "isThirdPartyConfig": True,
                "port":               SERVER_PORT,
                "transport_proto":    "udp",
                "protocol_version":   "2",
                "mtu":                "1280",
                "Jc": JC, "Jmin": JMIN, "Jmax": JMAX,
                "S1": S1, "S2": S2, "S3": S3, "S4": S4,
                "H1": H1, "H2": H2, "H3": H3, "H4": H4,
                "I1": "", "I2": "", "I3": "", "I4": "", "I5": "",
            }
        }],
        "defaultContainer": "amnezia-awg",
        "description": f"My VPN - {label}",
        "dns1": "1.1.1.1",
        "dns2": "8.8.8.8",
        "hostName": SERVER_IP,
    }

    uri = "vpn://" + base64.b64encode(json.dumps(config).encode()).decode()
    print(f"\n=== {label} ===")
    print(uri)
    return uri

make_uri(C1_PRIV, C1_PUB, C1_IP, C1_PSK, "Phone")
make_uri(C2_PRIV, C2_PUB, C2_IP, C2_PSK, "Laptop")
```

Run it:

```bash
python3 gen_vpn_uri.py
```

Each URI starts with `vpn://eyJ...`. **Copy each one** — you'll import it into the Amnezia app.

---

## 9. Install the Amnezia Client

### Download

| Platform | Download |
|----------|----------|
| macOS | [amnezia.org/en/downloads](https://amnezia.org/en/downloads) |
| Windows | [amnezia.org/en/downloads](https://amnezia.org/en/downloads) |
| iOS | [App Store — AmneziaVPN](https://apps.apple.com/app/amneziavpn/id1600529900) |
| Android | [Google Play — AmneziaVPN](https://play.google.com/store/apps/details?id=org.amnezia.vpn) |

### Import your configuration

1. Open the Amnezia app
2. Tap **"+"** or **"Add VPN"**
3. Choose **"Import config from text / QR code"**
4. Paste your `vpn://eyJ...` string and tap **Import**
5. The server should appear as **"AmneziaWG (version 2)"**
6. Tap **Connect**

> **Tip:** You can also generate a QR code from the URI and scan it on mobile. Use any online QR generator or the `qrencode` command:
> ```bash
> echo -n "vpn://eyJ..." | qrencode -t UTF8
> ```

---

## 10. Adding More Clients

To add a third device (e.g. a tablet):

### Step 1 — Generate a new key pair

```bash
C3_PRIV=$(wg genkey)
C3_PUB=$(echo "$C3_PRIV" | wg pubkey)
C3_PSK=$(wg genpsk)
echo "C3_PRIV=$C3_PRIV  C3_PUB=$C3_PUB  C3_PSK=$C3_PSK"
```

### Step 2 — Add the peer to the running server (no restart needed)

```bash
EIP="<YOUR_ELASTIC_IP>"

ssh -i ~/.ssh/amnezia-key.pem ubuntu@"$EIP" << EOF
# Add peer to the live AWG interface
echo "$C3_PSK" | sudo docker exec -i amnezia-awg2 \
  awg set awg0 peer "$C3_PUB" preshared-key /dev/stdin allowed-ips 10.8.0.4/32

# Persist to config file so it survives container restarts
sudo bash -c "cat >> /opt/amnezia/awg/awg0.conf" << PEER
[Peer]
#_Name = tablet
PublicKey    = $C3_PUB
PresharedKey = $C3_PSK
AllowedIPs   = 10.8.0.4/32
PEER

echo "Peer added."
sudo docker exec amnezia-awg2 awg show
EOF
```

> **Important:** Also add this new peer to the `userdata.sh` file and update the Launch Template, so future replacement instances include this peer.

### Step 3 — Generate the URI

Add C3 variables to `gen_vpn_uri.py` and run:

```python
make_uri(C3_PRIV, C3_PUB, "10.8.0.4/32", C3_PSK, "Tablet")
```

---

## 11. Troubleshooting

### Instance not starting

```bash
# Check ASG status
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names amnezia-spot-asg \
  --region ap-northeast-1 \
  --query 'AutoScalingGroups[0].{Desired:DesiredCapacity,Instances:Instances[*].InstanceId}'
```

### EIP not bound

```bash
# Check which instance holds the EIP
aws ec2 describe-addresses \
  --filters "Name=public-ip,Values=<YOUR_EIP>" \
  --region ap-northeast-1 \
  --query 'Addresses[0]'

# Manually bind if needed
aws ec2 associate-address \
  --instance-id <INSTANCE_ID> \
  --allocation-id <EIP_ALLOC_ID> \
  --region ap-northeast-1 \
  --allow-reassociation
```

### Container not running

```bash
ssh -i ~/.ssh/amnezia-key.pem ubuntu@<EIP>

# Check all containers
sudo docker ps -a

# View setup log
sudo tail -100 /var/log/amnezia-restore.log

# Manually start if stopped
sudo docker start amnezia-awg2
```

### Client connects but no internet access

```bash
# Check NAT rule inside container
sudo docker exec amnezia-awg2 iptables -t nat -L POSTROUTING -v

# If missing, re-add it
sudo docker exec amnezia-awg2 \
  iptables -t nat -A POSTROUTING -s 10.8.0.0/24 -o eth0 -j MASQUERADE
```

### Client shows "Legacy" instead of "AmneziaWG (version 2)"

This means the `vpn://` URI was not generated correctly. Re-run `gen_vpn_uri.py` — the script in this guide produces the correct JSON format. Delete the old server entry in the app and re-import.

### Test connectivity manually

```bash
# From your local machine — check if the UDP port is open
nc -zvu <YOUR_EIP> 41213

# Inside the server — watch for incoming packets
sudo tcpdump -i any udp port 41213 -n
```

---

## 12. How Automatic Recovery Works

```
 AWS interrupts the Spot instance
         │
         ▼
 Auto Scaling Group detects instance count = 0
         │
         ▼
 ASG launches a new Spot instance
 (picks cheapest available zone automatically)
         │
         ▼
 User Data script runs (~3–5 minutes):
   ✓ Installs Docker
   ✓ Builds AmneziaWG image
   ✓ Writes awg0.conf (same keys — embedded in script)
   ✓ Starts AWG container
   ✓ Binds the same Elastic IP
         │
         ▼
 Client reconnects automatically
 (PersistentKeepalive = 25s retries every 25 seconds)
         │
         ▼
 VPN restored — no client changes needed ✓
```

### Key points that make this work

| Mechanism | Why it matters |
|-----------|---------------|
| **Elastic IP** | Clients always connect to the same IP regardless of which instance is running |
| **Fixed server keys in User Data** | New instance has identical keys — existing client configs remain valid |
| **Docker `--restart always`** | If the container crashes, Docker restarts it automatically |
| **PersistentKeepalive = 25** | Client retries every 25 seconds — reconnects within ~5 minutes of recovery |
| **Multi-AZ ASG** | If one availability zone is out of Spot capacity, ASG tries another zone |

---

## Quick Reference

### Useful commands (run on the server after SSH)

```bash
# VPN status and connected clients
sudo docker exec amnezia-awg2 awg show

# View real-time traffic
sudo docker exec amnezia-awg2 awg show | grep transfer

# Restart VPN
sudo docker restart amnezia-awg2

# View setup log
sudo tail -f /var/log/amnezia-restore.log

# Check listening port
sudo ss -ulnp | grep <VPN_PORT>
```

### AWS commands (run locally)

```bash
# Check which instance the EIP is on
aws ec2 describe-addresses --region ap-northeast-1 \
  --query 'Addresses[*].{IP:PublicIp,Instance:InstanceId}' --output table

# Check ASG health
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names amnezia-spot-asg \
  --region ap-northeast-1 \
  --query 'AutoScalingGroups[0].Instances'

# Force ASG to replace the current instance
aws autoscaling terminate-instance-in-auto-scaling-group \
  --instance-id <INSTANCE_ID> \
  --should-decrement-desired-capacity \
  --region ap-northeast-1
```

---

## Security Recommendations

- **Never commit** `userdata.sh` to a public Git repository — it contains private keys and AWS credentials
- Store keys in a password manager (Bitwarden, 1Password, etc.)
- Use a dedicated IAM user with only `ec2:AssociateAddress` and `ec2:DescribeAddresses` permissions
- Rotate client keys periodically by generating new pairs, adding them to the server, and re-importing in the app
- Consider restricting the SSH (port 22) rule to your home IP address once setup is complete

---

*Built with [AmneziaVPN](https://amnezia.org) — open source, self-hosted, censorship-resistant VPN.*
