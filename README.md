

```markdown
# ✨ AI-Powered Outlook Email Automation with n8n

This repository provides a complete automation stack using **n8n**, Microsoft Outlook, Microsoft Teams, and **Google Gemini** (or another LLM). It reads incoming Outlook emails, generates a professional AI-powered draft, sends it to Teams for approval, and sends the email after one-click confirmation.

---

## 🚀 Features

- **Automated AI Drafts**: Auto-generates email replies using LLM when a new Outlook email arrives.
- **Intelligent Replies**: Context-aware, business-grade replies using Gemini or OpenAI.
- **Microsoft Teams Integration**: Sends an interactive adaptive card with the draft email.
- **One-Click Approval**: Instantly send or edit the draft via Teams.
- **Secure & Self-Hosted**: Dockerized deployment with Cloudflare SSL & reverse proxy.
- **Conversation Memory**: Retains context for threaded conversations.

---

## ✅ Prerequisites

Make sure you have the following:

- A server (VPS or dedicated) with **Docker** & **Docker Compose**
- A registered **domain name**
- A **Cloudflare** account for DNS and SSL
- A **Microsoft 365** account with admin access
- An **API Key** for Google Gemini or OpenAI

---

## 🛠️ Setup and Configuration

### Step 1: Server & Domain

- Create an A record (e.g. `n8n.yourdomain.com`) → Point to your server IP.
- On Cloudflare, set **SSL/TLS** to **Full** or **Full (Strict)**.

### Step 2: Deploy n8n with Docker

```bash
mkdir n8n-automation && cd n8n-automation
# Paste docker-compose.yml content here
docker-compose up -d
```

### Step 3: Setup Nginx Reverse Proxy

```bash
sudo nano /etc/nginx/sites-available/n8n.conf
# Paste nginx config here, update domain
sudo ln -s /etc/nginx/sites-available/n8n.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl restart nginx
```

You can now access n8n at: `https://n8n.yourdomain.com`

---

### Step 4: Azure App Registration

1. Go to [Azure Portal](https://portal.azure.com)
2. Register a new App under **Microsoft Entra ID** → **App registrations**
3. Redirect URI: `https://n8n.yourdomain.com/rest/oauth2-credential/callback`
4. Get **Client ID** and **Client Secret**
5. Add these **Microsoft Graph API** permissions:
   - `Mail.ReadWrite`
   - `Mail.Send`
   - `User.Read`
   - `offline_access`
6. Grant Admin Consent.

---

### Step 5: Configure n8n Workflows

- Import the two workflows:
  - `Outlook_Email_Draft_Teams_Alert.json`
  - `Outlook_Email_Create_Send.json`

- Set up credentials for:
  - Microsoft Outlook (OAuth2)
  - Microsoft Teams
  - Google Gemini / OpenAI

- Connect the webhook URL from `Send Approved Email` workflow into the Teams card in `AI Draft & Teams Alert`.

- **Activate both workflows** – You're live!

---

## 🔄 Workflows

### 1. **Outlook Email Draft + Teams Alert**

- Triggers on new unread Outlook email.
- Retrieves body and context.
- Uses Gemini/OpenAI to generate a reply.
- Creates a draft via Microsoft Graph API.
- Sends Adaptive Card to Microsoft Teams with buttons:
  - ✅ **Accept & Send**
  - ✏️ **Edit in Outlook**

### 2. **Teams Action → Send Email**

- Triggered when the user clicks ✅ in Teams.
- Sends the previously created draft via Microsoft Graph API.

---

## ⚙️ Configuration Files

### docker-compose.yml

```yaml
services:
  n8n:
    image: n8nio/n8n
    restart: unless-stopped
    ports:
      - "127.0.0.1:5678:5678"
    environment:
      - N8N_HOST=n8n.yourdomain.com
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://n8n.yourdomain.com/
      - GENERIC_TIMEZONE=Asia/Dhaka
      - NODE_OPTIONS=--max-old-space-size=4096
      - N8N_RUNNERS_ENABLED=true
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true
      - N8N_PUSH_BACKEND=websocket
    volumes:
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:
```

---

### nginx.conf

```nginx
server {
    listen 80;
    server_name n8n.yourdomain.com;

    location / {
        proxy_pass http://localhost:5678;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_read_timeout 86400;
    }
}
```

---

## 🧠 Example Prompt to Gemini / OpenAI

```text
You are a professional and helpful business assistant. Please read the following email and write a clear, concise, and polite reply. Format the response in HTML with proper <p> tags.

Subject: {{ $node["Microsoft Outlook Trigger"].json.subject }}
From: {{ $node["Microsoft Outlook Trigger"].json.from.emailAddress.address }}

Content:
{{ $node["Get Email Body"].json.body.content }}
```

---

## 📬 Adaptive Card Preview (Microsoft Teams)

| Action | Description |
|-------|-------------|
| ✅ Accept & Send | Instantly sends draft reply |
| ✏️ Edit in Outlook | Opens the draft in Outlook for manual review |

> The card also shows original sender, subject, time received, and AI-generated content.

---

## ✅ Final Notes

- All communication happens securely via HTTPS and OAuth2.
- All workflows are fully modular – adapt or expand as needed.
- You can enhance this further by storing conversation history or connecting to a CRM.

---


---

> Made with 💙 using [n8n.io](https://n8n.io) and the power of AI.
```
