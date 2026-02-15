# OSPW Matrix — Private Chat Infrastructure

Self-hosted, isolated Matrix homeserver with Element Web client and TURN server for voice/video calls.
Designed as a private Discord replacement with full data sovereignty.

## Comment

All Users are created and distributed by you. User side registration methods like authentik can (and will) be added in the future.
Users cant connect their Email right now due to no SMTP service having been configured. (for sending verification emails) ill fix this together with authentik.
If you want users registered on your server to be able to access any connectivity outside of your network, such as messaging users or joining public rooms like your used to with discord.
Then that means that all content send to your users from outside connections and servers WILL be cached on your server, meaning your responsible for that data without having the ability to aprove, limit or view that data.
Federation settings will have to be changed if you want that outside connectivity. In that case i dont recommend using this repo as a template for deployment.

## Benefits / Key Design Decisions

- **No federation**: isolated mode, no external servers, no external content cached
- **No public registration**: admin creates all accounts via CLI (users get account credentials and change their password)
- **E2EE**: end to end encryption available per room (full data privacy requires end2end to be turned on (should be automatically) in the settings otherwise server admin can read / change database entries)
- **Voice/video calls**: by selfhosted Coturn TURN server
- **Caddy reverse proxy**: no docker ports exposed to host

## Architecture

| Service | Domain | Purpose |
|---------|--------|---------|
| Synapse | matrix.yourdomain.com | Matrix homeserver — handles messaging, authentication, media storage |
| Element Web | chat.yourdomain.com | Browser-based Matrix client — this is what users interact with |
| Coturn | matrix.yourdomain.com:3478 | TURN/STUN relay — required for voice and video calls to work |
| PostgreSQL | Internal only | Database backend for Synapse — not exposed to the internet |

## Prerequisites

- A VPS or server with at least 4GB RAM (8GB recommended)
- A domain with DNS access (you need to create A records for your subdomains)
- Docker and Docker Compose installed
- Caddy (or another reverse proxy) running with access to the `frontend` Docker network
- UFW or another firewall configured

## Setup


### Step 1: Clone and configure environment

git clone https://github.com/osk4r8088/ospw-matrix.git
cd ospw-matrix
cp .env.example .env

Generate 5 secrets (one per line — use them for the values below):

python3 -c "import secrets; [print(secrets.token_hex(32)) for _ in range(5)]"

Open .env and replace every placeholder with one of the generated secrets:

nano .env

The file looks like this — replace each value:

SYNAPSE_DB_PASSWORD=<secret-1>
SYNAPSE_MACAROON_SECRET=<secret-2>
SYNAPSE_FORM_SECRET=<secret-3>
SYNAPSE_REGISTRATION_SECRET=<secret-4>
TURN_SHARED_SECRET=<secret-5>


### Step 2: Configure Synapse
nano synapse/homeserver.yaml

Important: Synapse does NOT read environment variables. You must replace the placeholders in this file with the actual secret strings:

Placeholder	Replace with
CHANGE_ME_DB_PASSWORD	Same value as SYNAPSE_DB_PASSWORD from .env
CHANGE_ME_MACAROON_SECRET	Same value as SYNAPSE_MACAROON_SECRET from .env
CHANGE_ME_FORM_SECRET	Same value as SYNAPSE_FORM_SECRET from .env
CHANGE_ME_REGISTRATION_SECRET	Same value as SYNAPSE_REGISTRATION_SECRET from .env
CHANGE_ME_TURN_SECRET	Same value as TURN_SHARED_SECRET from .env
Also update server_name and public_baseurl to your domain (default is matrix.ospw.de).


### Step 3: Configure Coturn
nano coturn/turnserver.conf

Replace these two values:

Placeholder	Replace with
YOUR_PUBLIC_IP	Your servers public IPv4 address (find it with curl -4 ifconfig.me)
CHANGE_ME_TURN_SECRET	Same value as TURN_SHARED_SECRET from .env (must match homeserver.yaml)
Also update realm to your domain if different from matrix.ospw.de.


### Step 4: Configure Element Web
nano element/config.json

Update base_url and server_name to match your domain.

Step 5: Generate the Synapse signing key
This key is used to sign all events on your server. Generate it once and back it up — if you lose it, your server identity is gone:

docker run --rm --entrypoint python matrixdotorg/synapse:latest -c \
  "from signedjson.key import generate_signing_key, write_signing_keys; import sys; \
  write_signing_keys(sys.stdout, [generate_signing_key('a_MNgy')])" \
  > synapse/matrix.ospw.de.signing.key

Rename the file if your domain is different from matrix.ospw.de and update the signing_key_path in homeserver.yaml accordingly.


### Step 6: Create the log config
cat > synapse/matrix.ospw.de.log.config << 'EOF'
version: 1
formatters:
  precise:
    format: '%(asctime)s - %(name)s - %(lineno)d - %(levelname)s - %(request)s - %(message)s'
handlers:
  console:
    class: logging.StreamHandler
    formatter: precise
loggers:
  synapse.storage.SQL:
    level: WARNING
root:
  level: WARNING
  handlers: [console]
disable_existing_loggers: false
EOF


### Step 7: Fix permissions
Synapse runs as UID 991 inside the container. The data directory needs to be owned by this user:

``sudo chown -R 991:991 synapse/``

### Step 8: Set up DNS
Create A records at your domain registrar pointing to your servers IP:

Subdomain	Type	Value
matrix.yourdomain.com	A	your-server-ip
chat.yourdomain.com	A	your-server-ip


### Step 9: Configure reverse proxy (Caddy)
Add these entries to your Caddyfile. Caddy will automatically handle HTTPS/TLS certificates:

``matrix.yourdomain.com {
    reverse_proxy synapse:8008
    encode gzip
    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
        X-Frame-Options "DENY"
        X-Content-Type-Options "nosniff"
        Referrer-Policy "strict-origin-when-cross-origin"
    }
}``

``chat.yourdomain.com {
    reverse_proxy element-web:80
    encode gzip
    header {
        Strict-Transport-Security "max-age=31536000; includeSubDomains; preload"
        X-Frame-Options "DENY"
        X-Content-Type-Options "nosniff"
        Referrer-Policy "strict-origin-when-cross-origin"
    }
}``

Reload Caddy after editing:

docker exec caddy caddy reload --config /etc/caddy/Caddyfile


### Step 10: Open firewall ports
Coturn needs these ports for voice/video call relay:

sudo ufw allow 3478/tcp    # TURN signaling
sudo ufw allow 3478/udp    # TURN signaling
sudo ufw allow 49152:65535/udp  # Media relay range


## Final Step: Start everything
docker compose up -d

Wait about 15 seconds for Synapse to initialize, then verify:

docker compose ps                    # All containers should be healthy
docker compose logs synapse --tail 5 # Should show "STARTING SERVER"

## User Management
Create a new user
Since public registration is disabled, you create all accounts as admin:

docker exec -it synapse register_new_matrix_user \
  -c /data/homeserver.yaml http://localhost:8008

It will ask you:

Username: this becomes their Matrix ID: @username:yourdomain.com

Password: give this to the user, they should change it after first login

Admin? say yes for yourself, no for regular users


## Platform / How to connect
Browser: 	Login to chat.yourdomain.com
Desktop:	Download Element Desktop from element.io/download, set homeserver to matrix.yourdomain.com
Android:	Install Element from Play Store, set homeserver to matrix.yourdomain.com
iOS:	Install Element from App Store, set homeserver to matrix.yourdomain.com

## What users can do
- Send text, images, files, videos, voice messages
- Make voice and video calls (1:1 and group)
- Create rooms and invite other users on the server
- Enable end-to-end encryption per room
- Change their display name, avatar, and password
- Set notification preferences
- What users cannot do
- Register new accounts themselves
- Communicate with users on other Matrix servers (federation is off)
- Join public rooms on other servers
- Access admin tools
- Reset a users password


## If a user forgets their password:

docker exec -it synapse python -m synapse._scripts.hash_password

Enter the new password when prompted. Then update the database:

docker exec -it synapse-db psql -U synapse -d synapse -c \
  "UPDATE users SET password_hash='<hash-from-above>' WHERE name='@username:yourdomain.com';"


## Operations
docker compose logs -f synapse     # Synapse logs (live)

docker compose logs -f coturn      # Coturn logs (live)

docker compose restart synapse     # Restart Synapse after config changes

docker compose down                # Stop all services

docker compose up -d               # Start all services

docker compose pull                # Pull latest images (for updates)


## Security
Signing key (*.signing.key) is gitignored: back it up separately, losing it means losing your server identity

E2EE: enable per room for true end-to-end encryption where even the server admin cannot read messages

Coturn blocks relay to all private IP ranges to prevent SSRF attacks

No secrets in this repo: all values are placeholders

No Docker ports exposed to host: all traffic routes through Caddy

Federation disabled: no external attack surface from other Matrix servers


## Future Improvements
- Authentik OIDC SSO integration
 
- SMTP for email verification
 
- Automated backups (Restic to offsite storage)

- AI bot integration (Ollama + matrix-nio)
 
- Synapse Admin UI
 
- Custom Element Web branding
