# Lab 14 — Virtual machines and containers

24 tasks on **ubuntu1** (14.19 also uses **ubuntu2**). VMs inside a VM are slow, especially
without hardware acceleration — that's expected. Do the quiz first.

**Setup:**

```bash
sudo apt update
sudo apt install -y qemu-system-x86 libvirt-daemon-system libvirt-clients virtinst cpu-checker
sudo usermod -aG libvirt,kvm "$USER"
# log out and back in, then:
cd /var/lib/libvirt/images
sudo curl -fLO https://download.cirros-cloud.net/0.6.2/cirros-0.6.2-x86_64-disk.img
```

**Cleanup:** restore the `clean` snapshot of ubuntu1 (this chapter installs a lot).

---

# Part A — Virtual machines

## 🟢 Warm-up

**14.1** Check whether ubuntu1 can use KVM acceleration. Explain the answer for your host.
*Verify:* `kvm-ok` output, `ls -l /dev/kvm`, and the `vmx|svm` count agree.

**14.2** Show libvirt's VMs, storage pools and networks, including inactive ones.
*Verify:* three `virsh … --all` commands; the `default` network and pool exist.

**14.3** Show the definition of libvirt's default network: its bridge name, subnet and DHCP
range.
*Verify:* `virbr0`, `192.168.122.0/24`.

## 🔵 Practical

**14.4** Create a VM `vm1` (256 MiB RAM, 1 vCPU) from the CirrOS image using a thin qcow2
overlay disk, on the default network, with only a serial console.
*Verify:* `virsh list` shows `vm1` running; `qemu-img info vm1.qcow2` shows the backing file.

**14.5** Log in to `vm1` on its console, find its IP address from inside, then leave the console
without shutting it down. Confirm the address from the host.
*Verify:* `virsh domifaddr vm1` (or `virsh net-dhcp-leases default`) matches what the guest
reported.

**14.6** Shut `vm1` down gracefully, start it again, then make it start automatically with the
host.
*Verify:* `virsh dominfo vm1` shows `Autostart: enable`.

**14.7** Raise `vm1` to 512 MiB of memory and 2 vCPUs, persistently.
*Verify:* after a restart, `virsh dominfo vm1` shows 512 MiB and 2 CPUs.

**14.8** Take a snapshot `before-change`, create a file in the guest, revert to the snapshot,
and show the file is gone.
*Verify:* `virsh snapshot-list vm1`; the file no longer exists after the revert.

**14.9** Create a 1 GiB qcow2 volume `data1.qcow2` in the default pool and attach it to `vm1` as
`vdb`, persistently.
*Verify:* `virsh domblklist vm1` lists `vdb`; inside the guest, `lsblk` (or `cat /proc/partitions`)
shows `vdb`.

**14.10** Create a new directory-based storage pool `vmpool` at `/srv/vmpool` that starts with the
host.
*Verify:* `virsh pool-list --all` shows `vmpool` active and autostart `yes`.

**14.11** Clone `vm1` into `vm2` and start both.
*Verify:* both running; `virsh domiflist` shows different MAC addresses.

## 🔴 Challenge

**14.12 — Delete properly.** Remove `vm2` completely, including its disk, and prove nothing is
left behind.
*Verify:* `virsh list --all` has no `vm2`; its disk file no longer exists.

**14.13 — Edit the XML.** Using `virsh edit`, give `vm1` a description "LFCS lab VM" and change
its network interface model to `e1000`. Start it and confirm.
*Verify:* `virsh dumpxml vm1 | grep -E 'description|model type'`.

---

# Part B — Containers

**Setup:** `sudo apt install -y docker.io podman && sudo usermod -aG docker "$USER"` — log out and
in again.

## 🟢 Warm-up

**14.14** Show the Docker storage driver, cgroup driver and data directory.
*Verify:* values from `docker info`.

**14.15** Run `nginx:1.27` detached as `web`, publishing it on host port 8080. Show its logs,
its processes from the host's point of view, and its IP address.
*Verify:* `curl -s localhost:8080` returns HTML; `ps -ef | grep nginx` on the host shows the
container's nginx processes.

## 🔵 Practical

**14.16** Open a shell inside `web`, change `index.html`, and show the change with curl. Then
remove and re-create the container. Where did your change go?
*Verify:* the change is gone after re-creation.

**14.17** Run `web` again, this time serving `/srv/site` from the host (read-only), restarting
automatically unless stopped, limited to 128 MiB and half a CPU.
*Verify:* editing `/srv/site/index.html` on the host changes curl's output immediately;
`docker inspect web` shows the restart policy and limits.

**14.18** Run PostgreSQL with its data in a named volume `pgdata`. Create a table, remove the
container, start a new one with the same volume, and show the table still exists.
*Verify:* `psql` in the new container lists your table.

**14.19** Build an image `hello:1.0` from a Dockerfile serving your own `index.html`. Run a
private registry on ubuntu1 port 5000, push the image to it, and pull it on **ubuntu2**.
*Verify:* `curl -s ubuntu1:5000/v2/_catalog` lists `hello`; `docker images` on ubuntu2 shows
`ubuntu1:5000/hello:1.0`.

**14.20** Configure Docker on ubuntu1 so container logs rotate at 10 MB with 3 files kept.
Restart the daemon and prove a new container uses it.
*Verify:* `docker inspect -f '{{.HostConfig.LogConfig}}' <new container>` shows the options.

**14.21** Run nginx as a **rootless** Podman container as your normal user, published on 8083.
*Verify:* `podman ps` as you shows it; `sudo podman ps` doesn't; `ps -o user= -p <pid>` shows
your user.

## 🔴 Challenge

**14.22 — Container as a service.** Using Quadlet, run nginx (`/srv/site` mounted read-only)
as a **system** service `site.service` on port 8082 that starts at boot.
*Verify:* `systemctl status site` active; `curl -s localhost:8082`; survives a reboot.

**14.23 — The firewall that didn't.** Enable ufw on ubuntu1 with SSH allowed and nothing else.
From ubuntu2, show that `web` on 8080 is still reachable. Explain why, and republish it so that
only ubuntu1 itself can reach it.
*Verify:* before: `curl ubuntu1:8080` from ubuntu2 works; after: it fails, while
`curl localhost:8080` on ubuntu1 works.

**14.24 — Debug a dying container.** Run
`docker run -d --name crash --restart on-failure:3 alpine sh -c 'echo starting; exit 3'`.
Find out how many times it restarted, its exit code, and its output — without guessing.
*Verify:* `docker inspect` shows `RestartCount` 3 and `ExitCode` 3; `docker logs crash` shows
the four `starting` lines.

---

## Self-check

- [ ] I can create, console into, resize, snapshot, clone and delete a VM with `virsh`
- [ ] I know `destroy` stops and `undefine` deletes
- [ ] I can add storage pools and volumes and attach disks
- [ ] I can explain a container in terms of namespaces, cgroups and overlay layers
- [ ] I can run containers with ports, volumes, limits and restart policies
- [ ] I can build, tag and push an image to a private registry
- [ ] I can configure the engine and run a container as a systemd service

➡ **Solutions:** [solutions/14-vms-containers.md](../solutions/14-vms-containers.md)
