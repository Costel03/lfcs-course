# Solutions 14 — Virtual machines and containers

---

# Part A — Virtual machines

## 🟢 Warm-up

**14.1**

```bash
kvm-ok
ls -l /dev/kvm
grep -cE 'vmx|svm' /proc/cpuinfo
```

With VirtualBox's nested virtualization working, all three say yes. On a Windows host with
Hyper-V active, the count is `0` and `kvm-ok` reports *KVM acceleration can NOT be used* —
libvirt falls back to QEMU's software emulation.

**14.2**

```bash
virsh list --all
virsh pool-list --all
virsh net-list --all
```

**14.3**

```bash
virsh net-dumpxml default
```

## 🔵 Practical

**14.4**

```bash
cd /var/lib/libvirt/images
sudo qemu-img create -f qcow2 -F qcow2 -b cirros-0.6.2-x86_64-disk.img vm1.qcow2 1G
sudo virt-install --name vm1 --memory 256 --vcpus 1 \
  --disk path=/var/lib/libvirt/images/vm1.qcow2,format=qcow2 \
  --import --osinfo detect=on,require=off \
  --network network=default --graphics none --noautoconsole
virsh list
qemu-img info vm1.qcow2           # backing file: cirros-0.6.2-x86_64-disk.img
```

Without KVM, the first boot can take a few minutes.

**14.5**

```bash
virsh console vm1
# login: cirros   password: gocubsgo
ip addr show eth0
# Ctrl-]
virsh domifaddr vm1
```

**14.6**

```bash
virsh shutdown vm1
virsh list --all                  # wait for "shut off"
virsh start vm1
virsh autostart vm1
virsh dominfo vm1 | grep -i autostart
```

If `shutdown` seems to do nothing, the guest isn't handling the ACPI power button yet
(still booting, or no ACPI daemon). `destroy` is the fallback — the equivalent of holding
the power button.

**14.7**

```bash
virsh shutdown vm1
virsh setmaxmem vm1 512M --config
virsh setmem vm1 512M --config
virsh setvcpus vm1 2 --config --maximum
virsh setvcpus vm1 2 --config
virsh start vm1
virsh dominfo vm1
```

Raise the *maximum* first; the current value can't exceed it.

**14.8**

```bash
virsh snapshot-create-as vm1 before-change
virsh console vm1        # touch /home/cirros/marker; sync; Ctrl-]
virsh snapshot-revert vm1 before-change
virsh console vm1        # ls /home/cirros — marker is gone
```

**14.9**

```bash
virsh vol-create-as default data1.qcow2 1G --format qcow2
virsh attach-disk vm1 /var/lib/libvirt/images/data1.qcow2 vdb \
  --driver qemu --subdriver qcow2 --persistent
virsh domblklist vm1
```

`--persistent` updates the saved definition as well as the running VM; without it the disk
disappears at the next restart. (Some older libvirt versions want `--live --config` instead.)

**14.10**

```bash
sudo mkdir -p /srv/vmpool
virsh pool-define-as vmpool dir --target /srv/vmpool
virsh pool-build vmpool
virsh pool-start vmpool
virsh pool-autostart vmpool
virsh pool-list --all
```

**14.11**

```bash
virsh shutdown vm1     # cloning needs the source shut off
sudo virt-clone --original vm1 --name vm2 --auto-clone
virsh start vm1; virsh start vm2
virsh domiflist vm1; virsh domiflist vm2
```

## 🔴 Challenge

**14.12**

```bash
virsh domblklist vm2                       # note the disk path(s) first
virsh destroy vm2
virsh undefine vm2 --remove-all-storage
virsh list --all
ls /var/lib/libvirt/images/
```

Without `--remove-all-storage`, `undefine` deletes the definition and leaves the disk files
behind — taking up space, and easy to forget.

**14.13**

```bash
virsh shutdown vm1
virsh edit vm1
```

Add after `<name>vm1</name>`:

```xml
<description>LFCS lab VM</description>
```

and change the interface model line to:

```xml
<model type='e1000'/>
```

```bash
virsh start vm1
virsh dumpxml vm1 | grep -E 'description|model type'
```

`virsh edit` validates the XML when you save. Editing the files under `/etc/libvirt/qemu/`
directly doesn't take effect until libvirtd re-reads them — and isn't validated.

---

# Part B — Containers

## 🟢 Warm-up

**14.14**

```bash
docker info | grep -E 'Storage Driver|Cgroup Driver|Docker Root Dir'
```

**14.15**

```bash
docker run -d --name web -p 8080:80 nginx:1.27
curl -s localhost:8080 | head -3
docker logs web
docker top web
ps -ef | grep 'nginx: '                    # the same processes, seen from the host
docker inspect -f '{{.NetworkSettings.IPAddress}}' web
```

The container's nginx appears in the host's process list with ordinary host PIDs — the
container is a process, isolated rather than hidden.

## 🔵 Practical

**14.16**

```bash
docker exec -it web bash
echo changed > /usr/share/nginx/html/index.html
exit
curl -s localhost:8080                 # changed
docker rm -f web
docker run -d --name web -p 8080:80 nginx:1.27
curl -s localhost:8080 | head -3       # the original page
```

The change lived in the container's writable layer, which `docker rm` deleted.

**14.17**

```bash
sudo mkdir -p /srv/site && echo 'from the host' | sudo tee /srv/site/index.html
docker rm -f web
docker run -d --name web -p 8080:80 \
  -v /srv/site:/usr/share/nginx/html:ro \
  --restart unless-stopped --memory 128m --cpus 0.5 nginx:1.27
curl -s localhost:8080
echo 'edited' | sudo tee /srv/site/index.html && curl -s localhost:8080
docker inspect -f '{{.HostConfig.RestartPolicy.Name}} {{.HostConfig.Memory}} {{.HostConfig.NanoCpus}}' web
```

**14.18**

```bash
docker run -d --name pg -e POSTGRES_PASSWORD=secret -v pgdata:/var/lib/postgresql/data postgres:16
sleep 10
docker exec -it pg psql -U postgres -c 'CREATE TABLE keepme (id int);'
docker rm -f pg
docker run -d --name pg2 -e POSTGRES_PASSWORD=secret -v pgdata:/var/lib/postgresql/data postgres:16
sleep 5
docker exec -it pg2 psql -U postgres -c '\dt'
```

**14.19**

```bash
mkdir -p ~/hello && cd ~/hello
echo '<h1>built by me</h1>' > index.html
cat > Dockerfile <<'EOF'
FROM nginx:1.27-alpine
COPY index.html /usr/share/nginx/html/index.html
EXPOSE 80
EOF
docker build -t hello:1.0 .
docker run -d --name registry --restart unless-stopped -p 5000:5000 registry:2
docker tag hello:1.0 localhost:5000/hello:1.0
docker push localhost:5000/hello:1.0
curl -s localhost:5000/v2/_catalog
```

On ubuntu2:

```bash
sudo apt install -y docker.io
echo '{ "insecure-registries": ["ubuntu1:5000"] }' | sudo tee /etc/docker/daemon.json
sudo systemctl restart docker
sudo docker pull ubuntu1:5000/hello:1.0
```

Without the `insecure-registries` entry, the pull fails with *server gave HTTP response to
HTTPS client*: Docker only talks plain HTTP to `localhost` by default. In production the
registry gets a TLS certificate instead (chapter 11) — and that's where something like Zot
comes in.

Note the pull from ubuntu2 reaches port 5000 even if ubuntu1's ufw from chapter 11 is still active —
see 14.23.

**14.20**

```bash
sudo tee /etc/docker/daemon.json > /dev/null <<'EOF'
{
  "log-driver": "json-file",
  "log-opts": { "max-size": "10m", "max-file": "3" }
}
EOF
python3 -m json.tool /etc/docker/daemon.json > /dev/null && echo valid
sudo systemctl restart docker
docker run -d --name logtest nginx:1.27
docker inspect -f '{{.HostConfig.LogConfig}}' logtest
```

Existing containers keep the log configuration they were created with; only new ones pick
up the default.

**14.21**

```bash
grep "^$USER:" /etc/subuid /etc/subgid || \
  sudo usermod --add-subuids 100000-165535 --add-subgids 100000-165535 "$USER"
podman run -d --name rootless-web -p 8083:80 docker.io/library/nginx:1.27
podman ps
sudo podman ps                       # empty — root's containers are separate
ps -o user= -p "$(pgrep -f 'nginx: master' | head -1)"
curl -s localhost:8083 | head -3
```

Rootless containers map container UID 0 onto a range of unprivileged host UIDs from
`/etc/subuid`. If `newuidmap` is missing, install the `uidmap` package. Use the fully
qualified image name, or configure `unqualified-search-registries`.

## 🔴 Challenge

**14.22**

```bash
sudo mkdir -p /etc/containers/systemd
sudo tee /etc/containers/systemd/site.container > /dev/null <<'EOF'
[Unit]
Description=Static site in a container

[Container]
Image=docker.io/library/nginx:1.27
PublishPort=8082:80
Volume=/srv/site:/usr/share/nginx/html:ro

[Service]
Restart=always

[Install]
WantedBy=multi-user.target
EOF
sudo systemctl daemon-reload
sudo systemctl start site
systemctl status site
curl -s localhost:8082
sudo reboot
```

You don't `enable` a Quadlet unit: the generated `site.service` is created at every
`daemon-reload` (and boot), and the `[Install]` section makes the generator wire it into
`multi-user.target`. `/usr/lib/systemd/system-generators/podman-system-generator --dryrun`
shows what it generates — handy when nothing appears.

**14.23**

```bash
sudo ufw default deny incoming && sudo ufw allow OpenSSH && sudo ufw enable
# from ubuntu2
curl -s ubuntu1:8080            # still works
# ubuntu1
sudo iptables -t nat -L DOCKER -n | grep 8080      # Docker's own DNAT rule
docker rm -f web
docker run -d --name web -p 127.0.0.1:8080:80 -v /srv/site:/usr/share/nginx/html:ro nginx:1.27
# from ubuntu2: fails;  on ubuntu1: curl -s localhost:8080 works
```

A published port is DNATed in the `PREROUTING` hook and then *forwarded* to the container,
so it never passes through ufw's `INPUT` rules. Binding to `127.0.0.1` stops it listening on
external addresses at all. To filter published ports by source address, add rules to the
`DOCKER-USER` chain, which Docker evaluates before its own.

**14.24**

```bash
docker run -d --name crash --restart on-failure:3 alpine sh -c 'echo starting; exit 3'
sleep 10
docker inspect -f 'restarts={{.RestartCount}} exit={{.State.ExitCode}} status={{.State.Status}}' crash
docker logs crash                # four "starting" lines: the first run plus 3 restarts
docker events --since 1m --filter container=crash    # die/start events, if you want the timeline
```

---

## Quiz answers

| Q | Answer | Why |
|---|---|---|
| 1 | KVM: runs guest code on the CPU (kernel module); QEMU: emulates the machine's devices; libvirt: manages definitions, storage and networks | |
| 2 | `kvm-ok`, `/dev/kvm`, `grep -cE 'vmx\|svm' /proc/cpuinfo` | |
| 3 | `destroy` forces power-off; `undefine` deletes the VM's definition | |
| 4 | `virsh autostart vm1` | |
| 5 | Boots an existing disk image rather than running an installer from media | |
| 6 | At the next VM start | `--config` edits the saved definition |
| 7 | The VM disk is a thin overlay: the base stays pristine and can back many VMs | |
| 8 | No — it's NAT; use a bridged network (`--network bridge=br0`) | |
| 9 | Namespaces, cgroups, an overlay root filesystem (plus capabilities/seccomp/MAC) | |
| 10 | Containers share the host kernel; a VM runs its own | |
| 11 | No — the container's main process exited; nothing kept it running | |
| 12 | They're deleted with the writable layer; use a volume or bind mount | |
| 13 | Host port 8080 forwards to container port 80; `-p 127.0.0.1:8080:80` | |
| 14 | Members can start containers that mount the host's root filesystem and act as root | |
| 15 | `insecure-registries` in `/etc/docker/daemon.json` on ubuntu2 (or give the registry TLS) | |
| 16 | `--restart unless-stopped` | |
| 17 | Quadlet `.container` files in `/etc/containers/systemd/` | |
| 18 | Docker's DNAT/forward rules bypass ufw's INPUT chain | |
