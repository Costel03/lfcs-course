# Solutions 11 — Network services and certificates

---

## 🟢 Warm-up

**11.1** (ubuntu1)

```bash
cat >> ~/.ssh/config <<'EOF'
Host u2
    HostName ubuntu2
    User vagrant
    IdentityFile ~/.ssh/id_ed25519

Host r1
    HostName rocky1
    User vagrant
    IdentityFile ~/.ssh/id_ed25519
EOF
chmod 600 ~/.ssh/config
ssh u2 hostname; ssh r1 hostname
```

**11.2**

```bash
sudo sshd -T | grep -Ei '^(permitrootlogin|passwordauthentication|port) '
```

`sshd -T` prints the configuration after every include and default is applied — the only
reliable answer when drop-ins are involved.

**11.3**

```bash
sudo ufw status                       # ubuntu: usually "inactive"
sudo nft list ruleset                 # empty, or tables created by other tools
sudo firewall-cmd --state             # rocky1: running
```

**11.4**

```bash
openssl s_client -connect ubuntu.com:443 -servername ubuntu.com </dev/null 2>/dev/null \
  | openssl x509 -noout -issuer -dates -ext subjectAltName
```

---

## 🔵 Practical

**11.5** (ubuntu2 — keep a session open)

```bash
sudo groupadd sshusers
sudo usermod -aG sshusers vagrant
sudo tee /etc/ssh/sshd_config.d/50-hardening.conf > /dev/null <<'EOF'
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
AllowGroups sshusers
EOF
sudo sshd -t
sudo systemctl reload ssh
```

From ubuntu1, in a new terminal:

```bash
ssh u2 true && echo ok
ssh -o PubkeyAuthentication=no u2      # Permission denied (publickey).
```

Group membership is read at login, so `vagrant`'s new connections carry `sshusers`
immediately. If you'd added the `AllowGroups` line *before* the group membership, only your
already-open session would have saved you.

**11.6** (ubuntu2)

```bash
printf 'Port 22\nPort 2222\n' | sudo tee /etc/ssh/sshd_config.d/40-ports.conf
sudo sshd -t
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
sudo ss -tlnp | grep -E ':(22|2222)\b'
```

From ubuntu1: `ssh -p 2222 u2 hostname`. Firewalls: none yet on ubuntu2 — but in 11.9 you'll
need to allow 2222 too.

Specifying `Port` at all replaces the default, which is why `Port 22` must be listed
alongside `2222`. On systems without socket activation, `systemctl restart ssh` is enough.

**11.7**

```bash
# ubuntu2
python3 -m http.server 8081 --bind 127.0.0.1 --directory /srv/app &
# ubuntu1
ssh -N -f -L 9091:localhost:8081 u2
curl -s localhost:9091
```

`localhost` in `-L 9091:localhost:8081` is resolved **on ubuntu2** — the far end of the
tunnel. `-f` backgrounds ssh after authentication.

**11.8**

```bash
ssh -J u2 vagrant@rocky1 hostname
cat >> ~/.ssh/config <<'EOF'

Host r1via
    HostName rocky1
    User vagrant
    ProxyJump u2
EOF
ssh r1via hostname
```

The connection to rocky1 is encrypted end-to-end from ubuntu1; ubuntu2 only relays TCP.
Your private key never leaves ubuntu1 — unlike the habit of `ssh u2` then `ssh rocky1` from
there, which needs a key on ubuntu2.

**11.9** (ubuntu2)

```bash
sudo ufw disable 2>/dev/null
sudo tee /etc/nftables.conf > /dev/null <<'EOF'
#!/usr/sbin/nft -f
flush ruleset

table inet filter {
    chain input {
        type filter hook input priority filter; policy drop;
        ct state established,related accept
        ct state invalid drop
        iif "lo" accept
        ip protocol icmp accept
        ip6 nexthdr icmpv6 accept
        tcp dport { 22, 2222 } accept
        ip saddr 192.168.56.0/24 tcp dport 8000 accept
    }
    chain forward {
        type filter hook forward priority filter; policy drop;
    }
    chain output {
        type filter hook output priority filter; policy accept;
    }
}
EOF
sudo nft -c -f /etc/nftables.conf
sudo systemctl enable --now nftables
sudo systemctl restart nftables
sudo nft list ruleset
```

The forward policy is `drop` — remember that in 11.22, and note it also stops the routing
you set up in chapter 10's 10.12. That's realistic: a host firewall applies to forwarded
traffic too.

**11.10** (ubuntu1)

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH
sudo ufw allow 80/tcp
sudo ufw allow 443/tcp
sudo ufw allow from 192.168.56.12 to any port 8000 proto tcp
sudo ufw enable
sudo ufw status numbered
```

**11.11** (rocky1)

```bash
sudo firewall-cmd --permanent --add-service=http
sudo firewall-cmd --permanent --add-port=8000/tcp
sudo firewall-cmd --permanent --remove-service=cockpit
sudo firewall-cmd --reload
sudo firewall-cmd --list-all
```

**11.12** (ubuntu2) — add to `/etc/nftables.conf`:

```text
table ip nat {
    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;
        tcp dport 80 redirect to :8000
    }
    chain output {
        type nat hook output priority -100; policy accept;
        oifname "lo" tcp dport 80 redirect to :8000
    }
}
```

```bash
sudo nft -c -f /etc/nftables.conf && sudo systemctl restart nftables
curl -s localhost         # on ubuntu2
curl -s ubuntu2           # from ubuntu1
```

No filter change was needed: NAT happens in `prerouting`, **before** `input`, so the filter
sees port 8000 — which 11.9 already allows from the lab network. Port 80 itself never needs
opening. Local traffic is accepted by the `lo` rule.

**11.13** (rocky1)

```bash
sudo firewall-cmd --permanent --add-forward-port=port=8888:proto=tcp:toport=8000:toaddr=192.168.56.12
sudo firewall-cmd --permanent --add-masquerade
sudo firewall-cmd --reload
sysctl net.ipv4.ip_forward          # firewalld enables it for masquerading; set it if 0
```

From ubuntu1: `curl -s rocky1:8888` → `ubuntu2`.

Masquerade is essential: ubuntu1 and ubuntu2 share a network, so without it ubuntu2 would
answer ubuntu1 directly, and ubuntu1 would discard a reply from an address it never
contacted. With it, ubuntu2 sees the request coming from rocky1 — which ubuntu2's nftables
allows, being on the lab network.

**11.14** (ubuntu1)

```bash
sudo rm /etc/nginx/sites-enabled/default
sudo tee /etc/nginx/sites-available/app.conf > /dev/null <<'EOF'
server {
    listen 80;
    server_name app.lab.local;
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF
sudo ln -s /etc/nginx/sites-available/app.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
curl -s http://app.lab.local/
```

**11.15** (ubuntu1)

```bash
sudo tee /etc/nginx/sites-available/lb.conf > /dev/null <<'EOF'
upstream app_pool {
    server 192.168.56.12:8000 max_fails=1 fail_timeout=10s;
    server 192.168.56.13:8000 max_fails=1 fail_timeout=10s;
}
server {
    listen 80;
    server_name lb.lab.local;
    location / {
        proxy_pass http://app_pool;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_connect_timeout 2s;
    }
}
EOF
sudo ln -s /etc/nginx/sites-available/lb.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
for i in 1 2 3 4; do curl -s lb.lab.local; done
ssh r1 sudo systemctl stop app
for i in 1 2 3 4; do curl -s lb.lab.local; done      # all ubuntu2
ssh r1 sudo systemctl start app
```

With a stopped backend nginx gets *connection refused*, retries the request on the next
server (`proxy_next_upstream` defaults to errors and timeouts), and marks the dead one
failed for `fail_timeout`. The client never sees the failure — passive health checking.

**11.16**

```bash
cd ~/lab11/tls
openssl req -x509 -newkey rsa:2048 -nodes -days 365 \
  -keyout selfsigned.key -out selfsigned.crt \
  -subj "/CN=app.lab.local" \
  -addext "subjectAltName=DNS:app.lab.local,IP:192.168.56.11"
openssl x509 -in selfsigned.crt -noout -ext subjectAltName -dates
```

**11.17**

```bash
cd ~/lab11/tls
openssl req -x509 -newkey rsa:4096 -nodes -days 3650 -keyout ca.key -out ca.crt -subj "/CN=Lab Root CA"
openssl req -newkey rsa:2048 -nodes -keyout app.key -out app.csr \
  -subj "/CN=app.lab.local" -addext "subjectAltName=DNS:app.lab.local"
printf 'subjectAltName=DNS:app.lab.local\n' > san.ext
openssl x509 -req -in app.csr -CA ca.crt -CAkey ca.key -CAcreateserial -days 825 \
  -out app.crt -extfile san.ext
openssl verify -CAfile ca.crt app.crt
openssl x509 -in app.crt -noout -pubkey | sha256sum
openssl pkey -in app.key -pubout | sha256sum
chmod 600 *.key
```

825 days is the longest validity many clients accept for server certificates; public CAs
issue far shorter ones now.

**11.18** (ubuntu1)

```bash
sudo install -d -m 755 /etc/nginx/tls
sudo install -m 644 ~/lab11/tls/app.crt /etc/nginx/tls/
sudo install -m 600 ~/lab11/tls/app.key /etc/nginx/tls/
sudo tee /etc/nginx/sites-available/app.conf > /dev/null <<'EOF'
server {
    listen 80;
    server_name app.lab.local;
    return 301 https://$host$request_uri;
}
server {
    listen 443 ssl;
    server_name app.lab.local;
    ssl_certificate     /etc/nginx/tls/app.crt;
    ssl_certificate_key /etc/nginx/tls/app.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    location / {
        proxy_pass http://127.0.0.1:8000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
EOF
sudo nginx -t && sudo systemctl reload nginx

# trust the CA on ubuntu1 and ubuntu2
sudo cp ~/lab11/tls/ca.crt /usr/local/share/ca-certificates/lab-root-ca.crt
sudo update-ca-certificates
scp ~/lab11/tls/ca.crt u2:/tmp/
ssh u2 'sudo cp /tmp/ca.crt /usr/local/share/ca-certificates/lab-root-ca.crt && sudo update-ca-certificates'
ssh u2 curl -s https://app.lab.local/
curl -sI http://app.lab.local/ | head -1        # HTTP/1.1 301 Moved Permanently
```

`update-ca-certificates` only picks up files ending in `.crt`.

---

## 🔴 Challenge

**11.19** (ubuntu2, in the open session)

```bash
sudo journalctl -u ssh --since '-5min' | grep -i -E 'refused|bad ownership'
#   Authentication refused: bad ownership or modes for file /home/vagrant/.ssh/authorized_keys
chmod 600 ~/.ssh/authorized_keys
```

sshd's `StrictModes yes` (the default) refuses keys anyone else could have written, because
whoever can write `authorized_keys` can log in as you.

**11.20** (ubuntu2)

```bash
sudo groupadd sftponly
sudo useradd -M -s /usr/sbin/nologin -G sftponly,sshusers uploader
printf '%s:%s\n' uploader 'Up1oad!' | sudo chpasswd
sudo install -d -o root -g root -m 755 /srv/sftp/uploader
sudo install -d -o uploader -g uploader -m 755 /srv/sftp/uploader/incoming

sudo tee -a /etc/ssh/sshd_config > /dev/null <<'EOF'

Match Group sftponly Address 192.168.56.0/24
    PasswordAuthentication yes
    ChrootDirectory /srv/sftp/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no
    X11Forwarding no
EOF
sudo sshd -t && sudo systemctl reload ssh

# prove the rule applies to uploader and not to vagrant
sudo sshd -T -C user=uploader,addr=192.168.56.11,host=ubuntu1 | grep -Ei 'passwordauth|forcecommand'
sudo sshd -T -C user=vagrant,addr=192.168.56.11,host=ubuntu1  | grep -Ei 'passwordauth|forcecommand'
```

Three traps:

- **`sshusers`** — `AllowGroups` from 11.5 applies to everyone, so uploader must be in it.
- **Chroot ownership** — the chroot directory and every directory above it must be owned by
  root and not writable by anyone else, or sshd refuses the session (*bad ownership or
  modes for chroot directory*). That's why uploads go into a subdirectory.
- **Where `Match` goes** — every option after a `Match` line belongs to that block until the
  next `Match`. Drop-ins are included at the *top* of `sshd_config`, so a `Match` in a
  drop-in risks capturing options that follow it, depending on the OpenSSH version.
  Appending `Match` blocks to the **end** of the main file avoids the question, and
  `sshd -T -C user=…,addr=…,host=…` shows the effective settings for a given connection —
  `forcecommand internal-sftp` for uploader, `passwordauthentication no` for vagrant.

**11.21**

```bash
sudo cp ~/lab11/tls/selfsigned.key /etc/nginx/tls/app.key
sudo nginx -t
#   SSL_CTX_use_PrivateKey(...) failed (... key values mismatch)
openssl x509 -in /etc/nginx/tls/app.crt -noout -pubkey | sha256sum
sudo openssl pkey -in /etc/nginx/tls/app.key -pubout | sha256sum      # different
sudo install -m 600 ~/lab11/tls/app.key /etc/nginx/tls/app.key
sudo nginx -t && sudo systemctl reload nginx
```

`nginx -t` caught it before the reload. A reload that had failed this way would have left
the old configuration running — `nginx -t` is cheap insurance.

**11.22**

On ubuntu2, extend `/etc/nftables.conf`:

```text
table inet filter {
    ...
    chain forward {
        type filter hook forward priority filter; policy drop;
        ct state established,related accept
        iifname "eth1" oifname "eth0" ip saddr 192.168.56.11 accept
    }
    ...
}

table ip nat {
    ... (the chains from 11.12) ...
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        oifname "eth0" ip saddr 192.168.56.0/24 masquerade
    }
}
```

```bash
sudo nft -c -f /etc/nftables.conf && sudo systemctl restart nftables
sysctl net.ipv4.ip_forward          # 1, from chapter 10
```

On ubuntu1:

```bash
sudo ip route replace default via 192.168.56.12 dev eth1
tracepath -n 1.1.1.1 | head -3
curl -sI https://ubuntu.com | head -1
sudo netplan apply                  # restore the original default route
```

Three pieces make a NAT gateway: **forwarding** enabled, the **forward chain** allowing the
traffic (and its replies), and **masquerade** on the way out. Miss any one and packets
vanish silently.

**11.23** (ubuntu1)

```bash
sudo tee /etc/haproxy/haproxy.cfg > /dev/null <<'EOF'
global
    log /dev/log local0
defaults
    mode http
    log global
    option httplog
    timeout connect 2s
    timeout client 30s
    timeout server 30s
frontend web
    bind *:8090
    default_backend app
backend app
    balance roundrobin
    option httpchk GET /
    server ubuntu2 192.168.56.12:8000 check inter 2s fall 2 rise 2
    server rocky1  192.168.56.13:8000 check inter 2s fall 2 rise 2
EOF
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl restart haproxy
ssh r1 sudo systemctl stop app
sleep 5; sudo journalctl -u haproxy --since '-1min' | grep -i down
for i in 1 2 3; do curl -s localhost:8090; done
ssh r1 sudo systemctl start app
```

Active checks probe every 2 seconds; after 2 failures (`fall 2`) the server is removed —
before a client request ever reaches it. nginx's passive checks need a real request to fail
first.

**11.24**

```bash
echo | openssl s_client -connect app.lab.local:443 -servername app.lab.local 2>/dev/null \
  | openssl x509 -noout -checkend $((30*24*3600)); echo "exit=$?"
```

`-checkend N` exits `1` if the certificate expires within N seconds. To test the failing
case, sign a new certificate with `-days 10`, install it, reload nginx, and run the command
again — then restore the 825-day one.

---

## Quiz answers

| Q | Answer | Why |
|---|---|---|
| 1 | `~/.ssh` 700, `authorized_keys` 600 | `StrictModes` refuses files others can write |
| 2 | Connecting to local port 8080 reaches port 5432 as seen *from db01* | Typical for a database bound to localhost |
| 3 | `ssh -J bastion internal01` | |
| 4 | The drop-in — it's included at the top, and sshd uses the first value it reads | |
| 5 | `sshd -t`; `sshd -T \| grep -i passwordauthentication` | |
| 6 | `systemctl daemon-reload` and `systemctl restart ssh.socket` | The socket owns the port on Ubuntu 22.10+ |
| 7 | prerouting, input, forward, output, postrouting; DNAT in prerouting (and output for local), masquerade in postrouting | |
| 8 | Otherwise replies to the machine's own outgoing connections (DNS, updates) are dropped | |
| 9 | One table for both IPv4 and IPv6 | |
| 10 | It was runtime only; add `--permanent` and `--reload` (or `--runtime-to-permanent`) | |
| 11 | Masquerade rewrites the **source** to the outgoing interface's address; DNAT rewrites the **destination** | |
| 12 | Locally generated packets don't traverse prerouting; add the redirect in the `output` nat hook | |
| 13 | Masquerade/SNAT so replies return through the NAT host (and `ip_forward`) | |
| 14 | The backend otherwise sees only the proxy's address as the client | |
| 15 | Without a path, `/api/users` is passed as-is; with `/`, the `/api/` prefix is replaced → `/users` | |
| 16 | round-robin (default), `least_conn`, `ip_hash`; `ip_hash` for session stickiness | |
| 17 | A Subject Alternative Name for that host | Clients match SAN, not CN |
| 18 | Compare the hash of `openssl x509 -pubkey` and `openssl pkey -pubout` | |
| 19 | Sends SNI so a multi-site server presents the right certificate | |
| 20 | Copy it to `/usr/local/share/ca-certificates/<name>.crt` and run `update-ca-certificates` | |
