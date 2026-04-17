# cn-pki

Private CA + certificate management + uptime monitoring stack for the home LAN.

## Services

| Service | URL | Description |
|---|---|---|
| `traefik` | `:80` / `:443` | Reverse proxy — terminates TLS with a cert from step-ca |
| `step-ca` | `:9000` | Private ACME CA — signs certificates with your existing root CA |
| `certwarden` | `https://<PKI_DOMAIN>/certwarden` | Certificate store, renewal engine, and web UI |
| `uptime-kuma` | `https://<PKI_DOMAIN>/` | Endpoint availability + SSL expiry monitoring |

Traefik obtains a TLS certificate from step-ca via ACME (http-01 challenge) on first start, and redirects HTTP to HTTPS. Step-ca keeps its own port (ACME clients connect to it directly).

## Firewall / port requirements

| Port | Protocol | Purpose | Who connects |
|------|----------|---------|--------------|
| `80` | TCP | Traefik HTTP — redirects to HTTPS; also serves ACME http-01 challenges from step-ca | Browsers (redirect), step-ca (challenge verification) |
| `443` | TCP | Traefik HTTPS — serves certwarden and uptime-kuma with TLS | You (browser) |
| `9000` | TCP | step-ca ACME endpoint (HTTPS) | LAN services requesting certificates (e.g. cn-vaultwarden's certwarden-client) |

All three ports must be reachable from the LAN. No other inbound ports are required — certwarden and uptime-kuma are only reachable through Traefik.

## Bring-up sequence

### 1. Prepare CA files

Place the following in `pki/` (this directory is gitignored):

```
pki/
  root_ca.crt     ← your existing root CA certificate (PEM)
  root_ca.key     ← your existing root CA private key (PEM)
  password.txt    ← provisioner password for step-ca (plain text, one line; not the root CA key passphrase)
```

### 2. Configure `.env`

```sh
cp .env.example .env
```

Edit `.env`:
- `CA_NAME` — display name for the CA (shown in cert issuer field)
- `CA_DNS_NAMES` — comma-separated hostnames/IPs step-ca will be reachable at. **Must include `step-ca`** (the Docker service name) so Traefik can connect to the ACME endpoint over TLS (e.g. `step-ca,pki.lan,192.168.1.42`)
- `PKI_DOMAIN` — hostname Traefik will request a TLS cert for (e.g. `pki.lan`). Must resolve to this machine's LAN IP — both from browsers and from inside Docker (step-ca uses it for the ACME http-01 challenge)
- `ADMIN_EMAIL` — email address for ACME registration with step-ca

### 3. Start

```sh
docker compose up -d
```

step-ca initialises on first run and creates its own TLS cert signed by your root CA. On subsequent starts it skips init.

Verify step-ca is healthy:
```sh
curl -k https://127.0.0.1:9000/health
# → {"status":"ok"}
```

### 4. Configure Cert Warden

Open `https://<PKI_DOMAIN>/certwarden/app`. Default credentials: `admin` / `password` (you'll be prompted to change the password on first login).

1. **Add ACME server** — URL: `https://step-ca:9000/acme/acme/directory`
   - The root CA is automatically injected into certwarden's system trust store on startup (via the custom entrypoint), so step-ca's HTTPS is trusted out of the box
2. **Order a certificate** — e.g. `passwords.lan`
   - Challenge type: `tls-alpn-01` (step-ca connects to the service on port 443 to verify)
3. **Create an API key** — used by `certwarden-client` in cn-vaultwarden

### 5. Configure Uptime Kuma

Open `https://<PKI_DOMAIN>/`.

- Add monitors for each LAN service (`https://passwords.lan`, etc.)
- Configure SMTP email notifications (Settings → Notifications)

### 6. Device trust (one-time per device)

Install `pki/root_ca.crt` as a trusted root CA on each device. After this, all `.lan`
services signed by this CA show a green padlock automatically.

| Platform | How |
|---|---|
| macOS | Keychain Access → drag in cert → set "Always Trust" |
| iOS | Settings → General → VPN & Device Management → install profile → Certificate Trust Settings → enable |
| Android | Settings → Security → Install from storage |
| Linux | `sudo cp root_ca.crt /usr/local/share/ca-certificates/ && sudo update-ca-certificates` |

## Caveats

- **DNS resolution**: `PKI_DOMAIN` must resolve to this machine's LAN IP both from browsers and from inside Docker containers. step-ca uses it for the ACME http-01 challenge callback. If your `.lan` DNS doesn't resolve from inside Docker, the challenge will fail and Traefik won't get a cert. As a workaround, you can use a nip.io domain (e.g. `192-168-1-42.nip.io`).
- **ACME retry**: Traefik does not automatically retry failed ACME requests. If step-ca is unreachable when Traefik starts, run `docker compose restart traefik` after step-ca is healthy.
- **Internal ACME DNS**: tls-alpn-01 and http-01 challenges require step-ca to reach the service being certified. This works when step-ca and the service are on the same LAN.
- **Cert Warden**: primarily maintained by one person. Review [the project](https://github.com/gregtwallace/certwarden) before using in production.
