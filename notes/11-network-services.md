# Chapter 11 — Network services and certificates

Chapter 10 got packets moving. This chapter controls what they're allowed to do and what
answers them: SSH for remote administration, the kernel firewall for filtering, NAT and
port redirection, nginx as a reverse proxy and load balancer, and the TLS certificates that
secure all of it.

LFCS: *Networking* — configure the OpenSSH server and client; configure packet filtering,
port redirection and NAT; implement reverse proxies and load balancers. *Essential
Commands* — work with SSL certificates.

## Objectives

After this chapter you can:

- Configure the SSH client with keys, a config file, jump hosts and port forwards.
- Harden `sshd` safely — validating the configuration before you lose access.
- Write a default-deny firewall with nftables, ufw or firewalld, and make it persistent.
- Configure masquerading NAT, destination NAT and local port redirection.
- Put nginx in front of an application as a reverse proxy, and load-balance across backends.
- Create keys, CSRs, self-signed certificates and a private CA; inspect and verify any
  certificate, local or remote; and add a CA to the system trust store.

---

## 1. The SSH client

### Keys

```bash
ssh-keygen -t ed25519 -C "costel@laptop"          # ~/.ssh/id_ed25519 and .pub
ssh-copy-id vagrant@ubuntu2                         # append the public key to authorized_keys
ssh ubuntu2
```

Permissions matter — sshd **ignores** keys in a directory or file others can write:

| Path | Mode |
|---|---|
| `~/.ssh/` | `700` |
| `~/.ssh/authorized_keys` | `600` |
| `~/.ssh/id_ed25519` (private) | `600` |

`ssh-agent` holds decrypted keys so you type a passphrase once per session:
`eval "$(ssh-agent)" && ssh-add`.

### ~/.ssh/config

```text
Host web
    HostName 192.168.56.12
    User vagrant
    Port 22
    IdentityFile ~/.ssh/id_ed25519

Host internal-*
    ProxyJump bastion
    User admin

Host *
    ServerAliveInterval 30
```

Now `ssh web` replaces `ssh -i ~/.ssh/id_ed25519 -p 22 vagrant@192.168.56.12`. The first
matching value for each option wins, so put specific hosts above `Host *`.

### Copying files

```bash
scp file.txt ubuntu2:/tmp/
scp -r ubuntu2:/etc/nginx ./nginx-backup
rsync -az --progress dir/ ubuntu2:/srv/dir/
sftp ubuntu2
```

### Tunnels

| Option | Direction | Example | Use |
|---|---|---|---|
| `-L` | local port → remote target | `ssh -L 8080:localhost:80 web` | reach a service only listening on the server's loopback |
| `-R` | remote port → local target | `ssh -R 9000:localhost:3000 web` | expose your local service on the server |
| `-D` | SOCKS proxy | `ssh -D 1080 bastion` | browse through the remote network |
| `-J` | jump host | `ssh -J bastion internal01` | reach a host only the bastion can see |

```bash
ssh -N -L 8080:localhost:80 web      # -N: no shell, just the tunnel
curl localhost:8080                  # arrives at port 80 on "web", as if from web itself
```

### Host keys

The first connection asks you to confirm the server's fingerprint and stores it in
`~/.ssh/known_hosts`. If it later changes, ssh refuses loudly
(*REMOTE HOST IDENTIFICATION HAS CHANGED*). That's either a reinstalled server — remove the
old entry with `ssh-keygen -R host` — or an attack. Find out which before you remove it.

---

## 2. The SSH server

`/etc/ssh/sshd_config` includes `/etc/ssh/sshd_config.d/*.conf` **at the top**, and for
each option **the first value read wins**. A drop-in is therefore the right place for your
changes:

```text
# /etc/ssh/sshd_config.d/50-hardening.conf
PermitRootLogin no
PasswordAuthentication no
KbdInteractiveAuthentication no
PubkeyAuthentication yes
AllowGroups sshusers
MaxAuthTries 3
ClientAliveInterval 300
X11Forwarding no
```

```bash
sudo sshd -t                     # syntax and semantic check — silent means OK
sudo sshd -T | grep -i passwordauth    # the effective value after all includes
sudo systemctl reload ssh        # sshd on RedHat
```

**Never close your session after changing sshd until you've opened a new one.** Keep the
working session; test in a second terminal; only then log out. `sshd -t` catches syntax,
not lockouts — like `AllowGroups` that doesn't include you.

### Per-user and per-network rules

```text
Match Group sftponly
    ChrootDirectory /srv/sftp/%u
    ForceCommand internal-sftp
    AllowTcpForwarding no

Match Address 192.168.56.0/24
    PasswordAuthentication yes
```

`Match` blocks go at the **end** of `sshd_config` — everything after a `Match` line belongs
to it until the next `Match`. Keep them out of the drop-in directory, which is included at the
*top*. Check the effective settings for a particular connection with
`sudo sshd -T -C user=alice,addr=192.168.56.11,host=client`.

### Changing the port

```text
Port 2222
```

On Ubuntu 22.10 and later, `ssh.socket` owns the listening port. Its configuration is
generated from `sshd_config`, so after changing `Port` or `ListenAddress`:

```bash
sudo systemctl daemon-reload
sudo systemctl restart ssh.socket
```

On RedHat, SELinux must also allow the new port:
`sudo semanage port -a -t ssh_port_t -p tcp 2222` (chapter 13). Open it in the firewall
before restarting.

---

## 3. Packet filtering

Linux filtering happens in the kernel's **netfilter** framework. Packets pass through
**hooks**; you attach **chains** of rules to them:

```text
             ┌──────────── INPUT ──▶ local process ── OUTPUT ─────────┐
 in ─▶ PREROUTING ─▶ routing                                            ├─▶ POSTROUTING ─▶ out
             └──────────────────────── FORWARD ───────────────────────┘
```

| Hook | Sees packets that are… |
|---|---|
| `prerouting` | arriving, before the routing decision — **DNAT** happens here |
| `input` | addressed to this machine |
| `forward` | passing through (router) |
| `output` | generated locally |
| `postrouting` | leaving — **SNAT / masquerade** happens here |

Three front-ends drive it. Use **one** per machine:

| Tool | Default on | Style |
|---|---|---|
| `nftables` (`nft`) | the kernel interface — everywhere | full control, one ruleset file |
| `ufw` | Ubuntu | simple allow/deny rules |
| `firewalld` | RedHat | zones and services, runtime vs permanent |
| `iptables` | legacy | still common in scripts; on modern systems it writes nftables rules |

### nftables

```text
#!/usr/sbin/nft -f
# /etc/nftables.conf
flush ruleset

table inet filter {
    chain input {
        type filter hook input priority filter; policy drop;
        ct state established,related accept
        ct state invalid drop
        iif "lo" accept
        ip protocol icmp accept
        ip6 nexthdr icmpv6 accept
        tcp dport 22 accept
        tcp dport { 80, 443 } accept
        ip saddr 192.168.56.0/24 tcp dport 9100 accept
        log prefix "nft-drop: " limit rate 5/minute
    }
    chain forward {
        type filter hook forward priority filter; policy drop;
    }
    chain output {
        type filter hook output priority filter; policy accept;
    }
}
```

```bash
sudo nft -c -f /etc/nftables.conf      # check syntax without applying
sudo nft -f /etc/nftables.conf         # apply
sudo systemctl enable --now nftables   # load it at boot
sudo nft list ruleset
sudo nft -a list chain inet filter input            # with handles
sudo nft add rule inet filter input tcp dport 8080 accept     # live — not saved
sudo nft delete rule inet filter input handle 12
```

`table inet` covers IPv4 **and** IPv6 together. Rule order matters: the first match decides.
`ct state established,related accept` near the top lets replies to your own connections
back in — forget it and the machine can't even do DNS lookups.

**Order of operations when applying a drop policy over SSH:** allow port 22 *before* the
policy takes effect — which loading a complete file does atomically. Typing rules one by
one with `nft add` after setting `policy drop` locks you out between two commands.

### ufw

```bash
sudo ufw default deny incoming
sudo ufw default allow outgoing
sudo ufw allow OpenSSH                  # or: sudo ufw allow 22/tcp
sudo ufw allow from 192.168.56.0/24 to any port 9100 proto tcp
sudo ufw limit 22/tcp                   # rate-limit brute force
sudo ufw enable
sudo ufw status numbered
sudo ufw delete 3
```

`ufw allow` **before** `ufw enable` over SSH — or you're out.

### firewalld

```bash
sudo firewall-cmd --get-active-zones
sudo firewall-cmd --list-all
sudo firewall-cmd --add-service=http                  # runtime only
sudo firewall-cmd --permanent --add-service=http      # saved, not yet active
sudo firewall-cmd --reload                            # load permanent into runtime
sudo firewall-cmd --permanent --add-port=8080/tcp
sudo firewall-cmd --permanent --zone=internal --add-source=192.168.56.0/24
sudo firewall-cmd --runtime-to-permanent              # save what you tested live
```

**Runtime versus permanent** is firewalld's equivalent of `ip` versus Netplan: a rule
without `--permanent` is gone at the next reload or reboot.

---

## 4. NAT and port redirection

### Masquerade — share one address (SNAT)

A gateway rewrites the source address of outgoing packets to its own, so a private network
can reach the internet:

```text
table ip nat {
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        ip saddr 10.30.0.0/24 oifname "eth0" masquerade
    }
}
```

Also needs `net.ipv4.ip_forward = 1` (chapter 10) and a `forward` chain that allows the
traffic.

### Destination NAT — port forwarding

Send connections arriving on this host's port 8080 to a backend:

```text
table ip nat {
    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;
        tcp dport 8080 dnat to 192.168.56.12:80
    }
    chain postrouting {
        type nat hook postrouting priority srcnat; policy accept;
        ip daddr 192.168.56.12 tcp dport 80 masquerade
    }
}
```

The masquerade line is needed when the client and backend are on the same network: without
it, the backend replies straight to the client, which drops the answer because it came
from the wrong address.

### Local redirection

Redirect a port to another port on the same machine — e.g. an app on 8080 without root,
reachable on 80:

```text
table ip nat {
    chain prerouting {
        type nat hook prerouting priority dstnat; policy accept;
        tcp dport 80 redirect to :8080
    }
    chain output {
        type nat hook output priority -100; policy accept;
        oifname "lo" tcp dport 80 redirect to :8080
    }
}
```

`prerouting` sees only packets arriving from outside. Connections from the machine itself
(`curl localhost`) go through **`output`** — the second chain makes them redirect too.

### firewalld equivalents

```bash
sudo firewall-cmd --permanent --add-masquerade
sudo firewall-cmd --permanent --add-forward-port=port=80:proto=tcp:toport=8080
sudo firewall-cmd --permanent --add-forward-port=port=8080:proto=tcp:toport=80:toaddr=192.168.56.12
sudo firewall-cmd --reload
```

ufw handles NAT through raw rules in `/etc/ufw/before.rules` (a `*nat` section) — workable,
but nftables is clearer.

---

## 5. Reverse proxies and load balancers

A **reverse proxy** receives client requests and forwards them to backend servers. It's
where you terminate TLS, add headers, cache, and spread load.

```text
client ──▶ nginx :80/:443 ──┬──▶ app1 192.168.56.12:8000
                            └──▶ app2 192.168.56.13:8000
```

### nginx as a reverse proxy

```nginx
# /etc/nginx/sites-available/app.conf   (conf.d/*.conf on RedHat)
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
```

```bash
sudo ln -s /etc/nginx/sites-available/app.conf /etc/nginx/sites-enabled/
sudo nginx -t                      # test configuration
sudo systemctl reload nginx
curl -H 'Host: app.lab.local' http://localhost/
```

Without the `X-Forwarded-For` / `X-Real-IP` headers, the backend sees every request coming
from the proxy's address.

**`proxy_pass` with or without a trailing path matters:**

| `location /api/` with | Request `/api/users` reaches backend as |
|---|---|
| `proxy_pass http://backend;` | `/api/users` |
| `proxy_pass http://backend/;` | `/users` |

### Load balancing

```nginx
upstream app_pool {
    least_conn;                           # default is round-robin
    server 192.168.56.12:8000 weight=2;
    server 192.168.56.13:8000 max_fails=3 fail_timeout=10s;
    server 192.168.56.14:8000 backup;     # used only when the others are down
}

server {
    listen 80;
    location / {
        proxy_pass http://app_pool;
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

| Method | Behaviour |
|---|---|
| (default) | round-robin, honouring `weight` |
| `least_conn` | to the server with fewest active connections |
| `ip_hash` | same client IP → same server (sticky sessions) |

Open-source nginx does **passive** health checks: a server failing `max_fails` times is
skipped for `fail_timeout`. HAProxy adds active checks:

```text
# /etc/haproxy/haproxy.cfg (excerpt)
frontend web
    bind *:80
    default_backend app

backend app
    balance roundrobin
    option httpchk GET /
    server app1 192.168.56.12:8000 check
    server app2 192.168.56.13:8000 check
```

`haproxy -c -f /etc/haproxy/haproxy.cfg` checks the file.

### Layer 4 balancing

nginx's `stream` module balances raw TCP (databases, SSH) instead of HTTP:

```nginx
# in nginx.conf, at the top level — NOT inside http { }
stream {
    upstream pg { server 192.168.56.12:5432; server 192.168.56.13:5432; }
    server { listen 5432; proxy_pass pg; }
}
```

(On Ubuntu the module ships as `libnginx-mod-stream`.)

---

## 6. TLS certificates

### The pieces

| Item | What it is | Typical file |
|---|---|---|
| Private key | secret half of a key pair — never leaves the server | `server.key` (mode 600) |
| CSR | certificate signing request: public key + identity, signed with the private key | `server.csr` |
| Certificate | public key + identity + validity, **signed by a CA** | `server.crt` |
| CA certificate | the issuer's certificate; clients must trust it | `ca.crt` |
| Chain | server cert + intermediate CA certs, in order | `fullchain.pem` |

A client trusts a certificate when: it chains to a CA in the client's trust store, it's
inside its validity dates, and its **Subject Alternative Name (SAN)** matches the hostname
the client asked for. Modern clients ignore the old Common Name field for matching — a
certificate without a SAN fails.

### Self-signed certificate, in one command

```bash
openssl req -x509 -newkey rsa:2048 -nodes -days 365 \
  -keyout server.key -out server.crt \
  -subj "/CN=app.lab.local" \
  -addext "subjectAltName=DNS:app.lab.local,DNS:localhost,IP:192.168.56.11"
```

`-nodes` means no passphrase on the key — necessary for a service to start unattended.

### A private CA

Self-signed certificates each need to be trusted individually. A private CA, trusted once,
signs everything:

```bash
# 1. the CA
openssl req -x509 -newkey rsa:4096 -nodes -days 3650 \
  -keyout ca.key -out ca.crt -subj "/CN=Lab Root CA"

# 2. a key and CSR for the server
openssl req -newkey rsa:2048 -nodes -keyout app.key -out app.csr \
  -subj "/CN=app.lab.local" -addext "subjectAltName=DNS:app.lab.local"

# 3. the CA signs it (the SAN must be given again — x509 -req doesn't copy extensions)
printf 'subjectAltName=DNS:app.lab.local\n' > san.ext
openssl x509 -req -in app.csr -CA ca.crt -CAkey ca.key -CAcreateserial \
  -days 825 -out app.crt -extfile san.ext
```

(OpenSSL 3.x can copy them with `-copy_extensions copyall`.)

### Inspecting and verifying

```bash
openssl x509 -in app.crt -noout -subject -issuer -dates
openssl x509 -in app.crt -noout -ext subjectAltName
openssl x509 -in app.crt -noout -text | less          # everything
openssl req  -in app.csr -noout -text                  # a CSR
openssl verify -CAfile ca.crt app.crt                  # app.crt: OK
openssl x509 -in app.crt -noout -checkend 2592000      # expires within 30 days? (exit status)
```

Does this key belong to this certificate?

```bash
openssl x509 -in app.crt -noout -pubkey | sha256sum
openssl pkey -in app.key -pubout | sha256sum           # the hashes must match
```

A remote server's certificate:

```bash
openssl s_client -connect app.lab.local:443 -servername app.lab.local </dev/null 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
openssl s_client -connect host:443 -servername host -showcerts </dev/null   # the whole chain
curl -v https://app.lab.local/                          # the TLS handshake, then the request
curl --cacert ca.crt https://app.lab.local/             # trust a specific CA for one request
```

`-servername` sends SNI — without it, a server hosting several sites may present the wrong
certificate.

### Trusting a CA system-wide

| Family | Put the CA here | Then run |
|---|---|---|
| Debian/Ubuntu | `/usr/local/share/ca-certificates/lab-ca.crt` (must end `.crt`) | `sudo update-ca-certificates` |
| RedHat | `/etc/pki/ca-trust/source/anchors/lab-ca.crt` | `sudo update-ca-trust` |

After that, `curl`, `wget`, Python and most system tools accept certificates signed by it.
(Browsers and Java may keep their own stores.)

### Formats

| Format | Looks like | Convert |
|---|---|---|
| PEM | text, `-----BEGIN CERTIFICATE-----` | the Linux default |
| DER | binary | `openssl x509 -in c.pem -outform der -out c.der` |
| PKCS#12 / PFX | binary bundle of key + certs, password-protected | `openssl pkcs12 -export -inkey k.key -in c.crt -out bundle.p12` |

### TLS in nginx

```nginx
server {
    listen 443 ssl;
    server_name app.lab.local;
    ssl_certificate     /etc/nginx/tls/app.crt;      # leaf + intermediates
    ssl_certificate_key /etc/nginx/tls/app.key;
    ssl_protocols       TLSv1.2 TLSv1.3;
    location / { proxy_pass http://127.0.0.1:8000; }
}

server {
    listen 80;
    server_name app.lab.local;
    return 301 https://$host$request_uri;           # redirect HTTP to HTTPS
}
```

---

## 7. Gotchas

**Locking yourself out of SSH.** Keep a session open; `sshd -t`; test with a new connection.

**Changing the SSH port on Ubuntu** without `daemon-reload` and restarting `ssh.socket`.

**Firewall drop policy before the allow rule** — over SSH, apply the whole ruleset at once.

**Runtime-only firewall rules** — `nft add`, `firewall-cmd` without `--permanent`, `iptables`
without saving: all gone at reboot.

**Two firewall tools at once** — ufw and a hand-written nftables file fighting over the same
hooks. Pick one.

**DNAT without forwarding** — the packet arrives and goes nowhere: enable `ip_forward` and
allow it in the `forward` chain.

**Testing local redirection from localhost** — needs a rule in the `output` hook too.

**nginx configuration not tested** — `nginx -t` before every reload.

**Certificates without a SAN** — rejected by modern clients even when the CN is right.

**Private keys readable by others** — `chmod 600`, owned by root or the service.

---

## 8. Commands introduced

| Command | Purpose |
|---|---|
| `ssh`, `ssh-keygen`, `ssh-copy-id`, `ssh-agent`, `scp`, `sftp` | SSH client |
| `sshd -t`, `sshd -T` | test and dump the server configuration |
| `nft` | nftables |
| `ufw` | Ubuntu firewall front-end |
| `firewall-cmd` | firewalld |
| `nginx -t`, `haproxy -c` | test proxy configuration |
| `openssl req / x509 / verify / s_client / pkey / pkcs12` | certificates |
| `update-ca-certificates`, `update-ca-trust` | system trust store |
| `man ssh_config`, `man sshd_config`, `man nft`, `man ufw`, `man firewall-cmd`, `man openssl-req`, `man openssl-x509` | offline references |
| `/usr/share/doc/nginx*`, `/usr/share/doc/nftables/examples/` | offline examples |

---

➡ **Next:** [quiz](../quizzes/11-network-services.md) → [lab](../labs/11-network-services.md)
