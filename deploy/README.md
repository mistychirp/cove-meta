# Deploying Cove

The stack is Caddy (TLS) + the Go server + Postgres + LiveKit. The client is
built into static files and served by the same Caddy, so the client and the API
live on one domain and users never have to type a server address.

## 1. Requirements

- Docker + Docker Compose
- A domain whose A record points at this server's **public** IP
- Router port forwards (see below)

## 2. Router ports

This is the one step that cannot be automated. Without it you get neither a
certificate nor voice.

| Port | Protocol | Why |
|------|----------|-----|
| 80 | TCP | Let's Encrypt (domain validation) + redirect to HTTPS |
| 443 | TCP | the site itself, API, WebSocket gateway, LiveKit signaling |
| 3478 | UDP | embedded TURN (friends behind strict NAT) |
| 50000–50060 | UDP | voice and screen share media |

**Why UDP is mandatory:** only signaling passes through Caddy. The audio and
video themselves fly straight to this server over UDP, bypassing the reverse
proxy. Forgetting UDP means "it connects and stays silent".

## 3. Installation

```sh
# side by side — compose builds the neighbouring directories
git clone https://github.com/mistychirp/cove-server.git
git clone https://github.com/mistychirp/cove-web.git
git clone <this repository> cove-meta      # provides deploy/

cd cove-meta/deploy
cp .env.example .env
```

Generate the secrets and put them into `.env`:

```sh
echo "COVE_DB_PASSWORD=$(openssl rand -base64 24)"
echo "COVE_LIVEKIT_API_KEY=$(openssl rand -hex 8)"
echo "COVE_LIVEKIT_API_SECRET=$(openssl rand -base64 36)"
```

The domain has to be set in one more place — `livekit.yaml`, field `turn.domain`
(LiveKit does not read it from the environment).

Start:

```sh
docker compose up -d --build
docker compose logs -f caddy      # watch the certificate being issued
```

Check:

```sh
curl https://<domain>/health           # {"status":"ok"}
curl https://<domain>/api/v1/instance  # voice_enabled: true
```

## 4. First login

Open `https://<domain>` and register — **the first account automatically becomes
the instance owner** and needs no registration code.

After that registration is closed: you hand out codes to friends via the
"Registration codes" button in the sidebar.

## 5. Updating

```sh
cd cove-server && git pull && cd ../cove-web && git pull
cd ../cove-meta/deploy && docker compose up -d --build
```

The server applies database migrations on boot — nothing to do by hand.

## 6. Data and backups

Everything lives in Docker volumes: `cove-db` (database) and `cove-media`
(attachments).

```sh
# database dump
docker compose exec -T db pg_dump -U cove cove > cove-$(date +%F).sql
# attachments
docker run --rm -v cove_cove-media:/m -v "$PWD":/out alpine \
  tar czf /out/media-$(date +%F).tar.gz -C /m .
```

## 7. Troubleshooting

- **No certificate is issued** — check that the domain resolves from the outside
  to this server's public IP and that port 80 is forwarded. From inside the
  network the domain may resolve to a local address (hairpin NAT); that is
  normal, verify with an external resolver.
- **Voice connects but stays silent** — almost always missing UDP forwards
  (3478 and 50000–50060).
- **Microphone/screen unavailable** — browsers only grant them over HTTPS. Make
  sure you opened `https://`, not `http://`.
- **Don't want voice** — drop the `COVE_LIVEKIT_*` entries from `.env`: the
  instance becomes text-only instead of breaking.
