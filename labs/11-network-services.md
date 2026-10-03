# Lab 11 — Network services and certificates

24 tasks across all three nodes. Firewall and SSH tasks can lock you out: **keep one SSH
session open while you test in a second**, and take a snapshot first
(`vagrant snapshot save pre-services`). Do the quiz first.

**Setup:**

```bash
# all three nodes: a tiny web app that reports which host answered
sudo mkdir -p /srv/app && hostname | sudo tee /srv/app/index.html
sudo tee /etc/systemd/system/app.service > /dev/null <<'EOF'
[Service]
ExecStart=/usr/bin/python3 -m http.server 8000 --directory /srv/app
[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload && sudo systemctl enable --now app
# ubuntu1 only
sudo apt install -y nginx haproxy
mkdir -p ~/lab11/tls
```

Add to `/etc/hosts` on all three nodes: `192.168.56.11 app.lab.local lb.lab.local`.

---

## 🟢 Warm-up

**11.1** On ubuntu1, create an SSH config so that `ssh u2` and `ssh r1` reach ubuntu2 and
rocky1 as `vagrant`, using your ed25519 key.
*Verify:* `ssh u2 hostname` prints `ubuntu2`; `ssh r1 hostname` prints `rocky1`.

**11.2** On ubuntu2, show the *effective* values of `PermitRootLogin`,
`PasswordAuthentication` and `Port`.
*Verify:* the values came from `sshd -T`, not from reading the file.

**11.3** On each node, find out which firewall front-end is active and whether it's
filtering anything.
*Verify:* one answer per node from `ufw status`, `nft list ruleset` or `firewall-cmd --state`.

**11.4** Inspect the certificate served by `ubuntu.com`: issuer, validity dates and SANs.
*Verify:* one pipeline from `openssl s_client` into `openssl x509`.

---

## 🔵 Practical

**11.5** Harden SSH on **ubuntu2** with a drop-in: no root login, no password
authentication, and only members of a new group `sshusers` may log in. Add `vagrant` to the
group *first*. Validate before reloading, and prove it works from a new session.
*Verify:* `sudo sshd -t` silent; `ssh u2` works; `ssh -o PubkeyAuthentication=no u2` is
refused.

**11.6** Make sshd on **ubuntu2** listen on **both** 22 and 2222.
*Verify:* `sudo ss -tlnp | grep -E ':(22|2222)\b'` shows both; `ssh -p 2222 u2 hostname` works.

**11.7** On ubuntu2, run `python3 -m http.server 8081 --bind 127.0.0.1 --directory /srv/app &`.
From ubuntu1, reach it through an SSH tunnel on local port 9091.
*Verify:* `curl -s localhost:9091` on ubuntu1 prints `ubuntu2`.

**11.8** Reach rocky1 from ubuntu1 *through* ubuntu2 with a single `ssh` command, then make
it permanent in your SSH config as host `r1via`.
*Verify:* `ssh r1via hostname` prints `rocky1`, and `ss` on ubuntu2 shows the session
originating there.

**11.9** On **ubuntu2**, replace any existing firewall with an nftables ruleset in
`/etc/nftables.conf`: drop by default; allow established traffic, loopback, ICMP/ICMPv6, SSH
on 22 and 2222, and port 8000 only from `192.168.56.0/24`. Load it at boot.
*Verify:* from ubuntu1, `ssh u2` and `curl -s ubuntu2:8000` work; `nc -zv -w3 ubuntu2 8081`
times out; `systemctl is-enabled nftables` is `enabled`.

**11.10** On **ubuntu1**, enable `ufw`: deny incoming by default, allow SSH, allow port 80
and 443 from anywhere, and allow port 8000 **only from ubuntu2**.
*Verify:* `curl ubuntu1:8000` works from ubuntu2 and times out from rocky1; `ufw status
numbered` lists your rules.

**11.11** On **rocky1** with firewalld: permanently allow the `http` service and port
`8000/tcp`, and remove `cockpit` if present.
*Verify:* `sudo firewall-cmd --list-all` after `--reload` shows them; `curl rocky1:8000` from
ubuntu1 works.

**11.12** On **ubuntu2**, make port 80 deliver to the app on 8000 — for remote clients *and*
for `curl localhost` on ubuntu2 itself — using nftables. Keep the filter from 11.9 working.
*Verify:* `curl -s ubuntu2` from ubuntu1 and `curl -s localhost` on ubuntu2 both print
`ubuntu2`.

**11.13** On **rocky1**, forward its port `8888` to `ubuntu2:8000`, with firewalld.
*Verify:* from ubuntu1, `curl -s rocky1:8888` prints `ubuntu2`.

**11.14** On **ubuntu1**, configure nginx as a reverse proxy: `http://app.lab.local/` serves
ubuntu1's own app on `127.0.0.1:8000`, passing the client's address in `X-Real-IP` and
`X-Forwarded-For`. Disable the default site.
*Verify:* `curl -s http://app.lab.local/` prints `ubuntu1`; `sudo nginx -t` passes.

**11.15** On **ubuntu1**, add a load balancer `http://lb.lab.local/` across the apps on
ubuntu2 and rocky1 (port 8000), round-robin. Then stop the app on rocky1 and show that
traffic shifts.
*Verify:* `for i in 1 2 3 4; do curl -s lb.lab.local; done` alternates; after
`ssh r1 sudo systemctl stop app`, every answer is `ubuntu2`.

**11.16** Create a self-signed certificate and key for `app.lab.local`, valid 365 days, with
SANs `DNS:app.lab.local` and `IP:192.168.56.11`, key without passphrase.
*Verify:* `openssl x509 -noout -ext subjectAltName` shows both SANs; `-dates` spans a year.

**11.17** Create a private CA `Lab Root CA`, then a key and CSR for `app.lab.local`, and sign
it with the CA (825 days, SAN included).
*Verify:* `openssl verify -CAfile ca.crt app.crt` → `OK`; the key's and certificate's
public-key hashes match.

**11.18** Serve `https://app.lab.local/` from nginx with the CA-signed certificate; redirect
HTTP to HTTPS. Install the CA in ubuntu1's **and** ubuntu2's trust stores.
*Verify:* from ubuntu2, `curl -s https://app.lab.local/` prints `ubuntu1` **without** `-k`;
`curl -sI http://app.lab.local/` shows `301`.

---

## 🔴 Challenge

**11.19 — Why won't my key work?** On ubuntu2 run `chmod 666 ~/.ssh/authorized_keys`. From
ubuntu1, key login fails. Find the reason **in ubuntu2's logs** (use your still-open
session) and fix it.
*Verify:* you quote the log line; `ssh u2` works again.

**11.20 — SFTP-only jail.** On ubuntu2, create user `uploader` (password `Up1oad!`) who can
only use SFTP, is confined to `/srv/sftp/uploader`, and can write to an `incoming`
subdirectory only. Password logins must be allowed for this user from the lab network
while staying disabled for everyone else.
*Verify:* `sftp uploader@ubuntu2` works and `put` into `incoming` succeeds;
`ssh uploader@ubuntu2` gets no shell; `ls /` inside sftp shows only `incoming`.

**11.21 — Mismatched key.** Give nginx the self-signed **key** from 11.16 with the CA-signed
**certificate** from 11.17. Read the error, prove the mismatch with hashes, and fix it.
*Verify:* `sudo nginx -t` fails with *key values mismatch*, then passes.

**11.22 — A NAT gateway.** Make ubuntu1 reach the internet **through ubuntu2**: ubuntu1's
default route points to `192.168.56.12`, and ubuntu2 forwards and masquerades out of its
own `eth0`. Keep ubuntu2's firewall from 11.9 in place (you'll need to change its forward
policy carefully). Undo afterwards.
*Verify:* on ubuntu1, `tracepath -n 1.1.1.1` shows `192.168.56.12` as the first hop and
`curl -sI https://ubuntu.com` works; afterwards `sudo netplan apply` restores the old route.

**11.23 — HAProxy with health checks.** On ubuntu1, run HAProxy on port `8090` balancing the
two backends from 11.15 with **active** HTTP health checks. Stop one backend and watch
HAProxy mark it down — before any client request fails.
*Verify:* `journalctl -u haproxy` shows the server going DOWN; curl through 8090 keeps working.

**11.24 — Certificate expiry check.** Write a one-line command that exits non-zero if the
certificate served on `app.lab.local:443` expires within 30 days, suitable for a monitoring
cron job.
*Verify:* exit status `0` for your 825-day certificate; recreate a 10-day certificate and
the same command exits `1`.

---

## Self-check

- [ ] I can set up keys, an SSH config, tunnels and jump hosts
- [ ] I change sshd with a drop-in, test with `sshd -t`, and never close my only session
- [ ] I can write a persistent default-drop ruleset with nftables, ufw and firewalld
- [ ] I can do masquerade, DNAT and local redirection, and know which hook each uses
- [ ] I can configure nginx as a reverse proxy and load balancer, and test it before reloading
- [ ] I can make a CA, sign a certificate with a SAN, verify it, and trust the CA system-wide

➡ **Solutions:** [solutions/11-network-services.md](../solutions/11-network-services.md)
