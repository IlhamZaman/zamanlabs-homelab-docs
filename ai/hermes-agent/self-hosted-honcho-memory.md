<!--
Organized from: STEP 16 - Self-Hosted Honcho Memory for Hermes (Canonical).txt
Source wording and technical details were preserved as closely as possible.
-->

> [!CAUTION]
> This is a historical implementation record. Verify versions, addresses, paths, and commands against the live environment before applying changes.

# STEP 16 - Self-Hosted Honcho Memory Integration (Canonical)

Goal Configure Hermes to use a locally hosted Honcho memory service on
the same Ubuntu Server VM.

Final Architecture

```text
Ubuntu Server VM ├── Hermes Agent ├── Hermes Discord Gateway ├── Honcho
API ├── Honcho Deriver ├── PostgreSQL (pgvector) └── Redis
```

Hermes connects to: http://127.0.0.1:8000

No Honcho Cloud account, Cloudflare Tunnel, reverse proxy, router port
forwarding, or Honcho API key is required.

==================================================================== 1.
## Install Docker (skip if already installed)

sudo apt update sudo apt upgrade -y sudo apt install -y docker.io
docker-compose-v2 git curl ca-certificates sudo systemctl enable –now
docker sudo usermod -aG docker “$USER”

Log out and back in.

Verify:

```text
docker –version docker compose version docker run –rm hello-world
```

==================================================================== 2.
## Download Honcho

```text
cd ~ git clone https://github.com/plastic-labs/honcho.git cd ~/honcho
```

```text
cp docker-compose.yml.example docker-compose.yml cp .env.template .env
```

==================================================================== 3.
## Configure Honcho

Edit:

```text
nano ~/honcho/.env
```

Configure:

```text
AUTH_USE_AUTH=false SENTRY_ENABLED=false
LLM_OPENAI_API_KEY=<OPENAI_API_KEY>
```

Protect the file:

```text
chmod 600 ~/honcho/.env
```

==================================================================== 4.
## Start Honcho

```text
cd ~/honcho docker compose up -d –build
```

Verify:

```text
docker compose ps
```

Health check:

```text
curl http://127.0.0.1:8000/health
```

Expected: HTTP 200

==================================================================== 5.
## Run Hermes Memory Setup

hermes memory setup

Select:

Deployment………………….local Base URL……………………http://127.0.0.1:8000 Local
JWT…………………..leave blank User peer…………………..izaman AI peer…………………….hermes
Workspace…………………..hermes Gateway mapping……………..Just me Observation
mode…………….directional Write frequency……………..async Recall
mode…………………hybrid Context tokens………………uncapped Dialectic cadence……………2
Reasoning level…………….low Session strategy…………….per-session

==================================================================== 6.
## Restart Hermes

hermes gateway restart hermes gateway status hermes memory status

==================================================================== 7.
## Verify Honcho

```text
cd ~/honcho
```

docker compose ps docker compose logs –tail=100 docker compose logs
deriver –tail=100

Check API:

```text
curl http://127.0.0.1:8000/health
```

Expected: 200 OK

==================================================================== 8.
## Functional Test

In Discord:

Remember that my primary Proxmox server is a Wiwynn Lyra SV315.

Start a new conversation later:

What is my primary Proxmox server?

Successful recall confirms Hermes and Honcho are working together.

==================================================================== 9.
## Troubleshooting

Check all containers:

```text
docker compose ps
```

View logs:

```text
docker compose logs –tail=200 docker compose logs deriver –tail=200
```

Restart stack:

```text
docker compose restart
```

Restart Hermes:

hermes gateway restart

==================================================================== 10.
## Updating Honcho

```text
cd ~/honcho git pull docker compose up -d –build
```

==================================================================== 11.
## Important Notes

• Honcho stores memory locally on this VM. • No Honcho Cloud account is
used. • No Honcho API key is required. • The only external dependency is
the OpenAI API key used by Honcho for memory derivation. • Keep Honcho
bound to localhost (127.0.0.1) whenever possible. • Do not store
passwords, SSH keys, API keys, or other secrets in conversations
intended for long-term memory.
