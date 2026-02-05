# OpenClaw GCP Complete Setup Guide

Complete guide for deploying OpenClaw AI Gateway on Google Cloud Platform with Cloudflare Tunnel, Vertex AI, and custom subagents.

---

## 📋 Requirements

- GCP Account with Billing enabled
- Domain (for Cloudflare Tunnel)
- Cloudflare Account (Zero Trust)

---

## Phase 1: Create GCP VM

### 1.1 Create Project & Enable APIs

```bash
# Create new project
gcloud projects create <PROJECT_ID> --name="OpenClaw Gateway"
gcloud config set project <PROJECT_ID>

# Link billing
gcloud billing projects link <PROJECT_ID> --billing-account=<BILLING_ID>

# Enable APIs
gcloud services enable compute.googleapis.com aiplatform.googleapis.com
```

### 1.2 Create VM

```bash
gcloud compute instances create openclaw-gateway \
    --zone=<ZONE> \
    --machine-type=e2-medium \
    --boot-disk-size=30GB \
    --image-family=debian-12 \
    --image-project=debian-cloud \
    --scopes=cloud-platform
```

**Recommended specs:**
| Spec | Value |
|------|-------|
| Machine | `e2-medium` (2 vCPU, 4GB RAM) |
| Disk | 30GB SSD |
| OS | Debian 12 |
| Scopes | `cloud-platform` ⚠️ Important! |

### 1.3 SSH into VM

```bash
gcloud compute ssh openclaw-gateway --zone=<ZONE>
```

---

## Phase 2: Setup Docker

```bash
# Install Docker
sudo apt-get update
sudo apt-get install -y git curl ca-certificates
curl -fsSL https://get.docker.com | sudo sh
sudo usermod -aG docker $USER

# Re-login to apply group
exit
gcloud compute ssh openclaw-gateway --zone=<ZONE>

# Verify
docker --version
```

---

## Phase 3: Deploy OpenClaw

### 3.1 Clone Repository

```bash
git clone https://github.com/openclaw/openclaw.git ~/openclaw
cd ~/openclaw
mkdir -p ~/.openclaw ~/.openclaw/workspace
```

### 3.2 Run Setup

```bash
chmod +x docker-setup.sh
./docker-setup.sh --non-interactive
```

### 3.3 Verify

```bash
docker compose ps
curl -s http://localhost:18789 | head -5
```

---

## Phase 4: CLIProxyAPI (LLM Backend)

```bash
git clone https://github.com/router-for-me/CLIProxyAPI.git ~/CLIProxyAPI
cd ~/CLIProxyAPI
mkdir -p auths logs

# Configure
nano config.yaml

# Start
sudo docker compose up -d
```

---

## Phase 5: Cloudflare Tunnel

### 5.1 Create Tunnel on Dashboard

1. Go to [Cloudflare Zero Trust](https://one.dash.cloudflare.com/)
2. **Networks** → **Tunnels** → **Create a tunnel**
3. Name your tunnel (e.g., `openclaw-gateway`)
4. Copy the **Tunnel Token**

### 5.2 Run Cloudflared Container

```bash
sudo docker run -d \
    --name cloudflared \
    --restart unless-stopped \
    --network openclaw_default \
    cloudflare/cloudflared:latest \
    tunnel --no-autoupdate run --token <TUNNEL_TOKEN>
```

### 5.3 Configure Public Hostname

In Cloudflare Dashboard → Tunnel → **Public Hostname**:

| Field | Value |
|-------|-------|
| Subdomain | `openclaw` |
| Domain | `<your-domain>` |
| Type | HTTP |
| URL | `openclaw-gateway:18789` or `localhost:18789` |

### 5.4 Configure trustedProxies

**IMPORTANT**: OpenClaw needs to trust Cloudflare proxy IPs.

**File:** `~/.openclaw/openclaw.json`

```json
{
  "gateway": {
    "mode": "local",
    "trustedProxies": [
      "172.18.0.1",
      "172.16.0.0/12",
      "10.0.0.0/8",
      "192.168.0.0/16",
      "127.0.0.1"
    ]
  }
}
```

---

## Phase 6: Cloudflare Access (Google SSO) - Optional

### 6.1 Create Google OAuth Credentials

1. Go to [Google Cloud Console > Credentials](https://console.cloud.google.com/apis/credentials)
2. **Create Credentials** → **OAuth client ID**
3. Application type: **Web application**
4. Authorized redirect URIs:
   ```
   https://<TEAM_NAME>.cloudflareaccess.com/cdn-cgi/access/callback
   ```
5. Copy **Client ID** and **Client Secret**

### 6.2 Add Google Login in Cloudflare

1. Cloudflare Zero Trust → **Settings** → **Authentication**
2. **Login methods** → **Add new** → **Google**
3. Paste Client ID and Client Secret
4. Save

### 6.3 Create Access Application

1. **Access** → **Applications** → **Add an application**
2. Select **Self-hosted**
3. Config:
   - Application name: `OpenClaw`
   - Session duration: `24 hours`
   - Application domain: `openclaw.<your-domain>`

4. Add Policy:
   - Policy name: `Allow Users`
   - Action: `Allow`
   - Include → Emails → `<your-email>`

5. Save

### 6.4 Verify

1. Open Incognito browser
2. Navigate to `https://openclaw.<your-domain>`
3. Cloudflare redirects to Google login
4. After auth → OpenClaw UI appears

---

## Phase 7: OpenClaw Configuration

### 7.1 Gateway Config

**File:** `~/.openclaw/openclaw.json`

```json
{
  "gateway": {
    "mode": "local",
    "trustedProxies": ["172.18.0.1", "10.0.0.0/8", "127.0.0.1"]
  },
  "models": {
    "providers": {
      "proxypal": {
        "baseUrl": "http://host.docker.internal:8317/v1",
        "apiKey": "<PROXY_KEY>",
        "api": "openai-completions",
        "models": [
          {"id": "claude-sonnet-4-5", "name": "Claude Sonnet 4.5"},
          {"id": "gemini-3-pro-preview", "name": "Gemini 3 Pro"}
        ]
      }
    }
  },
  "agents": {
    "defaults": {
      "model": {
        "primary": "proxypal/claude-sonnet-4-5",
        "fallbacks": ["proxypal/gemini-3-pro-preview"]
      }
    }
  }
}
```

### 7.2 Environment Variables

**File:** `~/openclaw/.env`

```bash
# Gateway
OPENCLAW_GATEWAY_TOKEN=<auto-generated>

# GCP
GOOGLE_CLOUD_PROJECT=<PROJECT_ID>
GOOGLE_CLOUD_LOCATION=us-central1

# LLM Backend
ANTHROPIC_API_KEY=<PROXY_KEY>
ANTHROPIC_BASE_URL=http://host.docker.internal:8317/v1
```

---

## Phase 8: Vertex AI & Python Packages

### 8.1 Grant IAM Roles

```bash
PROJECT_ID=$(gcloud config get-value project)
SA=$(gcloud compute instances describe openclaw-gateway --zone=<ZONE> \
  --format="get(serviceAccounts[0].email)")

gcloud projects add-iam-policy-binding $PROJECT_ID \
  --member="serviceAccount:$SA" \
  --role="roles/aiplatform.user"
```

### 8.2 Install Packages

```bash
sudo docker exec openclaw-openclaw-gateway-1 bash -c "
  apt-get update && apt-get install -y python3-pip chromium
  pip3 install google-cloud-aiplatform cognee playwright --break-system-packages
  /home/node/.local/bin/playwright install chromium
"
```

### 8.3 Test

```bash
# Vertex AI
sudo docker exec openclaw-openclaw-gateway-1 python3 -c "
import vertexai
vertexai.init(location='us-central1')
print('✅ Vertex AI ready!')
"

# Playwright
sudo docker exec openclaw-openclaw-gateway-1 python3 -c "
from playwright.sync_api import sync_playwright
with sync_playwright() as p:
    browser = p.chromium.launch(headless=True)
    print('✅ Playwright ready!')
    browser.close()
"
```

---

## Phase 9: Create Subagents

```bash
AGENT_NAME="lena-image-processor"

sudo docker exec openclaw-openclaw-gateway-1 mkdir -p /home/node/.openclaw/agents/$AGENT_NAME

sudo docker exec openclaw-openclaw-gateway-1 bash -c "cat > /home/node/.openclaw/agents/$AGENT_NAME/agent.yaml << 'EOF'
name: lena-image-processor
description: AI assistant for image processing
model: proxypal/gemini-3-pro-preview
systemPrompt: |
  You are Lena, an expert in image processing and generation.
tools:
  - generate_image
  - browser
maxTokens: 8192
EOF
"

# Restart
cd ~/openclaw && sudo docker compose restart openclaw-gateway
```

---

## Phase 10: Access & Verification

### Get Token

```bash
cat ~/openclaw/.env | grep TOKEN
```

### Access UI

- URL: `https://openclaw.<your-domain>/`
- Paste token in Settings
- Or: `https://openclaw.<your-domain>/?token=<TOKEN>`

---

## 📁 Directory Structure

```
~/openclaw/
├── docker-compose.yml
├── .env
└── Dockerfile

~/.openclaw/
├── openclaw.json
├── agents/
│   ├── main/agent.yaml
│   └── lena-image-processor/agent.yaml
├── devices/
│   ├── paired.json
│   └── pending.json
└── workspace/
```

---

## ⚠️ Troubleshooting

| Issue | Solution |
|-------|----------|
| `ACCESS_TOKEN_SCOPE_INSUFFICIENT` | VM needs `cloud-platform` scope |
| `token_mismatch` | Reset devices + clear browser |
| `Proxy headers from untrusted` | Add IP to `trustedProxies` |
| `Unknown model` | Add to providers config |
| 403 Forbidden (Cloudflare) | Check Access policy emails |
| Redirect loop | Check trustedProxies config |

### Nuclear Reset

```bash
sudo docker exec openclaw-openclaw-gateway-1 bash -c "
  echo {} > /home/node/.openclaw/devices/paired.json
  echo {} > /home/node/.openclaw/devices/pending.json
"
sudo docker compose restart openclaw-gateway
# Clear browser localStorage
```

---

## 🔧 Quick Commands

```bash
# SSH
gcloud compute ssh openclaw-gateway --zone=<ZONE>

# Logs
sudo docker logs openclaw-openclaw-gateway-1 --since 5m

# Restart
cd ~/openclaw && sudo docker compose restart openclaw-gateway

# Full rebuild
sudo docker compose down && sudo docker compose up -d

# Check tunnel
sudo docker logs cloudflared --since 5m
```

---

## License

MIT