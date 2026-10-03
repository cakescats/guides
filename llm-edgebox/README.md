# LLM EdgeBox, do it yourself: private AI with no cloud

[Русская версия](README.ru.md)

What you get: a **local language model** on your computer or server with a familiar **browser chat**, **search over your own documents** and an **OpenAI-compatible API**. Prompts and documents never leave your machine.

Components (all free and open source):
- **[Ollama](https://ollama.com)** — runs the models;
- **[Open WebUI](https://openwebui.com)** — web chat, user accounts, document knowledge base.

> Want a ready server tuned to your documents and tasks, with support? That's [llm.cakescats.com](https://llm.cakescats.com/). Buying it supports our non-profit project.

---

## 1. Hardware

What matters most is how much memory the model can use: GPU memory or unified memory (Apple silicon).

| Memory | Models | Good for |
|---|---|---|
| 8–16 GB | 7–14B parameters | Personal assistant, drafts, translation |
| 24–32 GB | up to ~32B | Working with documents, a small team |
| 64–128 GB (e.g. Mac Studio or NVIDIA DGX Spark) | MoE models up to ~120B | A strong assistant for a team of up to 20 |
| 16 GB GPU + 30 GB RAM + fast NVMe | ~125B MoE, slow (~10 tokens/s) | Our fork [QwFNfer Secure Multilang](https://github.com/cakescats/QwFNfer-Secure-Multilang) |

OS: Linux (Ubuntu 24.04+), macOS on Apple silicon, or Windows. Commands below are for Ubuntu; on macOS and Windows install Ollama with the regular installer from [ollama.com/download](https://ollama.com/download) and run Open WebUI with Docker Desktop.

## 2. Ollama and a model

```bash
curl -fsSL https://ollama.com/install.sh | sh
ollama --version
```

Pull a model — see [ollama.com/library](https://ollama.com/library) and choose by memory size from the table above, for example:

```bash
ollama pull qwen3        # the Qwen family, strong multilingual support
ollama run qwen3 "Hi! Briefly: what can you do?"
```

## 3. Open WebUI (browser chat)

Install Docker:

```bash
curl -fsSL https://get.docker.com | sh
```

Run Open WebUI:

```bash
sudo docker run -d --network=host \
  -v open-webui:/app/backend/data \
  -e OLLAMA_BASE_URL=http://127.0.0.1:11434 \
  --name open-webui --restart always \
  ghcr.io/open-webui/open-webui:main
```

Open **http://COMPUTER_IP:8080**. The first user you create becomes the administrator.

Admin settings (**Admin Panel → Settings**):
- **General → Default User Role**: *pending* — new users wait for your approval;
- **General → Enable New Sign Ups**: turn off once your team has registered;
- **Models**: choose the default model.

## 4. Fully offline

Once the models are downloaded, the server doesn't need the internet.

1. Restart Open WebUI in offline mode:

```bash
sudo docker rm -f open-webui
sudo docker run -d --network=host \
  -v open-webui:/app/backend/data \
  -e OLLAMA_BASE_URL=http://127.0.0.1:11434 \
  -e OFFLINE_MODE=true -e HF_HUB_OFFLINE=1 \
  --name open-webui --restart always \
  ghcr.io/open-webui/open-webui:main
```

2. Block the server's internet access on your router or firewall, allowing only your local network. Example with `ufw` on the server itself, if your network is `192.168.1.0/24`:

```bash
sudo ufw default deny incoming
sudo ufw default deny outgoing
sudo ufw allow from 192.168.1.0/24
sudo ufw allow out to 192.168.1.0/24
sudo ufw enable
```

To update models later, temporarily allow outgoing traffic (`sudo ufw default allow outgoing`), run `ollama pull …`, then block it again.

## 5. Working with your documents (RAG)

1. **Workspace → Knowledge → +** — create a collection, e.g. “Policies”.
2. Upload files: PDF, DOCX, TXT, Markdown.
3. In a chat type `#` and pick the collection — the model answers from your documents and shows its sources.

Your tone and terminology: **Workspace → Models → +** — create a custom model on top of a base one, set a system prompt (“Answer in a business tone, use our company terms…”) and attach the knowledge collection.

## 6. API for apps and AI agents

Ollama exposes an OpenAI-compatible API at `http://SERVER_IP:11434/v1`.

By default Ollama listens on `127.0.0.1` only. To expose the API on your LAN:

```bash
sudo systemctl edit ollama
# add:
# [Service]
# Environment="OLLAMA_HOST=0.0.0.0"
sudo systemctl restart ollama
```

Test from another computer:

```bash
curl http://SERVER_IP:11434/v1/chat/completions \
  -H "Content-Type: application/json" \
  -d '{"model":"qwen3","messages":[{"role":"user","content":"Hello"}]}'
```

> The Ollama API has no authentication — expose it only on a trusted network. For access from outside, use Open WebUI, which has accounts and API keys (**Settings → Account → API keys**).

## 7. HTTPS on your LAN (optional)

To avoid browser warnings, put [Caddy](https://caddyserver.com) with an internal certificate in front of Open WebUI:

```bash
sudo apt install -y caddy
echo 'https://ai.local {
  tls internal
  reverse_proxy 127.0.0.1:8080
}' | sudo tee /etc/caddy/Caddyfile
sudo systemctl restart caddy
```

Add `ai.local` to your router's DNS (or hosts files) and install Caddy's root certificate on your team's computers once.

## Troubleshooting

| Problem | Fix |
|---|---|
| Open WebUI doesn't see models | `curl http://127.0.0.1:11434/api/tags` should list them; check `OLLAMA_BASE_URL` |
| Answers are very slow | The model is too big for your memory — use a smaller or more quantized one |
| Out of GPU memory | `ollama ps` shows how much of the model runs on the GPU; close other programs |

Questions — Telegram [@cakescats](https://t.me/cakescats) or [mail@cakescats.com](mailto:mail@cakescats.com).
