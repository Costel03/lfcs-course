# Chapter 14 — Virtual machines and containers

Two ways to run many isolated workloads on one Linux host. A **virtual machine** emulates a
whole computer and runs its own kernel. A **container** is an ordinary process isolated by
the host kernel's namespaces and cgroups — the mechanisms from chapters 05 and 09. Both are
LFCS objectives, and the second is the foundation of everything you'll do with Kubernetes.

LFCS: *Operations Deployment* — manage virtual machines (libvirt); configure container
engines, create and manage containers.

## Objectives

After this chapter you can:

- Explain how KVM, QEMU and libvirt fit together, and check for hardware virtualization.
- Create, start, stop, snapshot, clone, resize and delete VMs with `virt-install` and `virsh`.
- Manage libvirt storage pools, volumes and networks.
- Explain what a container is in kernel terms, and how images, layers and registries work.
- Run, inspect, debug and remove containers with Docker or Podman.
- Persist container data, publish ports, set restart policies and resource limits.
- Build an image, push it to a private registry, and configure the engine itself.
- Run a container as a systemd service.

Work on **ubuntu1** (which has nested virtualization enabled).

---

# Part 1 — Virtual machines with libvirt

## 1. The stack

```text
virsh / virt-install / virt-manager      ← tools you use
        │
    libvirtd                              ← management daemon: VM definitions (XML), storage, networks
        │
      QEMU                                ← emulates the machine: disks, NICs, firmware
        │
      KVM  (/dev/kvm)                     ← kernel module: runs guest code directly on the CPU
```

KVM makes it fast; without it, QEMU emulates every instruction in software ("TCG") — it
works, many times slower.

```bash
grep -cE 'vmx|svm' /proc/cpuinfo       # >0: the CPU offers virtualization (Intel vmx / AMD svm)
ls -l /dev/kvm                          # present when the kvm module is loaded
sudo apt install cpu-checker && kvm-ok  # Ubuntu's verdict
```

Inside this lab's `ubuntu1` VM those checks depend on the host. On Windows with Hyper-V
active (WSL 2, Docker Desktop), VirtualBox can't pass virtualization through — the guests
here will run under TCG. Use small images and be patient.

### Install

```bash
sudo apt install -y qemu-system-x86 libvirt-daemon-system libvirt-clients virtinst
sudo usermod -aG libvirt,kvm "$USER"      # log out and in to use virsh without sudo
systemctl status libvirtd
virsh -c qemu:///system list --all
```

`qemu:///system` is the system-wide libvirt instance (VMs run as a service account, use the
system networks and storage). `qemu:///session` is a per-user instance. Being in the
`libvirt` group makes `virsh` default to `qemu:///system`.

---

## 2. Creating a VM

### From a ready-made disk image (fast)

CirrOS is a tiny test image — ideal for labs:

```bash
cd /var/lib/libvirt/images
sudo curl -fLO https://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img
sudo qemu-img create -f qcow2 -F qcow2 -b cirros-0.6.2-x86_64-disk.img vm1.qcow2 1G

sudo virt-install \
  --name vm1 \
  --memory 256 --vcpus 1 \
  --disk path=/var/lib/libvirt/images/vm1.qcow2,format=qcow2 \
  --import \
  --osinfo detect=on,require=off \
  --network network=default \
  --graphics none --noautoconsole
```

| Option | Meaning |
|---|---|
| `--import` | boot the existing disk; don't run an installer |
| `--cdrom file.iso` / `--location url` | install from media instead |
| `--osinfo` | OS type, for sensible defaults (`virt-install --osinfo list`); `detect=on,require=off` when unknown |
| `--network network=default` | attach to libvirt's NAT network; `bridge=br0` for a host bridge |
| `--graphics none` | serial console only — no VNC |
| `--noautoconsole` | return to the shell instead of attaching |

The `qemu-img create … -b` line makes `vm1.qcow2` a thin **overlay** on the downloaded
image: the base stays untouched, and each VM stores only its own changes.

### Console

```bash
virsh console vm1            # Enter to get a prompt; CirrOS login: cirros / gocubsgo
# leave with Ctrl-]
```

---

## 3. Managing VMs — virsh

```bash
virsh list --all                     # all VMs and their state
virsh start vm1
virsh shutdown vm1                   # graceful (ACPI) — the guest must respond
virsh destroy vm1                    # hard power-off — not a deletion
virsh reboot vm1
virsh suspend vm1 / resume vm1       # pause / continue in memory
virsh autostart vm1                  # start with the host  (--disable to undo)
virsh dominfo vm1                    # state, CPUs, memory, autostart
virsh domifaddr vm1                  # its IP addresses (from DHCP leases)
virsh dumpxml vm1                    # its full definition
virsh edit vm1                       # edit the definition (applies at next start)
virsh undefine vm1 --remove-all-storage    # delete the VM and its disks
```

Note the words: **`destroy` only pulls the plug.** `undefine` deletes the definition.

### Changing resources

```bash
virsh setmaxmem vm1 512M --config        # the ceiling (VM must be off for most changes)
virsh setmem vm1 512M --config
virsh setvcpus vm1 2 --config --maximum
virsh setvcpus vm1 2 --config
```

`--config` changes the saved definition (next boot); `--live` changes the running VM where
supported; both together do both.

### Snapshots

```bash
virsh snapshot-create-as vm1 clean --description "fresh install"
virsh snapshot-list vm1
virsh snapshot-revert vm1 clean
virsh snapshot-delete vm1 clean
```

(Internal snapshots need qcow2 disks.)

### Cloning

```bash
virsh shutdown vm1
sudo virt-clone --original vm1 --name vm2 --auto-clone
```

`virt-clone` copies the disks and gives the clone new MAC addresses and UUID.

---

## 4. Storage pools and volumes

```bash
virsh pool-list --all
virsh pool-info default                 # usually /var/lib/libvirt/images
virsh vol-list default
virsh vol-create-as default data1.qcow2 1G --format qcow2
virsh attach-disk vm1 /var/lib/libvirt/images/data1.qcow2 vdb \
      --driver qemu --subdriver qcow2 --persistent
virsh domblklist vm1
virsh detach-disk vm1 vdb --persistent
```

A new directory-backed pool:

```bash
sudo mkdir -p /srv/vmpool
virsh pool-define-as vmpool dir --target /srv/vmpool
virsh pool-build vmpool && virsh pool-start vmpool && virsh pool-autostart vmpool
```

`qemu-img` works on disk files directly:

```bash
qemu-img info vm1.qcow2                    # format, virtual vs actual size, backing file
qemu-img create -f qcow2 disk.qcow2 10G    # thin: grows as written
qemu-img resize disk.qcow2 +5G             # then grow the partition/filesystem inside the guest
qemu-img convert -f raw -O qcow2 in.img out.qcow2
```

---

## 5. Networks

```bash
virsh net-list --all
virsh net-dumpxml default        # 192.168.122.0/24, NAT, DHCP — bridge virbr0
virsh net-start default && virsh net-autostart default
virsh net-dhcp-leases default
```

| Network type | Guests are reachable from | Use |
|---|---|---|
| NAT (`default`) | the host only; guests reach out through the host | labs, desktops |
| Bridged (`--network bridge=br0`) | the whole LAN, like physical machines | servers |
| Isolated | other guests only | test networks |

Bridged networking uses the Linux bridge from chapter 10: the host's NIC becomes a bridge
port, and the guests' virtual NICs join the same bridge.

---

# Part 2 — Containers

## 6. What a container is

A container is a normal process — visible in the host's `ps` — that the kernel isolates:

| Mechanism | Gives the container |
|---|---|
| **Namespaces** — `pid`, `net`, `mnt`, `uts`, `ipc`, `user`, `cgroup` | its own PID 1, network stack, mount tree, hostname… |
| **cgroups** (chapter 05) | limits on CPU, memory, processes, I/O |
| **overlay filesystem** (chapter 09) | a root filesystem built from read-only image layers plus a writable layer |
| capabilities, seccomp, SELinux/AppArmor | restricted privileges |

```text
         VM                                   container
┌─────────────────────┐            ┌─────────────────────┐
│ app                 │            │ app                 │
│ guest libraries     │            │ image libraries     │
│ GUEST KERNEL        │            └──────────┬──────────┘
│ virtual hardware    │                       │ shares the HOST kernel
└─────────┬───────────┘            ┌──────────┴──────────┐
     hypervisor                     │ namespaces + cgroups│
     host kernel                    │ host kernel         │
```

Consequences: containers start in milliseconds and use little memory, but they share the
host kernel — you can't run a Windows container on Linux, and a kernel exploit escapes
every container.

### Images, layers, registries

An **image** is a stack of read-only layers plus metadata (default command, ports,
environment). A **container** is an image plus a writable layer and a running process. A
**registry** stores images: Docker Hub, `quay.io`, `ghcr.io`, or your own (Zot, Harbor,
`registry:2`).

Image names: `registry/namespace/name:tag` — `docker.io/library/nginx:1.27`. Without a
registry, Docker assumes Docker Hub. `latest` is just a tag name, not a promise of newness.

---

## 7. Engines — Docker and Podman

| | Docker | Podman |
|---|---|---|
| Architecture | `dockerd` daemon, clients talk to it | daemonless — each container is a child process |
| Rootless | possible, not default | the default for normal users |
| Default on | Ubuntu (`docker.io` package), most tutorials | RedHat family |
| CLI | `docker …` | `podman …` — almost the same commands |
| systemd integration | restart policies in the daemon | **Quadlet** — containers as systemd units |
| Engine config | `/etc/docker/daemon.json` | `/etc/containers/*.conf` (`registries.conf`, `containers.conf`, `storage.conf`) |

```bash
sudo apt install -y docker.io          # Ubuntu's Docker build
sudo systemctl enable --now docker
sudo usermod -aG docker "$USER"        # see the warning below
docker version
docker info                            # storage driver, cgroup driver, registries, root dir

sudo apt install -y podman             # can coexist with Docker
podman info
```

**Membership of the `docker` group is equivalent to root**: `docker run -v /:/host …` gives
full access to the host's filesystem. Grant it as you would sudo.

---

## 8. Running containers

```bash
docker pull nginx:1.27
docker images
docker run -d --name web -p 8080:80 nginx:1.27
docker ps                               # running   (-a: all, including stopped)
curl -s localhost:8080 | head -3
docker logs -f web                      # stdout/stderr of the container
docker exec -it web bash                # a shell inside (sh if bash isn't in the image)
docker inspect web                      # everything, as JSON
docker inspect -f '{{.NetworkSettings.IPAddress}}' web
docker top web                          # its processes
docker stats --no-stream                # CPU and memory per container
docker stop web && docker start web
docker rm -f web                        # stop and remove
docker rmi nginx:1.27                   # remove the image
docker system df                        # disk used by images, containers, volumes
docker system prune                     # remove stopped containers, unused networks, dangling images
```

| `docker run` option | Meaning |
|---|---|
| `-d` | detached (background) |
| `--name` | a name instead of a random one |
| `-p 8080:80` | host port 8080 → container port 80 (`-p 127.0.0.1:8080:80` for local only) |
| `-e KEY=value` | environment variable |
| `-v vol:/path` / `-v /host/dir:/path` | named volume / bind mount |
| `--restart unless-stopped` | restart on failure and at boot, unless you stopped it |
| `--memory 256m`, `--cpus 0.5` | cgroup limits |
| `--rm` | delete the container when it exits |
| `-it` | interactive terminal |
| `--network mynet` | attach to a user-defined network |

A container lives as long as its main process. `docker run ubuntu` exits immediately —
there's nothing to keep running. That's not a crash.

### Data

The container's writable layer is deleted with the container. Anything that must survive
goes in a **volume** or **bind mount**:

```bash
docker volume create dbdata
docker run -d --name db -e POSTGRES_PASSWORD=secret -v dbdata:/var/lib/postgresql/data postgres:16
docker volume ls; docker volume inspect dbdata      # lives under /var/lib/docker/volumes/

docker run -d --name site -p 8081:80 -v /srv/site:/usr/share/nginx/html:ro nginx:1.27
```

On SELinux hosts with Podman, add `:Z` to a bind mount (`-v /srv/site:/usr/share/nginx/html:Z`)
so the content is relabelled `container_file_t`; otherwise the container gets *Permission
denied*.

### Networks

```bash
docker network create appnet
docker run -d --name api --network appnet myapi
docker run -d --name proxy --network appnet -p 80:80 nginx
# inside "proxy", the hostname "api" resolves to the api container
```

User-defined networks give containers DNS by name; the default `bridge` network doesn't.

---

## 9. Building images

```dockerfile
# ~/hello/Dockerfile
FROM nginx:1.27-alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
```

```bash
cd ~/hello && echo '<h1>built by me</h1>' > index.html
docker build -t hello:1.0 .
docker run -d --name hello -p 8090:80 hello:1.0
docker history hello:1.0                 # the layers
```

| Instruction | Does |
|---|---|
| `FROM` | base image |
| `RUN` | run a command at build time — creates a layer |
| `COPY` / `ADD` | add files from the build context |
| `ENV`, `WORKDIR`, `USER` | environment, directory, user for later steps and at run time |
| `EXPOSE` | document the port (doesn't publish it) |
| `CMD` / `ENTRYPOINT` | what runs when the container starts |

### A private registry

```bash
docker run -d --name registry --restart unless-stopped -p 5000:5000 registry:2
docker tag hello:1.0 localhost:5000/hello:1.0
docker push localhost:5000/hello:1.0
curl -s localhost:5000/v2/_catalog
```

From another machine, a plain-HTTP registry must be declared insecure — engine
configuration, next section.

---

## 10. Configuring the engine

### Docker — /etc/docker/daemon.json

```json
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" },
  "insecure-registries": ["ubuntu1:5000"],
  "registry-mirrors": ["https://mirror.example.com"],
  "data-root": "/var/lib/docker",
  "live-restore": true
}
```

```bash
sudo systemctl restart docker
docker info | grep -iA2 -E 'insecure|logging driver|root dir'
```

Invalid JSON stops the daemon from starting — check with `python3 -m json.tool
/etc/docker/daemon.json` first. `log-opts` matter: the default json-file log has no size
limit, and a chatty container can fill `/var`.

### Podman — /etc/containers/

```toml
# /etc/containers/registries.conf.d/50-lab.conf
unqualified-search-registries = ["docker.io"]

[[registry]]
location = "ubuntu1:5000"
insecure = true
```

`unqualified-search-registries` decides where `podman pull nginx` looks; without it Podman
asks or refuses. Rootless Podman stores images in `~/.local/share/containers/`, separately
from root's.

---

## 11. Containers as services

### Docker — restart policies

`--restart unless-stopped` (or `always`) makes `dockerd` restart the container on failure
and when the daemon starts at boot. Enable `docker.service` and you're done.

### Podman — Quadlet

Podman has no daemon to restart things, so containers run as **systemd units**. Describe
the container in a `.container` file and systemd generates the service:

```ini
# /etc/containers/systemd/web.container
[Unit]
Description=nginx in a container

[Container]
Image=docker.io/library/nginx:1.27
PublishPort=8082:80
Volume=/srv/site:/usr/share/nginx/html:ro,Z

[Service]
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload
sudo systemctl start web            # the generated web.service
systemctl status web
```

For rootless containers the file goes in `~/.config/containers/systemd/`, with
`systemctl --user`, and `loginctl enable-linger $USER` so it runs without a login session.
(`podman generate systemd` did this before Quadlet and is deprecated.)

---

## 12. Gotchas

**`virsh destroy` is power-off, not delete.** Deleting is `undefine`.

**No KVM → TCG.** Works, slowly. Check `/dev/kvm` and `kvm-ok`.

**A container that exits immediately** has no long-running foreground process.

**Data written inside a container vanishes** with it — use volumes.

**Docker bypasses ufw.** Published ports are added straight to netfilter's NAT and forward
chains, *before* ufw's rules — a port published with `-p 8080:80` is reachable from the
network even if `ufw` denies 8080. Publish on `127.0.0.1:` for local-only services, or
filter in the `DOCKER-USER` chain.

**`docker` group = root.**

**Unbounded json-file logs** fill the disk — set `log-opts`.

**`latest` changes under you.** Pin tags (or digests) for anything you deploy.

**SELinux and bind mounts** — `:Z` with Podman on enforcing hosts.

---

## 13. Commands introduced

| Command | Purpose |
|---|---|
| `kvm-ok`, `/dev/kvm` | virtualization support |
| `virt-install`, `virt-clone` | create and clone VMs |
| `virsh list/start/shutdown/destroy/undefine/autostart/console/edit/dumpxml` | manage VMs |
| `virsh snapshot-*`, `setmem`, `setvcpus`, `attach-disk` | snapshots and resources |
| `virsh pool-*`, `vol-*`, `net-*` | storage and networks |
| `qemu-img` | disk images |
| `docker` / `podman` `pull/run/ps/logs/exec/inspect/stop/rm/rmi/build/tag/push` | containers |
| `docker volume`, `docker network` | data and networking |
| `/etc/docker/daemon.json`, `/etc/containers/registries.conf` | engine configuration |
| Quadlet `.container` files | containers as systemd services |
| `man virsh`, `man virt-install`, `man qemu-img`, `man docker-run`, `man podman-run`, `man podman-systemd.unit` (Quadlet) | offline references |

---

➡ **Next:** [quiz](../quizzes/14-vms-containers.md) → [lab](../labs/14-vms-containers.md)
