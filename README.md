
# Self-Hosted Private AI & Metasearch Stack (`Ollama + Open WebUI + SearXNG`)

A reproducible, privacy-first local AI deployment running on Fedora/Nobara Linux with hardware GPU acceleration, rootless Podman container orchestration, SELinux volume labeling, and self-hosted metasearch bound strictly to `127.0.0.1`.

## Business & Security Value
- **Zero Data Leakage:** All LLM inference, document/RAG indexing, and chat histories stay 100% on local hardware—ideal for businesses handling sensitive internal logs, configs, or client documentation.
- **Anonymous Real-Time Web Search:** Integrates a local `SearXNG` metasearch instance (`127.0.0.1:8080`) so the local model can retrieve current web documentation without external search tracking or third-party API keys.
- **Hardened Loopback Binding:** Services bind exclusively to `localhost` (`127.0.0.1`) rather than `0.0.0.0`, preventing unauthorized LAN or WAN exposure while preserving proper SELinux `:Z` volume contexts.

---

## Architecture Overview

| Component | Runtime / Binding | Purpose |
| :--- | :--- | :--- |
| **Ollama (`qwen2.5:14b`)** | Native `systemd` service (`127.0.0.1:11434`) | Local LLM inference accelerated via AMD ROCm / NVIDIA CUDA |
| **SearXNG** | Rootless Podman (`127.0.0.1:8080`) | Self-hosted privacy-respecting metasearch engine with JSON API output |
| **Open WebUI** | Rootless Podman (`127.0.0.1:3000`) | Browser-based chat interface, document RAG, and tool orchestration |

---

## Deployment Runbook

### 1. Install Ollama & Pull Local Model
```bash
curl -fsSL https://ollama.com | sh
ollama run qwen2.5:14b
```

Verify full GPU VRAM offloading while the model is active:
```bash
ollama ps
```

### 2. Create Persistent Directories & Configure SearXNG
```bash
mkdir -p ~/ai-stack/searxng ~/ai-stack/open-webui

cat << 'EOF' > ~/ai-stack/searxng/settings.yml
use_default_settings: true
server:
  secret_key: "REPLACE_WITH_RANDOM_SECRET_KEY"
  limiter: false
search:
  formats:
    - html
    - json
EOF
```

### 3. Launch Rootless Containers (`SearXNG` & `Open WebUI`)
Start **SearXNG** bound strictly to `127.0.0.1:8080`:
```bash
podman run -d --name searxng --restart=always \
  -p 127.0.0.1:8080:8080 \
  -v ~/ai-stack/searxng:/etc/searxng:Z \
  docker.io/searxng/searxng:latest
```

Start **Open WebUI** wired to local Ollama and SearXNG:
```bash
podman run -d --name open-webui --restart=always \
  --network=host \
  -e PORT=3000 \
  -e OLLAMA_BASE_URL=http://127.0.0.1:11434 \
  -e ENABLE_RAG_WEB_SEARCH=True \
  -e RAG_WEB_SEARCH_ENGINE=searxng \
  -e SEARXNG_QUERY_URL="http://127.0.0<query>" \
  -v ~/ai-stack/open-webui:/app/backend/data:Z \
  ghcr.io/open-webui/open-webui:main
```

### 4. Verification & Post-Install Hardening
1. Confirm both containers are healthy:
   ```bash
   podman ps
   ```
2. Open `http://127.0.0.1:3000` in Firefox, create the local offline administrator account, and verify **Admin Panel -> Settings -> Web Search** points to `http://127.0.0<query>`.
3. Toggle **Web Search** in the chat bar and confirm live citations return from the local `SearXNG` container.
