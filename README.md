AI-Powered Outlook Email Automation with n8n and Microsoft Teams
This repository contains the complete configuration for a powerful email automation system built on a self-hosted n8n instance. The system automatically reads new Outlook emails, uses a Large Language Model (like Google's Gemini) to generate draft replies, sends an interactive notification to a Microsoft Teams channel for approval, and sends the email upon approval.

Features
Automated AI Drafts: When a new email arrives in a specified Outlook inbox, the workflow triggers.

Intelligent Replies: Uses a Gemini language model to generate context-aware, professional email replies.

Microsoft Teams Integration: Sends an interactive notification card to a Teams channel with the original email content and the AI-generated draft.

One-Click Approval: Allows for sending the email directly from Teams with an "Accept & Send" button or opening the draft in Outlook with an "Edit" button.

Self-Hosted & Secure: Runs on your own server using Docker, with traffic routed through Cloudflare for security and SSL.

Conversation Memory: Remembers the context of an email thread for more accurate follow-up replies.

Prerequisites
Before you begin, ensure you have the following:

A server (VPS or dedicated) with Docker and Docker Compose installed.

A registered domain name.

A Cloudflare account to manage your domain's DNS.

A Microsoft 365 account with administrative access to the Azure portal.

An API key for a Google Gemini (or another LLM like OpenAI).

Setup and Configuration
Follow these steps to deploy and configure the entire automation stack.

Step 1: Server and Domain Setup
Point Your Domain: In your domain registrar or Cloudflare, create an A record for a subdomain (e.g., n8n.yourdomain.com) and point it to your server's IP address.

Cloudflare SSL: In your Cloudflare dashboard, ensure the SSL/TLS encryption mode is set to Full or Full (Strict).

Step 2: Self-Host n8n with Docker
SSH into your server.

Create a new directory for your n8n project and cd into it.

mkdir n8n-automation
cd n8n-automation

Create a docker-compose.yml file and paste the content from the Configuration Files section below.

Important: Edit the environment variables in the docker-compose.yml file to match your custom domain.

environment:
  - N8N_HOST=n8n.yourdomain.com
  - N8N_PROTOCOL=https
  - WEBHOOK_URL=https://n8n.yourdomain.com/
  # ... other settings

Start the n8n container:

docker-compose up -d

This will download the n8n image and start it in the background.

Step 3: Configure Nginx as a Reverse Proxy
Install Nginx on your server if it's not already installed.

Create a new Nginx configuration file for your n8n instance.

sudo nano /etc/nginx/sites-available/n8n.conf

Paste the Nginx configuration from the Configuration Files section below into this file.

Important: Change server_name n8n.adplay-mobile.com; to server_name n8n.yourdomain.com;.

Enable the site and restart Nginx.

sudo ln -s /etc/nginx/sites-available/n8n.conf /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl restart nginx

You should now be able to access your n8n instance at https://n8n.yourdomain.com.

Step 4: Create Azure AD App for Microsoft Graph API
To allow n8n to access Outlook and Teams, you must register an application in Microsoft Entra ID (Azure AD).

Log in to the Azure portal with an admin account.

Navigate to Microsoft Entra ID > App registrations and click + New registration.

Name: n8n Email Automation (or similar).

Redirect URI: Select Web and enter your n8n callback URL: https://n8n.yourdomain.com/rest/oauth2-credential/callback

Click Register.

Get Client ID: On the app's overview page, copy the Application (client) ID.

Create Client Secret: Go to Certificates & secrets, click + New client secret, and copy the secret Value immediately. It is only shown once.

Add API Permissions: Go to API permissions > + Add a permission > Microsoft Graph > Delegated permissions. Add the following permissions:

Mail.ReadWrite

Mail.Send

offline_access

User.Read (for basic profile info)

Grant Admin Consent: Click the Grant admin consent for [Your Organization] button to approve the permissions.

Step 5: Configure n8n Workflows
This automation uses two workflows: one to process the email and create the draft, and a second to listen for the approval and send the email.

Import Workflows:

In your n8n instance, create two new blank workflows.

For each workflow, click the three dots (···) > "Import from file" and paste the corresponding JSON from the Workflows section below.

Configure Credentials:

In both workflows, you will see nodes for Microsoft Outlook, Microsoft Teams, and Google Gemini. Click on each of these nodes.

Create new credentials for each service, using the Azure App details (Client ID & Secret) for Outlook/Teams and your API key for Gemini.

Connect the Workflows:

Open the second workflow ("Teams Action: Send Approved Email"). Click on the Webhook node and copy its Production URL.

Open the first workflow ("Outlook AI Draft & Teams Alert"). Find the Send Adaptive Card to Teams node.

In the JSON parameter of this node, find the placeholder PASTE_YOUR_WEBHOOK_URL_HERE and replace it with the URL you just copied.

Activate Workflows:

Activate both workflows using the toggle switch at the top of the screen.

Your automation is now live!

How It Works
The system is split into two parts to handle the asynchronous nature of a human approval step.

Workflow 1: Draft & Alert

An Outlook Trigger fires when a new email arrives.

The AI Agent (using Gemini) reads the email and generates a reply.

An HTTP Request node creates a draft of this reply in Outlook.

A Teams node sends an Adaptive Card with the draft and two buttons ("Accept & Send", "Edit in Outlook") to a specified channel.

Workflow 2: Send on Approval

A Webhook node listens for incoming requests.

When the "Accept & Send" button is clicked in Teams, it sends the draftId to this webhook URL, triggering the workflow.

An HTTP Request node uses the draftId to call the Microsoft Graph API's /send endpoint, sending the email.

Workflows
1. Outlook_Email_Draft_Teams_Alert.json
This workflow reads incoming emails, generates an AI reply, creates a draft, and sends an interactive alert to Microsoft Teams.

{
  "name": "Outlook AI Draft & Teams Alert",
  "nodes": [
    {
      "parameters": {
        "method": "POST",
        "url": "=https://graph.microsoft.com/v1.0/me/messages/{{$node[\"Microsoft Outlook Trigger\"].json[\"id\"]}}/createReply",
        "authentication": "predefinedCredentialType",
        "nodeCredentialType": "microsoftOutlookOAuth2Api",
        "sendHeaders": true,
        "headerParameters": {
          "parameters": [
            {
              "name": "Content-Type",
              "value": "application/json"
            }
          ]
        },
        "sendBody": true,
        "specifyBody": "json",
        "jsonBody": "={\n  \"message\": {\n    \"body\": {\n      \"contentType\": \"HTML\",\n      \"content\": {{ JSON.stringify($node['AI Agent'].json.output) }}\n    }\n  }\n}",
        "options": {
          "response": {
            "response": {
              "neverError": true,
              "responseFormat": "json"
            }
          }
        }
      },
      "id": "15bfd425-a46f-4793-8534-d732d27e8d7c",
      "name": "Create Reply Draft (HTTP)",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.1,
      "position": [
        1340,
        -100
      ]
    },
    {
      "parameters": {
        "sessionIdType": "customKey",
        "sessionKey": "={{ $node['Microsoft Outlook Trigger'].json.conversationId }}"
      },
      "type": "@n8n/n8n-nodes-langchain.memoryBufferWindow",
      "typeVersion": 1.3,
      "position": [
        1000,
        120
      ],
      "id": "8fdeee3c-a1d8-4833-92c3-cbdefc48da07",
      "name": "Simple Memory"
    },
    {
      "parameters": {
        "modelName": "models/gemini-1.5-flash",
        "options": {
          "maxOutputTokens": 1000,
          "temperature": 0.7
        }
      },
      "type": "@n8n/n8n-nodes-langchain.lmChatGoogleGemini",
      "typeVersion": 1,
      "position": [
        840,
        100
      ],
      "id": "2a909b89-1ef5-4f34-969a-48767d9598f1",
      "name": "Google Gemini Chat Model"
    },
    {
      "parameters": {
        "promptType": "define",
        "text": "=You are a professional and helpful business assistant. Based on the email below, please write a clear, concise, and polite email BODY for a reply. Do not include a subject line. Format the response as HTML with proper paragraph tags.\n\n---START OF ORIGINAL EMAIL---\nSubject: {{ $nodes['Microsoft Outlook Trigger'].json.subject }}\nFrom: {{ $nodes['Microsoft Outlook Trigger'].json.from.emailAddress.name }} ({{ $nodes['Microsoft Outlook Trigger'].json.from.emailAddress.address }})\nReceived: {{ $nodes['Microsoft Outlook Trigger'].json.receivedDateTime }}\n\nEmail Content:\n{{ $node['Get Email Body'].json.body.content || $node['Microsoft Outlook Trigger'].json.bodyPreview || 'No content available' }}\n---END OF ORIGINAL EMAIL---\n\nIMPORTANT: Before responding, verify that you have received the email content above. If the email content appears to be missing or blank, respond with a request for the sender to resend their message with more details.\n\nPlease provide a professional email response in HTML format. Use <p> tags for paragraphs and ensure the response is complete and ready to send. The response should be appropriate for the context and tone of the original email.",
        "options": {
          "systemMessage": "You are a professional email assistant. Always respond with well-formatted HTML content suitable for email replies. If an email appears to have no content, politely ask the sender to resend with more details rather than assuming it's blank."
        }
      },
      "type": "@n8n/n8n-nodes-langchain.agent",
      "typeVersion": 2,
      "position": [
        880,
        -100
      ],
      "id": "4919cb23-a9cd-4142-b753-9ddbb248836e",
      "name": "AI Agent"
    },
    {
      "parameters": {
        "resource": "adaptiveCard",
        "teamId": "YOUR_TEAM_ID",
        "channelId": "YOUR_CHANNEL_ID",
        "json": "=\n{\n\t\"type\": \"AdaptiveCard\",\n\t\"$schema\": \"http://adaptivecards.io/schemas/adaptive-card.json\",\n\t\"version\": \"1.5\",\n\t\"body\": [\n\t\t{\n\t\t\t\"type\": \"Container\",\n\t\t\t\"style\": \"emphasis\",\n\t\t\t\"items\": [\n\t\t\t\t{\n\t\t\t\t\t\"type\": \"TextBlock\",\n\t\t\t\t\t\"text\": \"📧 New Email Received & AI Draft Generated\",\n\t\t\t\t\t\"weight\": \"Bolder\",\n\t\t\t\t\t\"size\": \"Medium\"\n\t\t\t\t}\n\t\t\t]\n\t\t},\n\t\t{\n\t\t\t\"type\": \"FactSet\",\n\t\t\t\"facts\": [\n\t\t\t\t{\n\t\t\t\t\t\"title\": \"From\",\n\t\t\t\t\t\"value\": \"{{ $node['Microsoft Outlook Trigger'].json.from?.emailAddress?.address ?? 'System Notification' }}\"\n\t\t\t\t},\n\t\t\t\t{\n\t\t\t\t\t\"title\": \"Subject\",\n\t\t\t\t\t\"value\": \"{{ $node['Microsoft Outlook Trigger'].json.subject }}\"\n\t\t\t\t},\n\t\t\t\t{\n\t\t\t\t\t\"title\": \"Received\",\n\t\t\t\t\t\"value\": \"{{ $node['Microsoft Outlook Trigger'].json.receivedDateTime ? new Date($node['Microsoft Outlook Trigger'].json.receivedDateTime).toLocaleString('en-US', { timeZone: 'Asia/Dhaka' }) : 'Date Not Available' }} (BST)\"\n\t\t\t\t}\n\t\t\t]\n\t\t},\n\t\t{\n\t\t\t\"type\": \"TextBlock\",\n\t\t\t\"text\": \"🤖 **AI-Generated Reply Draft:**\",\n\t\t\t\"wrap\": true,\n\t\t\t\"weight\": \"Bolder\"\n\t\t},\n\t\t{\n\t\t\t\"type\": \"RichTextBlock\",\n\t\t\t\"inlines\": [\n\t\t\t\t{\n\t\t\t\t\t\"type\": \"TextRun\",\n\t\t\t\t\t\"text\": \"{{ $node['AI Agent'].json.output }}\"\n\t\t\t\t}\n\t\t\t]\n\t\t}\n\t],\n\t\"actions\": [\n\t\t{\n\t\t\t\"type\": \"Action.Http\",\n\t\t\t\"title\": \"✅ Accept & Send\",\n\t\t\t\"method\": \"POST\",\n\t\t\t\"url\": \"PASTE_YOUR_WEBHOOK_URL_HERE\",\n\t\t\t\"body\": \"{ \\\"draftId\\\": \\\"{{ $node['Create Reply Draft (HTTP)'].json.id }}\\\" }\",\n\t\t\t\"style\": \"positive\"\n\t\t},\n\t\t{\n\t\t\t\"type\": \"Action.OpenUrl\",\n\t\t\t\"title\": \"✏️ Edit in Outlook\",\n\t\t\t\"url\": \"{{ $node['Create Reply Draft (HTTP)'].json.webLink }}\"\n\t\t}\n\t]\n}",
        "options": {}
      },
      "id": "bac79384-ce90-45e9-8cb1-f894f03e6e56",
      "name": "Send Adaptive Card to Teams",
      "type": "n8n-nodes-base.microsoftTeams",
      "typeVersion": 2,
      "position": [
        1560,
        -100
      ]
    },
    {
      "parameters": {
        "pollTimes": {
          "item": [
            {
              "mode": "everyMinute"
            }
          ]
        },
        "filters": {
          "readStatus": "unread"
        },
        "options": {
          "downloadAttachments": false
        }
      },
      "type": "n8n-nodes-base.microsoftOutlookTrigger",
      "typeVersion": 1,
      "position": [
        460,
        -100
      ],
      "id": "30d3b380-b9b6-4b5d-a282-9fd6ef6657b4",
      "name": "Microsoft Outlook Trigger"
    },
    {
      "parameters": {
        "url": "=https://graph.microsoft.com/v1.0/me/messages/{{$node[\"Microsoft Outlook Trigger\"].json[\"id\"]}}?$select=body",
        "authentication": "predefinedCredentialType",
        "nodeCredentialType": "microsoftOutlookOAuth2Api",
        "options": {
          "response": {
            "response": {
              "neverError": true,
              "responseFormat": "json"
            }
          }
        }
      },
      "id": "47db264a-ad4a-4656-817a-2db3ccbe2ed4",
      "name": "Get Email Body",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.1,
      "position": [
        680,
        -100
      ]
    }
  ],
  "connections": {
    "Simple Memory": {
      "ai_memory": [
        [
          {
            "node": "AI Agent",
            "type": "ai_memory",
            "index": 0
          }
        ]
      ]
    },
    "Google Gemini Chat Model": {
      "ai_languageModel": [
        [
          {
            "node": "AI Agent",
            "type": "ai_languageModel",
            "index": 0
          }
        ]
      ]
    },
    "AI Agent": {
      "main": [
        [
          {
            "node": "Create Reply Draft (HTTP)",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Create Reply Draft (HTTP)": {
      "main": [
        [
          {
            "node": "Send Adaptive Card to Teams",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Microsoft Outlook Trigger": {
      "main": [
        [
          {
            "node": "Get Email Body",
            "type": "main",
            "index": 0
          }
        ]
      ]
    },
    "Get Email Body": {
      "main": [
        [
          {
            "node": "AI Agent",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}

2. Outlook_Email_Create_Send.json (Webhook)
This workflow listens for the "Accept & Send" action from Teams and sends the corresponding draft email.

{
  "name": "Teams Action: Send Approved Email",
  "nodes": [
    {
      "parameters": {},
      "id": "19b88f34-8c17-4581-8b38-65158e08d248",
      "name": "Listen for 'Accept' Click",
      "type": "n8n-nodes-base.webhook",
      "typeVersion": 1,
      "position": [
        20,
        220
      ],
      "webhookId": "YOUR_WEBHOOK_ID_WILL_BE_GENERATED_HERE"
    },
    {
      "parameters": {
        "url": "={{ `https://graph.microsoft.com/v1.0/me/messages/${$json.body.draftId}/send` }}",
        "authentication": "predefinedCredentialType",
        "nodeCredentialType": "microsoftOutlookOAuth2Api",
        "options": {}
      },
      "id": "2701f5ae-7b6c-4b5a-935f-15579f168ac1",
      "name": "Send the Draft Email",
      "type": "n8n-nodes-base.httpRequest",
      "typeVersion": 4.1,
      "position": [
        260,
        220
      ]
    }
  ],
  "connections": {
    "Listen for 'Accept' Click": {
      "main": [
        [
          {
            "node": "Send the Draft Email",
            "type": "main",
            "index": 0
          }
        ]
      ]
    }
  }
}

Configuration Files
docker-compose.yml
services:
  n8n:
    image: n8nio/n8n
    restart: unless-stopped
    ports:
      # Expose n8n on port 5678, but only accessible from the host server itself (localhost).
      - "127.0.0.1:5678:5678"
    environment:
      # --- SETTINGS FOR NGINX REVERSE PROXY ---
      - N8N_HOST=n8n.yourdomain.com
      - N8N_PROTOCOL=https
      - WEBHOOK_URL=https://n8n.yourdomain.com/

      # --- RECOMMENDED SETTINGS ---
      - GENERIC_TIMEZONE=Asia/Dhaka
      
      # === INCREASE NODE.JS MEMORY LIMIT ===
      # This allows the n8n process to use up to 4GB of RAM, preventing crashes.
      - NODE_OPTIONS=--max-old-space-size=4096

      # === TASK RUNNERS (RECOMMENDED) ===
      # Enable task runners to avoid deprecation warnings and improve performance
      - N8N_RUNNERS_ENABLED=true

      # === FILE PERMISSIONS ===
      # Automatically enforce correct file permissions
      - N8N_ENFORCE_SETTINGS_FILE_PERMISSIONS=true

      # === CONNECTION SETTINGS ===
      # Improve WebSocket connection stability
      - N8N_PUSH_BACKEND=websocket
      - N8N_DISABLE_UI=false

    volumes:
      # Creates a persistent volume to store all your n8n data
      - n8n_data:/home/node/.n8n

volumes:
  n8n_data:

nginx.conf
server {
    listen 80;
    listen [::]:80;

    server_name n8n.yourdomain.com;

    location / {
        proxy_pass http://localhost:5678;
        proxy_set_header Connection '';
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_http_version 1.1;
        proxy_buffering off;
        
        # === PROXY TIMEOUTS ===
        proxy_connect_timeout 300s;
        proxy_read_timeout 300s;
        proxy_send_timeout 300s;
        
        chunked_transfer_encoding off;
    }

    # === WEBSOCKET SUPPORT FOR n8n ===
    location /rest/push {
        proxy_pass http://localhost:5678;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_read_timeout 86400;
    }
}
