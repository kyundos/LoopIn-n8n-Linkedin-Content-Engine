# 🚀 LoopIn: n8n LinkedIn Content Engine

> 🇬🇧 [Read the English documentation](README.md)

Un sistema di automazione a micro-servizi basato su **n8n** per la curatela e la pubblicazione di contenuti su LinkedIn. LoopIn legge feed RSS di settore, sfrutta l'intelligenza artificiale di Claude (Anthropic) per generare post ottimizzati e utilizza Telegram per un'approvazione "Human-in-the-Loop" prima della pubblicazione via API su LinkedIn.

## 📑 Indice

1. [Architettura del Sistema](#architettura)
2. [⚠️ Avviso di Sicurezza Fondamentale](#sicurezza)
3. [Prerequisiti](#prerequisiti)
4. [Guida al Setup - Fase 1: Testing & Local (Plug-and-Play)](#fase-1)
5. [Guida al Setup - Fase 2: Produzione](#fase-2)
6. [Importazione dei Workflow](#importazione)

---

<a name="architettura"></a>

## 🏗 Architettura del Sistema (Human-in-the-Loop)

Il sistema è diviso in due workflow interconnessi:

1. **Content Generator (Schedulato):**
   - Raccoglie gli ultimi articoli da 5 feed RSS.
   - Filtra le notizie tramite Regex (Keyword specifiche + finestra di 24h) e ne seleziona al massimo 3.
   - Passa i contenuti a **Claude (Anthropic)** con un prompt specifico per generare un post LinkedIn (privo di Markdown).
   - Salva il post come backup in formato `.md` su **GitHub**.
   - Invia una notifica a **Telegram** con la bozza e due bottoni inline: ✅ _Approva_ o ❌ _Scarta_.

2. **Listener & Publisher (Webhook):**
   - Rimane in ascolto del Webhook da Telegram.
   - Se l'utente clicca **Scarta**: Il workflow si interrompe e aggiorna il messaggio Telegram segnalando che la bozza è stata ignorata.
   - Se l'utente clicca **Approva**: Invia il payload JSON all'endpoint API `ugcPosts` di LinkedIn, pubblica il post e aggiorna il messaggio Telegram con la conferma di avvenuta pubblicazione.

---

<a name="sicurezza"></a>

## ⚠️ Avviso di Sicurezza Fondamentale

**MAI committare dati sensibili su GitHub!**

- Non caricare mai il file `.env` sulla tua repository pubblica.
- Non caricare mai la cartella `n8n_data/` (contiene il database SQLite con le tue credenziali).
- **Sanificazione JSON:** Se modifichi i workflow, esportali sempre dalla UI di n8n. n8n rimuove automaticamente le credenziali salvate nel Credential Manager. Tuttavia, se scrivi _direttamente in chiaro_ un token dentro un nodo (es. un nodo HTTP Request), questo **verrà esportato**. Usa sempre le Credenziali di n8n!

---

<a name="prerequisiti"></a>

## 📋 Prerequisiti

Prima di iniziare, assicurati di avere:

1. **Docker e Docker Compose** installati.
2. **Anthropic API Key**: Per il nodo Claude.
3. **Telegram Bot Token**: Creato tramite [@BotFather](https://t.me/botfather) su Telegram.
4. **GitHub Personal Access Token (PAT)**: Con permessi di lettura/scrittura per la repo dove salverai le bozze.
5. **LinkedIn App**: Creata sul [LinkedIn Developer Portal](https://www.linkedin.com/developers/) con i permessi per l'API `ugcPosts` (richiede autenticazione OAuth2 o Access Token generato).

---

<a name="fase-1"></a>

## 🛠 Guida al Setup - Fase 1: Testing & Local

In questa fase, avvieremo n8n in locale. Poiché Telegram ha bisogno di contattare il nostro sistema tramite Webhook, utilizzeremo un tunnel Cloudflare integrato per esporre la nostra istanza locale in pochi secondi senza toccare router o firewall.

1. **Clona la repository e prepara l'ambiente:**

   ```bash
   git clone https://github.com/kyundos/LoopIn-n8n-Linkedin-Content-Engine.git
   cd LoopIn-n8n-Linkedin-Content-Engine
   cp .env.example .env
   ```

2. **Avvia il cluster locale:**

   ```bash
   docker-compose -f docker-compose.local.yml up -d
   ```

3. **Recupera l'URL del Tunnel Webhook:**
   Il container `cloudflared` sta generando un URL pubblico temporaneo. Trovalo leggendo i log:

   ```bash
   docker logs loopin_tunnel_local 2>&1 | grep "trycloudflare.com"
   ```

   Copia l'URL risultante (es. `https://parola-random.trycloudflare.com`).

4. **Aggiorna il file .env e riavvia:**
   Apri il file `.env` e inserisci l'URL copiato nella variabile `WEBHOOK_URL`. Riavvia il container di n8n per applicare la modifica:
   ```bash
   docker restart loopin_n8n_local
   ```

Ora il tuo n8n locale è pronto per ricevere chiamate dal bot Telegram all'indirizzo pubblico fornito da Cloudflare! Accedi a n8n su `http://localhost:5678`.

---

<a name="fase-2"></a>

## 🚀 Guida al Setup - Fase 2: Produzione

Una volta testato il sistema, è il momento di portarlo in produzione su una VPS o su un NAS (es. Synology) eliminando il tunnel effimero.

1. **Ferma l'ambiente di test:**

   ```bash
   docker-compose -f docker-compose.local.yml down
   ```

2. **Configura il tuo Dominio e Reverse Proxy:**
   Assicurati di avere un dominio (es. `n8n.tuodominio.com`) puntato sull'IP del tuo server, gestito da un Reverse Proxy (Nginx, Traefik, Caddy) che gestisce i certificati SSL.

3. **Aggiorna il .env:**
   Modifica il `.env` sostituendo il vecchio link Cloudflare con il tuo dominio reale nelle variabili `WEBHOOK_URL` e `N8N_HOST`.

4. **Avvia in Produzione:**
   ```bash
   docker-compose -f docker-compose.prod.yml up -d
   ```

---

<a name="importazione"></a>

## 📥 Importazione e Setup dei Workflow in n8n

1. Accedi alla UI di n8n.
2. Vai su **Workflows > Add Workflow**.
3. Clicca sui tre puntini in alto a destra e seleziona **Import from File**.
4. Importa prima il file `workflows/1-Content-Generator.json`.
5. Importa il file `workflows/2-Listener-Publisher.json`.
6. 🔑 **Configura le Credenziali (Obbligatorio):** In entrambi i workflow, apri i nodi che richiedono accesso esterno (Telegram, Anthropic, GitHub, LinkedIn). Clicca sul menu a tendina delle _Credentials_ e crea le relative connessioni inserendo i dati richiesti nei [Prerequisiti](#prerequisiti).
7. 📰 **Personalizza Fonti e Filtri:** - Apri il nodo **"Code in JavaScript"** nel workflow 1 per sostituire gli URL di default con i feed RSS del tuo settore.
   - Apri il nodo **"Filter"** per modificare la Regex e inserire le keyword pertinenti alla tua nicchia, scartando così le notizie irrilevanti.
8. ✍️ **Modifica del Prompt e della Lingua:** Di default, l'AI è istruita per scrivere post in _inglese_. Apri il nodo **"Message a model"** (Anthropic) nel primo workflow per tradurre il prompt nella tua lingua o cambiare il tono di voce e il ruolo del bot (es. da Software Engineer a Marketing Manager).
9. Attiva (_Activate_) i workflow tramite lo switch in alto a destra.

**Fatto!** Il tuo motore di contenuti LinkedIn automatizzato è pronto a girare.
