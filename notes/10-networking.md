# Chapter 10 — Networking

Networking is a quarter of the exam and most of real-world troubleshooting. This chapter
covers configuring interfaces, addresses, routes, name resolution, bonds and bridges, and
time synchronisation — and a method for finding *where* a connection breaks instead of
guessing.

LFCS: *Networking* — configure IPv4 and IPv6 networking and hostname resolution; set and
synchronise time; monitor and troubleshoot networking; configure static routing; configure
bridge and bonding devices.

## Objectives

After this chapter you can:

- Read and change interfaces, addresses and routes with `ip`, temporarily and persistently.
- Configure static IPv4 and IPv6 addressing with Netplan (Ubuntu) and NetworkManager (RedHat).
- Set the hostname, and control name resolution through `/etc/hosts`, `nsswitch.conf` and
  `systemd-resolved`.
- Add static routes and enable forwarding.
- Build a bond and a bridge from two interfaces.
- Synchronise time with `chrony` and diagnose clock problems.
- Troubleshoot a connection layer by layer with `ip`, `ping`, `tracepath`, `ss`, `dig`, `nc`
  and `tcpdump`.

---

## 1. Interfaces and addresses — `ip`

`ip` replaced `ifconfig`, `route` and `arp`. Learn its object/verb pattern once:

```bash
ip link                       # interfaces, MAC, state
ip -br link                   # brief: one line each
ip addr                       # addresses (ip a)
ip -br addr
ip -4 addr show dev eth1
ip route                      # routing table (ip r)
ip -6 route
ip neigh                      # ARP / neighbour cache
ip -s link show eth0          # counters: errors and drops
```

**Changes made with `ip` are temporary** — gone at reboot or when the network service
reapplies its configuration. Use them to experiment and troubleshoot; make permanent
changes in the configuration system (sections 2 and 3).

```bash
sudo ip link set eth1 up
sudo ip addr add 10.10.10.5/24 dev eth1
sudo ip addr del 10.10.10.5/24 dev eth1
sudo ip route add 10.20.0.0/16 via 192.168.56.1
sudo ip route del 10.20.0.0/16
```

### Reading an address

```text
3: eth1: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP
    link/ether 08:00:27:3a:1b:9c brd ff:ff:ff:ff:ff:ff
    inet 192.168.56.11/24 brd 192.168.56.255 scope global eth1
    inet6 fe80::a00:27ff:fe3a:1b9c/64 scope link
```

`UP` means administratively up; **`LOWER_UP` means the cable/link is actually up**. `UP`
without `LOWER_UP` is a link problem. `fe80::` addresses are IPv6 link-local — every
IPv6 interface has one automatically; they don't route.

### Interface names

Modern names are predictable from hardware: `enp0s3` (PCI bus 0, slot 3), `ens160`,
`eth0` (on some VMs). Use the names `ip link` shows, never assume `eth0`.

---

## 2. Persistent configuration on Ubuntu — Netplan

Ubuntu configures the network with YAML in `/etc/netplan/*.yaml`, which Netplan renders for
a backend (`systemd-networkd` on servers, NetworkManager on desktops).

```yaml
# /etc/netplan/60-lab.yaml
network:
  version: 2
  ethernets:
    eth1:
      dhcp4: false
      addresses:
        - 192.168.56.11/24
        - "2001:db8:56::11/64"
      routes:
        - to: 10.20.0.0/16
          via: 192.168.56.1
      nameservers:
        addresses: [192.168.56.12, 1.1.1.1]
        search: [lab.local]
```

```bash
sudo chmod 600 /etc/netplan/60-lab.yaml   # netplan warns if others can read it
sudo netplan generate                      # validate and render, change nothing live
sudo netplan try                           # apply with automatic rollback after 120 s
sudo netplan apply                         # apply for real
networkctl status eth1                     # networkd's view
```

**`netplan try`** is the habit that saves remote sessions: if you lose connectivity and
can't confirm, it reverts by itself.

Files are merged in lexical order; a later file overrides an earlier one. YAML indentation
is significant and **tabs are invalid** — two spaces.

Vagrant writes its own file for the private network (usually `50-vagrant.yaml`). Read
what's there before adding to it: `cat /etc/netplan/*.yaml`.

---

## 3. Persistent configuration on RedHat — NetworkManager

```bash
nmcli device status
nmcli connection show
nmcli connection show "System eth1"          # every property
sudo nmcli connection modify "System eth1" \
  ipv4.method manual ipv4.addresses 192.168.56.13/24 ipv4.gateway 192.168.56.1 \
  ipv4.dns "192.168.56.12 1.1.1.1" ipv4.dns-search lab.local
sudo nmcli connection modify "System eth1" +ipv4.routes "10.20.0.0/16 192.168.56.1"
sudo nmcli connection modify "System eth1" ipv6.method manual ipv6.addresses 2001:db8:56::13/64
sudo nmcli connection up "System eth1"       # re-apply
```

`nmcli` changes are persistent (stored in `/etc/NetworkManager/system-connections/`), but
most only take effect when the connection is brought up again. `nmtui` is a text menu
version. `+property` adds to a list; plain `property` replaces it.

---

## 4. Hostname and name resolution

```bash
hostnamectl                                  # static hostname, OS, kernel
sudo hostnamectl set-hostname web01.lab.local
```

Also add the name to `/etc/hosts` so the machine can resolve itself without DNS.

### The resolution order

```text
getent hosts web01 ─▶ /etc/nsswitch.conf "hosts:" line ─▶ files (/etc/hosts) ─▶ dns (resolver)
```

```text
# /etc/nsswitch.conf
hosts:  files dns
```

`files` before `dns` is why an `/etc/hosts` entry overrides DNS — useful for testing,
dangerous when forgotten.

### Testing resolution properly

| Command | Uses nsswitch (hosts file)? | Use for |
|---|---|---|
| `getent hosts name` | **yes** | what applications will actually get |
| `dig name`, `host name`, `nslookup name` | no — DNS only | DNS server behaviour |
| `resolvectl query name` | via systemd-resolved | what the stub resolver returns |

```bash
dig web01.lab.local
dig +short example.com
dig @1.1.1.1 example.com          # ask a specific server
dig -x 192.168.56.12              # reverse lookup
dig example.com MX
dig AAAA example.com
```

### systemd-resolved

On Ubuntu, `/etc/resolv.conf` is a symlink to a **stub** that points at `127.0.0.53`,
a local caching resolver. The real upstream servers are elsewhere:

```bash
ls -l /etc/resolv.conf
resolvectl status                # upstream DNS per interface
resolvectl dns eth1 192.168.56.12   # temporary
sudo resolvectl flush-caches
```

Editing `/etc/resolv.conf` by hand on such a system is overwritten. Configure DNS through
Netplan (`nameservers:`), NetworkManager (`ipv4.dns`), or
`/etc/systemd/resolved.conf` (`DNS=`, `FallbackDNS=`, `Domains=`).

---

## 5. Routing

```bash
ip route
#  default via 10.0.2.2 dev eth0 proto dhcp metric 100
#  10.0.2.0/24 dev eth0 proto kernel scope link src 10.0.2.15
#  192.168.56.0/24 dev eth1 proto kernel scope link src 192.168.56.11
ip route get 8.8.8.8             # which route, interface and source a packet would use
```

The kernel picks the **most specific** matching route (longest prefix); `default` matches
everything else. When two routes are equally specific, the lower **metric** wins.

Static route, persistent:

```yaml
# Netplan
      routes:
        - to: 10.20.0.0/16
          via: 192.168.56.12
```

```bash
# NetworkManager
sudo nmcli connection modify "System eth1" +ipv4.routes "10.20.0.0/16 192.168.56.12"
```

### Making Linux a router

By default Linux drops packets that aren't addressed to it. To forward between
interfaces:

```bash
sudo sysctl -w net.ipv4.ip_forward=1                          # now
echo 'net.ipv4.ip_forward = 1' | sudo tee /etc/sysctl.d/90-forward.conf
sudo sysctl --system                                          # load all sysctl files
# IPv6: net.ipv6.conf.all.forwarding = 1
```

Forwarding plus NAT (chapter 11) is how a Linux box becomes a gateway.

---

## 6. IPv6 essentials

| | IPv4 | IPv6 |
|---|---|---|
| Size | 32 bits | 128 bits |
| Notation | `192.168.56.11/24` | `2001:db8:56::11/64` — `::` compresses one run of zero groups |
| Loopback | `127.0.0.1` | `::1` |
| Link-local | `169.254.0.0/16` (rare) | `fe80::/10` — always present |
| Private | `10/8`, `172.16/12`, `192.168/16` | `fd00::/8` (unique local) |
| Docs/examples | `192.0.2.0/24` | `2001:db8::/32` |
| Address resolution | ARP | Neighbour Discovery (`ip -6 neigh`) |
| Auto-configuration | DHCP | SLAAC (router advertisements) or DHCPv6 |

```bash
ping -6 ::1
ping -6 2001:db8:56::12
ping -6 fe80::a00:27ff:fe3a:1b9c%eth1     # link-local needs the interface (zone)
ip -6 addr; ip -6 route
```

In URLs and some tools an IPv6 address goes in brackets: `http://[2001:db8::1]:8080/`.

---

## 7. Bonding and bridging

### Bonding — two NICs acting as one

A bond aggregates interfaces for redundancy and/or bandwidth.

| Mode | Name | Needs switch support | Gives |
|---|---|---|---|
| 0 | `balance-rr` | yes (static) | throughput, failover |
| 1 | `active-backup` | **no** | failover only — the safe default |
| 2 | `balance-xor` | yes | throughput, failover |
| 4 | `802.3ad` (LACP) | **yes** (LACP) | throughput, failover — standard in data centres |
| 5 / 6 | `balance-tlb` / `balance-alb` | no | adaptive load balancing |

Netplan:

```yaml
network:
  version: 2
  ethernets:
    eth2: {}
    eth3: {}
  bonds:
    bond0:
      interfaces: [eth2, eth3]
      addresses: [10.99.0.11/24]
      parameters:
        mode: active-backup
        mii-monitor-interval: 100
        primary: eth2
```

```bash
cat /proc/net/bonding/bond0        # mode, active slave, each member's link state
```

NetworkManager:

```bash
sudo nmcli connection add type bond con-name bond0 ifname bond0 \
  bond.options "mode=active-backup,miimon=100" ipv4.method manual ipv4.addresses 10.99.0.13/24
sudo nmcli connection add type ethernet slave-type bond con-name bond0-eth2 ifname eth2 master bond0
sudo nmcli connection add type ethernet slave-type bond con-name bond0-eth3 ifname eth3 master bond0
sudo nmcli connection up bond0
```

### Bridging — a virtual switch

A bridge connects interfaces at layer 2, like a switch. VMs and containers attach to
bridges to share a physical network (chapter 14).

```yaml
network:
  version: 2
  ethernets:
    eth2: {}
  bridges:
    br0:
      interfaces: [eth2]
      addresses: [10.98.0.11/24]
      parameters:
        stp: false
        forward-delay: 0
```

```bash
bridge link                     # ports attached to bridges
ip -br link show master br0
```

**The address moves to the bridge.** Once an interface is a bridge port, it carries no IP
of its own — configure the address on `br0`. Doing this on the interface you're connected
through drops your SSH session; use `netplan try` or a console.

---

## 8. Time synchronisation

Wrong time breaks TLS certificates, Kerberos and LDAP authentication, log correlation,
and cron. Two implementations are common: RHEL-family systems use **chrony**; Ubuntu
24.04 uses the simpler `systemd-timesyncd` by default, and newer Ubuntu releases are
moving to chrony. Learn chrony — it can both sync and serve time, and it's what a task
asking you to *configure a time server* will expect. Check which one your machine runs
with `systemctl status chrony systemd-timesyncd`.

```bash
timedatectl                          # time, zone, "System clock synchronized: yes"
sudo timedatectl set-timezone Europe/Bucharest
timedatectl list-timezones | grep Europe
```

chrony:

```bash
sudo apt install chrony              # replaces systemd-timesyncd
```

```text
# /etc/chrony/chrony.conf   (/etc/chrony.conf on RedHat)
server 192.168.56.12 iburst prefer
pool ntp.ubuntu.com iburst
makestep 1.0 3                       # step the clock if > 1 s off, during the first 3 updates
```

```bash
sudo systemctl restart chrony        # chronyd on RedHat
chronyc sources -v                   # servers, reachability, offset; ^* = current source
chronyc tracking                     # how far off we are, and drift
sudo chronyc makestep                # step now instead of slewing slowly
```

To make a machine a **time server** for others, add `allow 192.168.56.0/24` to its
chrony config and open UDP 123.

| Symptom | Check |
|---|---|
| `sources` all `^?` | no reply — firewall (UDP 123), DNS name of the server |
| Large offset, slowly shrinking | chrony is *slewing*; `makestep` to jump |
| `timedatectl` says not synchronised | which service runs? (`timesyncd` vs `chrony` — only one) |

---

## 9. Troubleshooting, layer by layer

Work **bottom up**. Each step proves the layers beneath it.

| # | Layer | Question | Commands |
|---|---|---|---|
| 1 | Link | Is the interface up, with a link? | `ip -br link` (`LOWER_UP`), `ip -s link` (errors) |
| 2 | Address | Do I have the right IP and prefix? | `ip -br addr` |
| 3 | Local network | Can I reach the gateway / neighbour? | `ping <gateway>`, `ip neigh` |
| 4 | Routing | Is there a route, and where does it fail? | `ip route get <dst>`, `tracepath <dst>`, `mtr <dst>` |
| 5 | Name | Does the name resolve — to the right address? | `getent hosts <name>`, `dig <name>` |
| 6 | Port | Is the service listening, and reachable? | on server: `ss -tlnp`; from client: `nc -zv <host> <port>` |
| 7 | Firewall | Is something filtering? | `nft list ruleset`, `ufw status`, `firewall-cmd --list-all` |
| 8 | Application | Does the service answer correctly? | `curl -v`, the service's logs |

### ss — sockets

```bash
ss -tlnp            # TCP listening, numeric, with process
ss -ulnp            # UDP
ss -tnp             # established TCP connections
ss -tan state time-wait | wc -l
ss -tnp 'dport = :443'
```

A service listening on `127.0.0.1:8080` is reachable only from the machine itself;
`0.0.0.0:8080` or `[::]:8080` means all addresses. That one detail explains many
"works locally, not remotely" problems.

### Other tools

```bash
ping -c 3 192.168.56.12
tracepath 8.8.8.8                   # route + path MTU, no root needed
mtr -rwc 10 8.8.8.8                 # traceroute + ping statistics
nc -zv ubuntu2 22                   # can I open a TCP connection?
nc -l 9000                          # listen — test a port with another nc
curl -v http://ubuntu2:8000/        # full HTTP exchange
sudo tcpdump -ni eth1 port 53       # see packets: here, DNS
sudo tcpdump -ni eth1 host 192.168.56.12 and tcp port 22 -c 20
sudo tcpdump -ni any -w capture.pcap    # save for Wireshark
```

`tcpdump -n` skips name lookups (faster, and doesn't generate DNS traffic of its own).
If packets leave but nothing returns, the problem is beyond this host. If they never
arrive, it's before it.

---

## 10. Gotchas

**`ip` changes vanish** at reboot or the next `netplan apply`. Persist in Netplan/nmcli.

**Tabs in Netplan YAML** — invalid. `netplan generate` catches it.

**Editing `/etc/resolv.conf` under systemd-resolved** — overwritten. Configure the source.

**`dig` ignores `/etc/hosts`.** Use `getent hosts` to see what applications see.

**Bridging or bonding the interface you're logged in through** drops your session — use
`netplan try`, or work on a different interface.

**A service bound to `127.0.0.1`** can't be reached remotely no matter what the firewall says.

**Two time services** (`timesyncd` and `chrony`) fighting — enable only one.

**Forwarding without persistence** — `sysctl -w` is lost at reboot.

---

## 11. Commands introduced

| Command | Purpose |
|---|---|
| `ip link/addr/route/neigh` | interfaces, addresses, routes, neighbours |
| `netplan generate/try/apply`, `networkctl` | Ubuntu configuration |
| `nmcli`, `nmtui` | NetworkManager |
| `hostnamectl` | hostname |
| `getent hosts`, `dig`, `host`, `resolvectl` | name resolution |
| `sysctl` | forwarding and other kernel network settings |
| `bridge`, `/proc/net/bonding/` | bridges and bonds |
| `timedatectl`, `chronyc` | time |
| `ping`, `tracepath`, `mtr`, `ss`, `nc`, `curl`, `tcpdump` | troubleshooting |
| `man netplan`, `man nmcli-examples`, `man ip-route`, `man chrony.conf`, `man resolved.conf` | offline references |

---

➡ **Next:** [quiz](../quizzes/10-networking.md) → [lab](../labs/10-networking.md)
