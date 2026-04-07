# 🚀 LoopIn: n8n LinkedIn Content Engine

> 🇮🇹 [Leggi la documentazione in Italiano](README-IT.md)

A micro-services automation system based on **n8n** for curating and publishing content on LinkedIn. LoopIn reads industry RSS feeds, leverages Claude's AI (Anthropic) to generate optimized posts, and uses Telegram for a "Human-in-the-Loop" approval before publishing via the LinkedIn API.

## 📑 Table of Contents

1. [System Architecture](#architecture)
2. [⚠️ Crucial Security Warning](#security)
3. [Prerequisites](#prerequisites)
4. [Setup Guide - Phase 1: Testing & Local (Plug-and-Play)](#phase-1)
5. [Setup Guide - Phase 2: Production](#phase-2)
6. [Importing Workflows](#importing)

---

<a name="architecture"></a>

## 🏗 System Architecture (Human-in-the-Loop)

The system is divided into two interconnected workflows:

1. **Content Generator (Scheduled):**
   - Collects the latest articles from 5 RSS feeds.
   - Filters news via Regex (Specific keywords + 24h window) and selects a maximum of 3 articles.
   - Passes the content to **Claude (Anthropic)** with a specific prompt to generate a LinkedIn post (without Markdown).
   - Saves the post as a draft in `.md` format on **GitHub**.
   - Sends a notification to **Telegram** with the draft and two inline buttons: ✅ _Approve_ or ❌ _Reject_.

2. **Listener & Publisher (Webhook):**
   - Listens for the Webhook from Telegram.
   - If the user clicks **Reject**: Deletes the draft from GitHub and updates the Telegram message.
   - If the user clicks **Approve**: Sends the JSON payload to the LinkedIn `ugcPosts` API endpoint, publishes the post, and updates the Telegram message with the confirmation.

---

<a name="security"></a>

## ⚠️ Crucial Security Warning

**NEVER commit sensitive data to GitHub!**

- Never upload the `.env` file to your public repository.
- Never upload the `n8n_data/` folder (it contains the SQLite database with your credentials).
- **JSON Sanitization:** If you modify the workflows, always export them from the n8n UI. n8n automatically removes credentials saved in the Credential Manager. However, if you write a token _in plain text_ directly inside a node (e.g., an HTTP Request node), it **will be exported**. Always use n8n Credentials!

---

<a name="prerequisites"></a>

## 📋 Prerequisites

Before you begin, ensure you have:

1. **Docker and Docker Compose** installed.
2. **Anthropic API Key**: For the Claude node.
3. **Telegram Bot Token**: Created via [@BotFather](https://t.me/botfather) on Telegram.
4. **GitHub Personal Access Token (PAT)**: With read/write permissions for the repo where you will save drafts.
5. **LinkedIn App**: Created on the [LinkedIn Developer Portal](https://www.linkedin.com/developers/) with permissions for the `ugcPosts` API (requires OAuth2 authentication or a generated Access Token).

---

<a name="phase-1"></a>

## 🛠 Setup Guide - Phase 1: Testing & Local

In this phase, we will start n8n locally. Since Telegram needs to contact our system via Webhook, we'll use an integrated Cloudflare tunnel to expose our local instance in seconds without touching routers or firewalls.

1. **Clone the repository and prepare the environment:**

   ```bash
   git clone [https://github.com/YOUR-USER/LoopIn-n8n-Linkedin-Content-Engine.git](https://github.com/YOUR-USER/LoopIn-n8n-Linkedin-Content-Engine.git)
   cd LoopIn-n8n-Linkedin-Content-Engine
   cp .env.example .env
   ```

2. **Start the local cluster:**

   ```bash
   docker-compose -f docker-compose.local.yml up -d
   ```

3. **Retrieve the Webhook Tunnel URL:**
   The `cloudflared` container is generating a temporary public URL. Find it by reading the logs:

   ```bash
   docker logs loopin_tunnel_local 2>&1 | grep "trycloudflare.com"
   ```

   Copy the resulting URL (e.g., `https://random-word.trycloudflare.com`).

4. **Update the .env file and restart:**
   Open the `.env` file and insert the copied URL into the `WEBHOOK_URL` variable. Restart the n8n container to apply the change:
   ```bash
   docker restart loopin_n8n_local
   ```

Now your local n8n is ready to receive calls from the Telegram bot at the public address provided by Cloudflare! Access n8n at `http://localhost:5678`.

---

<a name="phase-2"></a>

## 🚀 Setup Guide - Phase 2: Production

Once the system is tested, it's time to deploy it to production on a VPS or NAS (e.g., Synology) by eliminating the ephemeral tunnel.

1. **Stop the test environment:**

   ```bash
   docker-compose -f docker-compose.local.yml down
   ```

2. **Configure your Domain and Reverse Proxy:**
   Ensure you have a domain (e.g., `n8n.yourdomain.com`) pointing to your server's IP, managed by a Reverse Proxy (Nginx, Traefik, Caddy) that handles SSL certificates.

3. **Update the .env:**
   Edit the `.env` file, replacing the old Cloudflare link with your real domain in the `WEBHOOK_URL` and `N8N_HOST` variables.

4. **Start in Production:**
   ```bash
   docker-compose -f docker-compose.prod.yml up -d
   ```

---

<a name="importing"></a>

## 📥 Importing Workflows in n8n

1. Access the n8n UI.
2. Go to **Workflows > Add Workflow**.
3. Click the three dots in the top right corner and select **Import from File**.
4. Import `workflows/1-Content-Generator.json` first and configure it (insert your RSS feeds).
5. Import `workflows/2-Listener-Publisher.json`.
6. **Important:** In both workflows, open the nodes (Telegram, Anthropic, GitHub, LinkedIn) and create/select the credentials from the n8n **Credential Manager**.
7. Enable (_Activate_) the workflows using the toggle in the top right corner.

**Done!** Your automated LinkedIn content engine is ready to run.
