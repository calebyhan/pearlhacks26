# Deployment

## Infrastructure Overview

| Component | Where | What |
|---|---|---|
| Python server | Vultr Ubuntu VPS | WebSocket server, Gemini relay, static file host |
| coturn | Same VPS, separate process | STUN/TURN for WebRTC NAT traversal |
| SSL | Let's Encrypt (certbot) | TLS for WSS and HTTPS |
| Domain | visual911.mooo.com (freedns) | Required for SSL + coturn TLS |

Minimum VPS: Vultr $6/month (1 vCPU, 1GB RAM). Sufficient for a hackathon with 1 active call at a time.

---

## Step 1: Provision VPS

On Vultr, deploy Ubuntu 24.04 LTS. Note the public IP address.

```bash
# SSH in
ssh root@YOUR_VPS_IP
```

---

## Step 2: Point Domain to VPS

In your DNS registrar, create an A record:

```
visual911.mooo.com  →  YOUR_VPS_IP
```

Wait for DNS propagation (can be instant to 30 minutes with TTL=60).

Verify:
```bash
dig visual911.mooo.com +short
# Should return YOUR_VPS_IP
```

---

## Step 3: Install Dependencies

```bash
apt update && apt upgrade -y
apt install -y python3 python3-pip coturn certbot ufw

cd /opt/visual911
pip3 install -r server/requirements.txt --break-system-packages
```

---

## Step 4: Firewall

```bash
ufw allow 22      # SSH
ufw allow 80      # HTTP (certbot challenge)
ufw allow 443     # HTTPS/WSS
ufw allow 3478    # STUN/TURN UDP+TCP
ufw allow 5349    # TURN TLS
ufw allow 49152:65535/udp   # TURN relay ports
ufw enable
```

---

## Step 5: SSL Certificate

```bash
certbot certonly --standalone -d visual911.mooo.com

# Certificates will be at:
# /etc/letsencrypt/live/visual911.mooo.com/fullchain.pem
# /etc/letsencrypt/live/visual911.mooo.com/privkey.pem

# Auto-renewal (certbot installs a systemd timer automatically)
# Verify: systemctl status certbot.timer
```

---

## Step 6: coturn Configuration

Edit `/etc/turnserver.conf`:

```conf
listening-port=3478
tls-listening-port=5349

fingerprint
use-auth-secret
static-auth-secret=GENERATE_A_RANDOM_SECRET_HERE

realm=visual911.mooo.com

cert=/etc/letsencrypt/live/visual911.mooo.com/fullchain.pem
pkey=/etc/letsencrypt/live/visual911.mooo.com/privkey.pem

total-quota=100
no-multicast-peers
log-file=/var/log/turnserver.log
```

Generate a random secret:
```bash
openssl rand -hex 32
# Copy the output into static-auth-secret=
```

Start coturn:
```bash
systemctl enable coturn
systemctl start coturn
systemctl status coturn  # Should show "active (running)"
```

Test STUN:
```bash
# Install turnutils if needed: apt install coturn
turnutils_stunclient yourdomain.com
# Should return your public IP
```

---

## Step 7: Deploy Server Code

```bash
mkdir -p /opt/visual911
cd /opt/visual911

# Upload server.py and dashboard/index.html
# (scp, rsync, or git clone)
scp -r server/ root@YOUR_VPS_IP:/opt/visual911/
scp dashboard/index.html root@YOUR_VPS_IP:/opt/visual911/static/

mkdir -p /opt/visual911/static
```

---

## Step 8: Environment Variables

```bash
cat > /opt/visual911/.env << 'EOF'
GEMINI_API_KEY=your-gemini-api-key-here
TURN_SECRET=SAME_SECRET_AS_COTURN_STATIC_AUTH_SECRET
PORT=443
SSL_CERT=/etc/letsencrypt/live/visual911.mooo.com/fullchain.pem
SSL_KEY=/etc/letsencrypt/live/visual911.mooo.com/privkey.pem
EOF
```

---

## Step 9: Run Server with SSL

Update `server.py` to add SSL support:

```python
import ssl

if __name__ == "__main__":
    port = int(os.environ.get("PORT", 443))
    ssl_cert = os.environ.get("SSL_CERT")
    ssl_key = os.environ.get("SSL_KEY")

    if ssl_cert and ssl_key:
        ssl_context = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
        ssl_context.load_cert_chain(ssl_cert, ssl_key)
    else:
        ssl_context = None

    web.run_app(create_app(), port=port, ssl_context=ssl_context)
```

Start the server:
```bash
cd /opt/visual911
source .env
python3 server/server.py &
```

---

## Step 10: Systemd Service (Keep it running)

```bash
cat > /etc/systemd/system/visual911.service << 'EOF'
[Unit]
Description=Visual911 Server
After=network.target

[Service]
Type=simple
WorkingDirectory=/opt/visual911
EnvironmentFile=/opt/visual911/.env
ExecStart=/usr/bin/python3 server/server.py
Restart=always
RestartSec=5
User=root

[Install]
WantedBy=multi-user.target
EOF

systemctl daemon-reload
systemctl enable visual911
systemctl start visual911
systemctl status visual911  # Should show "active (running)"
```

View logs:
```bash
journalctl -u visual911 -f
```

---

## Step 11: Update iOS Config

In `ios/Config.swift`:
```swift
static let serverHost = "wss://visual911.mooo.com"
```

In `ios/Secrets.swift`, set the TURN secret matching the coturn `static-auth-secret`:
```swift
static let turnSecret = "SAME_SECRET_AS_COTURN"
```

**Dashboard TURN credentials are automatic.** The dashboard fetches time-limited credentials from `/api/turn-credentials` on each call — no hardcoded TURN creds needed in `index.html`. The server uses the `TURN_SECRET` env var (Step 8) to generate HMAC-SHA1 credentials matching coturn's `static-auth-secret`.

The iOS app generates its own TURN credentials locally using `TURNCredentials.swift` with the same shared secret.

---

## Verify Full Stack

```bash
# 1. Server responding
curl -k https://visual911.mooo.com/
# Should serve index.html

# 2. WebSocket reachable (install wscat: npm install -g wscat)
wscat -c wss://visual911.mooo.com/ws/signal?call_id=test&role=caller
# Should connect without error

# 3. coturn TURN working
turnutils_uclient -T visual911.mooo.com -e YOUR_VPS_IP
# Should complete relay test
```

---

## SSL Certificate Renewal

Let's Encrypt certificates expire every 90 days. Auto-renewal is configured by certbot but the server needs to reload after renewal.

```bash
# Add post-renewal hook
cat > /etc/letsencrypt/renewal-hooks/post/visual911.sh << 'EOF'
#!/bin/bash
systemctl restart visual911
EOF
chmod +x /etc/letsencrypt/renewal-hooks/post/visual911.sh
```

---

## Troubleshooting

**coturn not starting:**
```bash
journalctl -u coturn -n 50
# Common cause: SSL cert path wrong, or port 3478 already in use
```

**WebRTC not connecting (video freezes):**
- TURN server usually the culprit on restrictive networks
- Verify TURN credentials match between server and dashboard
- Check coturn logs: `tail -f /var/log/turnserver.log`

**Gemini 429 errors:**
- Free tier: 10 RPM, 250 RPD
- Each call open/close = 1 request
- Avoid rapid reconnect loops during testing

**SSL cert not found:**
```bash
ls /etc/letsencrypt/live/visual911.mooo.com/
# fullchain.pem and privkey.pem should exist
certbot renew --dry-run  # Test renewal
```

**iOS can't connect:**
- Confirm `Config.swift` has `wss://` not `ws://`
- Confirm port 443 is open in firewall
- Test from browser first: open `https://visual911.mooo.com` — if dashboard loads, network is fine
- Verify TURN credentials: `curl https://visual911.mooo.com/api/turn-credentials` should return JSON with username, credential, uris
