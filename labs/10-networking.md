# Lab 10 — Networking

24 tasks across all three nodes. Network changes can cut you off: **use `netplan try`**,
work on the spare interfaces where told, and take a snapshot first
(`vagrant snapshot save pre-net`). If you lock yourself out, the VirtualBox console still
works, or restore the snapshot. Do the quiz first.

**Interfaces on ubuntu1** (check with `ip -br link`):

| Interface | Network | Use |
|---|---|---|
| `eth0` | Vagrant NAT | internet access — **don't touch** |
| `eth1` | 192.168.56.0/24 | lab network — careful |
| `eth2`, `eth3` | isolated `bondnet` | yours to break |

Names may differ (`enp0s3`…) — substitute.

---

## 🟢 Warm-up

**10.1** In one command each, show all interfaces with their state and all addresses, in
brief form. Identify which interface reaches the internet and which is the lab network.
*Verify:* `ip -br link` and `ip -br addr`; `eth2`/`eth3` are `DOWN` with no address.

**10.2** Show the routing table, then ask the kernel how it would reach `8.8.8.8` and
`192.168.56.12`.
*Verify:* different interfaces in the two `ip route get` answers.

**10.3** Show the hostname, the `hosts:` line of `nsswitch.conf`, where `/etc/resolv.conf`
points, and the real upstream DNS servers.
*Verify:* `resolvectl status` lists servers that are *not* `127.0.0.53`.

**10.4** List every listening TCP and UDP socket with its process.
*Verify:* `sshd` (or `systemd`, for socket activation) on port 22.

**10.5** Show the time, time zone and whether the clock is synchronised, and which time
service is running.
*Verify:* `timedatectl` and `systemctl status` of the active service.

---

## 🔵 Practical

**10.6** With `ip` only, give `eth2` the address `10.10.10.11/24` and bring it up. Then
reboot ubuntu1 and check again. Explain.
*Verify:* the address exists, then is gone after the reboot.

**10.7** Persistently give `eth2` the addresses `10.99.0.11/24` and `fd00:99::11/64` using a
new file `/etc/netplan/60-lab.yaml`, applied with `netplan try`.
*Verify:* `ip -br addr show eth2` shows both; `sudo netplan generate` prints no warnings;
`sudo netplan get ethernets.eth2` shows the merged configuration.

**10.8** Rename ubuntu1's hostname to `ubuntu1.lab.local` persistently, and make the machine
resolve its own new name without DNS.
*Verify:* `hostnamectl` shows the new static name; `getent hosts ubuntu1.lab.local` answers.

**10.9** Make `test.lab.local` resolve to `192.168.56.12` on ubuntu1 only. Show that
applications see it while DNS tools don't.
*Verify:* `getent hosts test.lab.local` → `192.168.56.12`; `dig +short test.lab.local` → nothing.

**10.10** Make ubuntu1 use `9.9.9.9` as an additional DNS server for `eth1`, persistently,
and add `lab.local` as a search domain. Then look up `ubuntu.com` from a *specific* server,
and do a reverse lookup of `9.9.9.9`.
*Verify:* `resolvectl status eth1` shows the server and domain; `dig @1.1.1.1 ubuntu.com`
answers; `dig -x 9.9.9.9 +short` returns a name.

**10.11** Add a persistent static route on ubuntu1: `10.30.0.0/24` via `192.168.56.12`.
*Verify:* `ip route get 10.30.0.13` shows `via 192.168.56.12`; still present after
`netplan apply`.

**10.12** Make packets from ubuntu1 to `10.30.0.13` travel through ubuntu2 to rocky1:

```bash
# rocky1 — an address that only rocky1 has
sudo nmcli connection add type dummy ifname dummy0 con-name dummy0 \
  ipv4.method manual ipv4.addresses 10.30.0.13/24
# ubuntu2 — knows rocky1 is the way to 10.30.0.0/24
sudo ip route add 10.30.0.0/24 via 192.168.56.13
```

From ubuntu1, `ping 10.30.0.13` fails. Make it work by changing **one kernel setting on
ubuntu2**, persistently.
*Verify:* the ping succeeds; `sysctl net.ipv4.ip_forward` on ubuntu2 is `1` and a file in
`/etc/sysctl.d/` sets it.

**10.13** IPv6: ping ubuntu2's **link-local** address from ubuntu1. Then give `eth1` on both
Ubuntu nodes a unique-local address (`fd00:56::11/64`, `fd00:56::12/64`) persistently and
ping between them.
*Verify:* `ping -6 -c2 fe80::…%eth1` and `ping -6 -c2 fd00:56::12` both succeed.

**10.14** Replace the configuration of `eth2` from 10.7 with a bond `bond0` of `eth2` and `eth3`
in `active-backup` mode, address `10.99.0.11/24`, monitoring links every 100 ms. Then take the
active member down and watch the bond fail over.
*Verify:* `/proc/net/bonding/bond0` shows `active-backup`, two slaves, and a changed
`Currently Active Slave` after `sudo ip link set eth2 down`.

**10.15** Remove the bond and instead put `eth2` into a bridge `br0` with the address
`10.98.0.11/24`.
*Verify:* `bridge link` shows `eth2` as a port of `br0`; the address is on `br0`, not `eth2`.

**10.16** Make **ubuntu2** a time server for `192.168.56.0/24` using chrony, and make ubuntu1
use ubuntu2 as its preferred source.
*Verify:* on ubuntu1, `chronyc sources` lists `ubuntu2` (it becomes `^*` once synced);
on ubuntu2, `chronyc clients` lists ubuntu1.

**10.17** Set ubuntu1's time zone to `Europe/Bucharest`.
*Verify:* `timedatectl` shows it; `date` shows EET/EEST.

**10.18** Trace the path from ubuntu1 to `8.8.8.8`, and capture the DNS packets generated by
`dig ubuntu.com` with `tcpdump`.
*Verify:* `tracepath` lists hops; `tcpdump` shows a query and a response on port 53.

**10.19** On **rocky1**, using `nmcli` only, add a second address `192.168.56.113/24` to the
lab connection, a route `10.40.0.0/16` via `192.168.56.12`, and DNS server `192.168.56.12`,
all persistent.
*Verify:* `nmcli -f ipv4.addresses,ipv4.routes,ipv4.dns connection show <name>`; `ip addr`
shows both addresses after `nmcli connection up`.

---

## 🔴 Challenge

**10.20 — No internet.** On ubuntu1 run `sudo ip route del default`. Find the fault layer by
layer, write down which step first failed, and repair it *without rebooting*.
*Verify:* `curl -sI https://ubuntu.com | head -1` works again; you can name the layer.

**10.21 — Wrong answer.** Run `echo '10.255.255.1 ubuntu.com' | sudo tee -a /etc/hosts`.
Then `curl https://ubuntu.com` hangs. Diagnose using a command that shows what applications
resolve, prove DNS itself is fine, and fix it.
*Verify:* `getent hosts ubuntu.com` returns a real address after your fix.

**10.22 — Works on my machine.** On ubuntu2 run
`python3 -m http.server 8080 --bind 127.0.0.1 &`. From ubuntu1, `curl ubuntu2:8080` fails.
Prove the cause from both sides and fix it.
*Verify:* `ss -tlnp` on ubuntu2 shows `0.0.0.0:8080` (or `*:8080`); the curl from ubuntu1 works.

**10.23 — Refused or filtered?** On ubuntu2, listen on port 9000 with `nc -lk 9000`, and add
a rule that silently drops port 9001:
`sudo iptables -I INPUT -p tcp --dport 9001 -j DROP`. From ubuntu1, test 9000, 9001 and 9002.
Explain the three different results.
*Verify:* 9000 succeeds, 9002 is *refused* immediately, 9001 *times out*. Remove the rule after.

**10.24 — Read a handshake.** Capture a TCP connection from ubuntu1 to ubuntu2 port 22 with
`tcpdump`, and identify the SYN, SYN-ACK and ACK packets and the flags `tcpdump` prints for
each.
*Verify:* you can point at `[S]`, `[S.]` and `[.]` in your capture.

---

## Self-check

- [ ] I configure addresses, routes and DNS persistently on Netplan *and* NetworkManager
- [ ] I use `netplan try` before `netplan apply` on remote machines
- [ ] I know the difference between `getent hosts` and `dig`, and where real DNS servers are listed
- [ ] I can enable forwarding persistently and explain what it does
- [ ] I can build a bond and a bridge and check their state
- [ ] I can make a chrony server and client and read `chronyc sources`
- [ ] I troubleshoot bottom-up and can tell *refused* from *filtered*

➡ **Solutions:** [solutions/10-networking.md](../solutions/10-networking.md)
