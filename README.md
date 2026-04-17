# cn-pki

Private CA + certificate management + uptime monitoring stack for the home LAN.

## Services

| Service | Port | Description |
|---|---|---|
| `step-ca` | `127.0.0.1:9000` | Private ACME CA — signs certificates with your existing root CA |
| `certwarden` | `127.0.0.1:4050` | Certificate store, renewal engine, and web UI |
| `uptime-kuma` | `127.0.0.1:3001` | Endpoint availability + SSL expiry monitoring |

All ports are bound to `127.0.0.1` only — accessible on the Pi itself or via SSH tunnel / tailnet.

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
- `CA_DNS_NAMES` — comma-separated hostnames/IPs step-ca will be reachable at on the LAN (e.g. `step-ca,192.168.1.42`)

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

Open `http://127.0.0.1:4050`.

1. **Add ACME server** — URL: `https://127.0.0.1:9000/acme/acme/directory`
   - Upload `pki/root_ca.crt` as the CA bundle (so Cert Warden trusts step-ca's HTTPS)
2. **Order a certificate** — e.g. `passwords.lan`
   - Challenge type: `tls-alpn-01` (step-ca connects to the service on port 443 to verify)
3. **Create an API key** — used by `certwarden-client` in cn-vaultwarden

### 5. Configure Uptime Kuma

Open `http://127.0.0.1:3001`.

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

- **Internal ACME DNS**: tls-alpn-01 and http-01 challenges require step-ca to reach the service being certified. This works when step-ca and the service are on the same LAN. If your `.lan` DNS doesn't resolve from the Pi, challenges will fail.
- **Cert Warden**: primarily maintained by one person. Review [the project](https://github.com/gregtwallace/certwarden) before using in production.
