# OSPW Matrix — Private Chat Infrastructure

Self-hosted, isolated Matrix homeserver with Element Web client and TURN server for voice/video calls. 
Designed as a private Discord replacement with full data sovereignty.

## Comment

All Users are created and distributed by you, although user side registration methods like authentik can (and will) be added in the future.
Users cant connect their email right now due to no SMTP service having been configured. (for sending verification emails, ill fix this together with authentik.)

If you want users registered on your server to be able to access any connectivity outside of your network, such as messaging users or joining public rooms federation settings will have to be changed.
This means that all content send to your users from outside connections and servers WILL be cached on your server, meaning your responsible for that data without having the ability to aprove, limit or view that data.

## Benefits / Key Design Decisions

- **No federation**: isolated mode, no external servers, no external content cached
- **No public registration**: admin creates all accounts via CLI (users get account credentials and change their password)
- **E2EE**: end to end encryption available per room (full data privacy requires end2end to be turned on (should be automatically) in the settings otherwise server admin can read / change database entries)
- **Voice/video calls**: by selfhosted Coturn TURN server
- **Caddy reverse proxy**: no docker ports exposed to host

## Architecture

| Service | Domain | Purpose |
|---------|--------|---------|
| Synapse | matrix.ospw.de | Matrix homeserver |
| Element Web | chat.ospw.de | Browser-based Matrix client |
| Coturn | matrix.ospw.de:3478 | TURN/STUN relay for voice and video calls |
| PostgreSQL | Internal only | Database for Synapse |

## Setup
1. Copy `.env.example` to `.env` and generate secrets
2. Replace all `CHANGE_ME` placeholders in `synapse/homeserver.yaml` and `coturn/turnserver.conf`
3. Generate signing key: docker run --rm --entrypoint python matrixdotorg/synapse:latest -c
"from signedjson.key import generate_signing_key, write_signing_keys; import sys;
write_signing_keys(sys.stdout, [generate_signing_key('a_MNgy')])" \

synapse/matrix.ospw.de.signing.key

4. Fix permissions: `sudo chown -R 991:991 synapse/`
5. Start: `docker compose up -d`
6. Add Caddy reverse proxy entries for `matrix.ospw.de` and `chat.ospw.de` (your domains go here & end obv need domain host subdomain entry (A + host ip))
7. Open firewall: `sudo ufw allow 3478/tcp && sudo ufw allow 3478/udp && sudo ufw allow 49152:65535/udp`

## User Management

```bash
docker exec -it synapse register_new_matrix_user -c /data/homeserver.yaml http://localhost:8008

Security
Signing key (*.signing.key) is gitignored — back it up separately
Enable E2EE per room for true end-to-end encryption
Coturn blocks relay to private IP ranges (SSRF protection)
No secrets in this repo — all placeholders
REOF
