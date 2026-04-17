# Bring Your Own LLM to Alteryx GenAI Tools

A comprehensive, GitHub-ready build guide for connecting Alteryx Designer Desktop GenAI tools to a customer-hosted, OpenAI-compatible LLM endpoint using **GCP + Ubuntu + Caddy + Ollama**.

## What this guide builds

This guide documents a working reference pattern in which:

- **Alteryx One** stores an **OpenAI Compatible** LLM connection.
- **Designer Desktop** uses the **Prompt Tool** to call that connection.
- **AIMS** (Alteryx AI Model Service) sends requests to a **public HTTPS `/v1` endpoint**.
- The public endpoint lands on **Caddy** running on a **GCP Ubuntu VM**.
- **Caddy** terminates TLS, validates a bearer token, and reverse proxies to **Ollama**.
- **Ollama** serves an OpenAI-compatible API on **`localhost:11434`** and invokes a local chat model.

This pattern keeps the model runtime private to the VM while presenting a standards-friendly HTTPS entry point to Alteryx.

---

## Architecture at a glance

```mermaid
flowchart LR
    U[Designer Desktop Prompt Tool] --> A[AIMS / Alteryx AI Model Service]
    A -->|HTTPS /v1/models\nHTTPS /v1/chat/completions| E[Public HTTPS Endpoint]
    E --> C[Caddy Reverse Proxy]
    C -->|localhost:11434| O[Ollama API]
    O --> M[Local model\nollama/llama3:8b]
```

### Runtime request path

1. User configures the **Prompt Tool** in Designer Desktop.
2. Designer sends the run request to **AIMS**.
3. **AIMS** retrieves connection metadata from the **Alteryx Connections Service**.
4. **AIMS** calls the configured **OpenAI-compatible HTTPS endpoint**.
5. **Caddy** receives the request, terminates TLS, validates the bearer token, and proxies to **Ollama** on `localhost:11434`.
6. **Ollama** invokes the local model.
7. The response returns to **Caddy**, then **AIMS**, then back to the Prompt Tool output.

---

## Why this architecture

This design solves four practical problems:

1. **Alteryx compatibility**  
   Alteryx One supports **OpenAI Compatible** LLM connections that require a **Base URL** and **API key**. The Prompt Tool then sends prompts to the selected model and returns the response as output. ([Alteryx docs](https://help.alteryx.com/current/en/designer/tools/genai/create-llm-connections.html), [Prompt Tool docs](https://help.alteryx.com/current/en/designer/tools/genai/prompt-tool.html))

2. **Private model hosting**  
   Ollama provides an OpenAI-compatible API including **`/v1/chat/completions`** and **`/v1/models`**, which makes it a practical runtime for a customer-hosted endpoint. ([Ollama docs](https://docs.ollama.com/openai))

3. **Security boundary**  
   Ollama is kept on **localhost only**, while **Caddy** becomes the only public ingress.

4. **Model naming control**  
   Ollama supports **`ollama cp`** so you can expose a model under the exact name your tooling expects. This is critical when Alteryx only displays models that match the naming convention it recognises. ([Ollama docs](https://docs.ollama.com/openai))

---

## Target environment

This guide assumes:

- **Cloud**: Google Cloud Platform (GCP)
- **OS**: Ubuntu 24.04 LTS
- **Reverse proxy**: Caddy
- **Inference server**: Ollama
- **Example model**: `llama3:8b`, exposed as `ollama/llama3:8b`
- **Public hostname**: a Duck DNS hostname such as `alteryxllmdemo.duckdns.org`

> You can adapt the same pattern to Azure or AWS. The important parts are the OpenAI-compatible endpoint, TLS, authentication, and keeping the model runtime private.

---

## Prerequisites

### Alteryx side

- Alteryx One workspace with **GenAI tools enabled**
- User role that can create and use LLM connections
- Designer Desktop version that supports GenAI tools
- Access to create an **OpenAI Compatible** connection in Alteryx One

### Infrastructure side

- GCP project with billing enabled
- Ability to create a VM, firewall rules, and a static public IP
- A public DNS hostname, for example via **Duck DNS**

### Local admin side

- SSH access to the Ubuntu VM
- Ability to edit systemd units and install packages

---

## Step 1: Provision the Ubuntu VM on GCP

### Recommended shape

For a demo or low-volume reference build:

- **Machine type**: `e2-standard-4`
- **Disk**: 100 GB
- **OS**: Ubuntu 24.04 LTS
- **Public IP**: static external IPv4

### Network requirements

Open only what is required:

- **443/tcp**: public HTTPS to Caddy
- **80/tcp**: HTTP redirect and certificate issuance
- **22/tcp**: SSH, ideally restricted to your public IP

Do **not** expose Ollama publicly on `11434`.

### DNS

Point your hostname at the VM’s public static IP.

Example:

- Hostname: `alteryxllmdemo.duckdns.org`
- Public IP: `34.x.x.x`

Validate before proceeding:

```bash
nslookup alteryxllmdemo.duckdns.org
```

You want it to resolve to the VM’s public IP.

---

## Step 2: Update the VM and install Ollama

SSH to the VM:

```bash
sudo apt update
sudo apt upgrade -y
sudo apt install -y curl jq
```

Install Ollama:

```bash
curl -fsSL https://ollama.com/install.sh | sh
```

Check the service:

```bash
sudo systemctl status ollama --no-pager
```

If required:

```bash
sudo systemctl enable ollama
sudo systemctl start ollama
```

---

## Step 3: Bind Ollama to localhost only

By default, Ollama binds `127.0.0.1:11434`, and that is what we want for this pattern. Ollama documents that its server bind address is controlled by `OLLAMA_HOST`, and that on Linux the value should be set through the systemd service environment. ([Ollama FAQ](https://docs.ollama.com/faq))

Create or edit a systemd override:

```bash
sudo systemctl edit ollama
```

Add:

```ini
[Service]
Environment="OLLAMA_HOST=127.0.0.1:11434"
```

Then reload and restart:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ollama
sudo systemctl status ollama --no-pager
```

Test locally:

```bash
curl -s http://127.0.0.1:11434/v1/models | jq .
```

If Ollama is healthy, you should see a JSON response.

---

## Step 4: Pull the model and create the Alteryx-friendly alias

Pull the base model:

```bash
ollama pull llama3:8b
```

Check what is installed:

```bash
ollama list
```

Create an alias that matches the exposed provider-style name:

```bash
ollama cp llama3:8b ollama/llama3:8b
```

Check again:

```bash
ollama list
```

Test the alias with the OpenAI-compatible endpoint:

```bash
curl -s http://127.0.0.1:11434/v1/models | jq .
```

You should now see both the base model and the alias, for example:

```json
{
  "object": "list",
  "data": [
    {
      "id": "ollama/llama3:8b",
      "object": "model"
    },
    {
      "id": "llama3:8b",
      "object": "model"
    }
  ]
}
```

### Optional cleanup

If you want Alteryx to see only the aliased model name, remove the base one after validating the alias works:

```bash
ollama rm llama3:8b
```

Only do this if you are certain you want the alias to be the only exposed model.

---

## Step 5: Install Caddy

Install the official Caddy package:

```bash
sudo apt install -y debian-keyring debian-archive-keyring apt-transport-https curl
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/gpg.key' | sudo gpg --dearmor -o /usr/share/keyrings/caddy-stable-archive-keyring.gpg
curl -1sLf 'https://dl.cloudsmith.io/public/caddy/stable/debian.deb.txt' | sudo tee /etc/apt/sources.list.d/caddy-stable.list
sudo chmod o+r /usr/share/keyrings/caddy-stable-archive-keyring.gpg
sudo chmod o+r /etc/apt/sources.list.d/caddy-stable.list
sudo apt update
sudo apt install -y caddy
```

Check service status:

```bash
sudo systemctl status caddy --no-pager
```

---

## Step 6: Configure Caddy as the public HTTPS endpoint

Back up the default config:

```bash
sudo cp /etc/caddy/Caddyfile /etc/caddy/Caddyfile.bak
```

Edit the Caddyfile:

```bash
sudo nano /etc/caddy/Caddyfile
```

### Minimal working config

Use this as the baseline:

```caddy
alteryxllmdemo.duckdns.org {
    reverse_proxy 127.0.0.1:11434 {
        header_up Host localhost:11434
    }
}
```

### Why the `Host` header override matters

Ollama’s FAQ explicitly shows that when you put Ollama behind a proxy such as Nginx, you should forward traffic to `localhost:11434` and set `Host localhost:11434`. Without this override, proxied requests may be rejected. ([Ollama FAQ](https://docs.ollama.com/faq))

Validate the Caddy config:

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
```

Restart Caddy:

```bash
sudo systemctl restart caddy
sudo systemctl status caddy --no-pager
```

Test the public endpoint from the VM:

```bash
curl -vk https://alteryxllmdemo.duckdns.org/v1/models
```

Test it again with JSON formatting:

```bash
curl -s https://alteryxllmdemo.duckdns.org/v1/models | jq .
```

If DNS, firewall rules, and certificate issuance are correct, Caddy will obtain and serve a certificate for the hostname.

---

## Step 7: Add bearer-token validation in Caddy

At this point the endpoint is working, but it is public. The next step is to require an `Authorization` header.

Generate a token:

```bash
python3 -c "import secrets; print(secrets.token_urlsafe(32))"
```

For example:

```text
cTFYwz7fh3_iqZqDEOXAdT6F-movvLX4kCoNEzUcXX0
```

Update `/etc/caddy/Caddyfile`:

```caddy
alteryxllmdemo.duckdns.org {
    @authorized header Authorization "Bearer cTFYwz7fh3_iqZqDEOXAdT6F-movvLX4kCoNEzUcXX0"

    handle @authorized {
        reverse_proxy 127.0.0.1:11434 {
            header_up Host localhost:11434
        }
    }

    respond "Unauthorized" 401
}
```

Validate and restart:

```bash
sudo caddy validate --config /etc/caddy/Caddyfile
sudo systemctl restart caddy
```

Test **without** auth:

```bash
curl -vk https://alteryxllmdemo.duckdns.org/v1/models
```

Expected result:

- HTTP 401
- body: `Unauthorized`

Test **with** auth:

```bash
curl -s https://alteryxllmdemo.duckdns.org/v1/models \
  -H "Authorization: Bearer cTFYwz7fh3_iqZqDEOXAdT6F-movvLX4kCoNEzUcXX0" | jq .
```

Expected result: valid JSON model list.

Test a completion:

```bash
curl -s https://alteryxllmdemo.duckdns.org/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer cTFYwz7fh3_iqZqDEOXAdT6F-movvLX4kCoNEzUcXX0" \
  -d '{
    "model": "ollama/llama3:8b",
    "messages": [
      {"role": "user", "content": "Reply with exactly: auth working"}
    ]
  }' | jq .
```

Expected result: a normal OpenAI-style chat completion response.

---

## Step 8: Create the OpenAI Compatible connection in Alteryx One

Alteryx One supports **OpenAI Compatible** LLM connections using a **Base URL** and **API key**. Once created, the connection becomes available in the Prompt Tool and LLM Override Tool. ([Alteryx docs](https://help.alteryx.com/current/en/designer/tools/genai/create-llm-connections.html))

In Alteryx One:

1. Go to **Connections**.
2. Select **New Connection**.
3. Filter to **LLMs**.
4. Select **OpenAI Compatible**.
5. Enter:
   - **Connection Name**: something descriptive
   - **Base URL**: `https://alteryxllmdemo.duckdns.org/v1`
   - **API Key**: the value that matches your Caddy header rule

### Important note on header format

Your Caddy rule expects this exact header shape:

```http
Authorization: Bearer <secret>
```

Whether Alteryx prepends `Bearer ` automatically or sends the value exactly as entered should be validated in your tenant. In the reference build described here, the bearer-token rule was made to match the actual runtime behaviour observed when testing the connection end to end.

Test the connection from Alteryx One.

Expected result: **successful connection test**.

---

## Step 9: Validate in Designer Desktop

The Prompt Tool sends prompts to the selected LLM connection and returns the model response as output. It supports row-by-row prompt generation, optional incoming data columns, and output columns for both prompt and response. ([Alteryx Prompt Tool docs](https://help.alteryx.com/current/en/designer/tools/genai/prompt-tool.html))

In Designer Desktop:

1. Ensure your Alteryx One workspace link is active.
2. Drop a **Prompt Tool** on the canvas.
3. Select the hosted LLM connection.
4. Confirm the model list includes `ollama/llama3:8b`.
5. Send a small test prompt.

If the model appears and returns a response, the architecture is working end to end.

---

## Step 10: Validate the whole stack quickly

### Health checks

Check Ollama locally:

```bash
curl -s http://127.0.0.1:11434/v1/models | jq .
```

Check the public endpoint with auth:

```bash
curl -s https://alteryxllmdemo.duckdns.org/v1/models \
  -H "Authorization: Bearer <your-secret>" | jq .
```

Check both services:

```bash
sudo systemctl status ollama --no-pager
sudo systemctl status caddy --no-pager
```

### Common failure modes

#### `401 Unauthorized`
Cause:
- missing or incorrect bearer token

Fix:
- check the `Authorization` header value
- confirm the Caddy matcher matches what Alteryx actually sends

#### `403` from Ollama via proxy
Cause:
- upstream `Host` header not rewritten

Fix:
- ensure `header_up Host localhost:11434` is present in the Caddy config

#### `404 Not Found`
Cause:
- wrong path handling or wrong base URL

Fix:
- use `https://<hostname>/v1` as the Alteryx Base URL
- do not strip `/v1` unless your backend expects stripped paths

#### TLS / certificate issues
Cause:
- DNS not pointing to the VM
- port 80 or 443 not open

Fix:
- verify DNS resolution
- check firewall rules
- inspect `journalctl -u caddy`

---

## Practical constraint: synchronous runtime behaviour

In this reference pattern, Alteryx’s runtime interaction with the external endpoint is synchronous. In lab testing, short prompts worked, while longer CPU-bound requests could exceed a practical timeout threshold and return a `504` from the Alteryx side. Treat long-running CPU inference as a design risk and validate your own tenant behaviour early.

Practical mitigations include:

- using shorter prompts
- reducing context size
- constraining output length
- splitting the task into multiple faster calls
- using a smaller model
- pre-processing data heavily in Alteryx before prompt submission

This is especially important on CPU-only infrastructure.

---

## Security posture of this design

### What is exposed publicly

- `443/tcp` to Caddy
- optionally `80/tcp` for redirect / certificate operations

### What is not exposed publicly

- Ollama on `localhost:11434`
- the model runtime itself

### Operational access

- `22/tcp` for SSH, ideally restricted to known admin IPs

### Why this matters

This moves the security boundary to the reverse proxy rather than making the model server itself internet-facing.

---

## Production hardening recommendations

This guide is intentionally pragmatic. For a real production deployment, harden at least these areas:

1. **DNS**  
   Replace Duck DNS with enterprise-managed DNS.

2. **Ingress**  
   Put a load balancer or application gateway in front of the proxy.

3. **Authentication**  
   Replace a static bearer string with a managed secret and rotation process.

4. **Network isolation**  
   Use private networking, tighter security groups, and potentially private endpoints.

5. **Monitoring**  
   Add centralised logs, metrics, and service health monitoring.

6. **Model lifecycle**  
   Formalise versioning, promotion, rollback, and model benchmarking.

7. **Data controls**  
   Redact, minimise, and transform data before sending prompts.

---

## End-to-end request flow

```text
Admin
→ Alteryx One UI
→ Alteryx Connections Service
→ Connections Metadata Database

End User
→ Designer Desktop
→ Prompt Tool
→ AIMS / Alteryx AI Model Service
→ Alteryx Connections Service
→ Connections Metadata Database
→ AIMS
→ HTTPS `/v1` endpoint
→ Public hostname
→ Caddy reverse proxy
→ Bearer-token validation
→ Ollama API on `localhost:11434`
→ Local model `ollama/llama3:8b`
→ Ollama API
→ Caddy
→ AIMS
→ Designer Desktop
→ Prompt Tool output
```

---

## File structure suggestion for GitHub

If you want to publish this cleanly, use something like:

```text
.
├── README.md
├── docs/
│   ├── architecture.png
│   ├── troubleshooting.md
│   └── hardening-notes.md
└── examples/
    ├── Caddyfile
    └── curl-tests.md
```

---

## Suggested repo title

**byo-llm-alteryx-genai-reference**

---

## Suggested README introduction

> This repository documents a working reference implementation for connecting Alteryx Designer Desktop GenAI tools to a customer-hosted, OpenAI-compatible LLM endpoint using GCP, Ubuntu, Caddy, and Ollama. It is intended as a technical walkthrough and architecture reference rather than a production-ready product build.

---

## References

- Alteryx GenAI overview: https://help.alteryx.com/current/en/designer/tools/genai.html
- Alteryx Prompt Tool: https://help.alteryx.com/current/en/designer/tools/genai/prompt-tool.html
- Alteryx Create LLM Connections: https://help.alteryx.com/current/en/designer/tools/genai/create-llm-connections.html
- Ollama OpenAI compatibility: https://docs.ollama.com/openai
- Ollama FAQ: https://docs.ollama.com/faq
- Caddy reverse proxy docs: https://caddyserver.com/docs/caddyfile/directives/reverse_proxy

