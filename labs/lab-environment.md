# Lab environment

Three virtual machines on a private network. The two Ubuntu nodes match the exam
environment; the Rocky node exists for chapter 13, because SELinux is native there.

| Host | OS | IP | Spare disks | Used for |
|---|---|---|---|---|
| `ubuntu1` | Ubuntu | 192.168.56.11 | 2 × 5 GB | Main lab machine — most tasks |
| `ubuntu2` | Ubuntu | 192.168.56.12 | 1 × 5 GB | Second node: NFS/NBD server, SSH target, load-balancer backend |
| `rocky1` | Rocky Linux 9 | 192.168.56.13 | 1 × 5 GB | SELinux, and RedHat-family comparisons |

`ubuntu1` also gets two extra network cards on an isolated network for the bonding and
bridge tasks in chapter 10, and nested virtualization for the libvirt tasks in chapter 14.

Host requirements: **VirtualBox 7**, **Vagrant 2.4+**, about 7 GB free RAM and 35 GB of disk.

---

## Match the exam's Ubuntu version

The exam has run on Ubuntu since 2023. Check which release it uses now on the Linux
Foundation site, then set `UBUNTU_BOX` below to match — for example
`bento/ubuntu-22.04` or `bento/ubuntu-24.04`. The differences that matter (default
packages, Netplan details, `nftables` vs `iptables` front-ends) are small but real.

---

## Vagrantfile

```ruby
# -*- mode: ruby -*-
UBUNTU_BOX = "bento/ubuntu-24.04"   # set to the exam's release
ROCKY_BOX  = "bento/rockylinux-9"

Vagrant.configure("2") do |config|
  config.vm.box_check_update = false

  nodes = {
    "ubuntu1" => { box: UBUNTU_BOX, ip: "192.168.56.11", disks: 2, mem: 2560, main: true },
    "ubuntu2" => { box: UBUNTU_BOX, ip: "192.168.56.12", disks: 1, mem: 1536 },
    "rocky1"  => { box: ROCKY_BOX,  ip: "192.168.56.13", disks: 1, mem: 1536 },
  }

  nodes.each do |name, spec|
    config.vm.define name do |node|
      node.vm.box      = spec[:box]
      node.vm.hostname = name
      node.vm.network "private_network", ip: spec[:ip]

      (1..spec[:disks]).each do |i|
        node.vm.disk :disk, name: "spare#{i}", size: "5GB"
      end

      if spec[:main]
        # Two unconfigured NICs on an isolated network, for bonding and bridges (ch. 10)
        node.vm.network "private_network", auto_config: false, virtualbox__intnet: "bondnet"
        node.vm.network "private_network", auto_config: false, virtualbox__intnet: "bondnet"
      end

      node.vm.provider "virtualbox" do |vb|
        vb.name   = "lfcs-#{name}"
        vb.memory = spec[:mem]
        vb.cpus   = 2
        vb.customize ["modifyvm", :id, "--nested-hw-virt", "on"] if spec[:main]
      end

      node.vm.provision "shell", inline: <<~SHELL
        cat >> /etc/hosts <<'EOF'
        192.168.56.11 ubuntu1
        192.168.56.12 ubuntu2
        192.168.56.13 rocky1
        EOF
      SHELL
    end
  end
end
```

---

## Bring it up

```bash
export VAGRANT_EXPERIMENTAL="disks"          # PowerShell: $env:VAGRANT_EXPERIMENTAL="disks"
vagrant up
vagrant status
vagrant ssh ubuntu1
```

Check the extra hardware arrived:

```bash
lsblk                 # ubuntu1: two unpartitioned 5 GB disks
ip -br link           # ubuntu1: two extra interfaces, DOWN, no address
```

If the disks are missing, `VAGRANT_EXPERIMENTAL` wasn't set when the VM was created:
`vagrant destroy -f ubuntu1` and `vagrant up ubuntu1` with it set.

---

## Practise the exam's workflow

In the exam you start on a base host and `ssh` to the host each task names. Set up the
same habit now: from `ubuntu1`, reach the others by name.

```bash
# on ubuntu1, as vagrant
ssh-keygen -t ed25519 -N '' -f ~/.ssh/id_ed25519
ssh-copy-id vagrant@ubuntu2        # password: vagrant
ssh-copy-id vagrant@rocky1
ssh ubuntu2 hostname
```

From now on, treat `ubuntu1` as your "base" host and `ssh` to the others for tasks
that name them. Always `exit` back to base before going to the next host — the exam
does not support nested SSH sessions.

---

## Snapshots

```bash
vagrant snapshot save clean                 # once, right after setup
vagrant snapshot restore ubuntu1 clean      # whenever a lab breaks a machine
```

Restore between chapters so leftovers from one chapter don't confuse the next. Several
labs ask you to break a machine deliberately; snapshots make that safe.

---

## Windows hosts: Hyper-V and nested virtualization

If Docker Desktop or WSL 2 is installed, Windows runs Hyper-V underneath, and VirtualBox
then runs on top of it. Everything in this course still works **except** hardware-
accelerated virtualization inside `ubuntu1`: in chapter 14, libvirt will fall back to
software emulation (QEMU TCG). VMs still start — just slowly. Use small images (the
chapter uses CirrOS for exactly this reason) and you will not notice much.

Check from inside `ubuntu1`:

```bash
grep -cE 'vmx|svm' /proc/cpuinfo     # 0 = no hardware virtualization available
```

---

## Day-to-day commands

| Command | What it does |
|---|---|
| `vagrant ssh ubuntu1` | log in |
| `vagrant halt` | shut everything down |
| `vagrant up rocky1` | start one machine |
| `vagrant snapshot list` | list snapshots |
| `vagrant destroy -f` | delete everything and start over |

You are user `vagrant` with passwordless `sudo` on every node.
