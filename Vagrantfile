# -*- mode: ruby -*-
# vi: set ft=ruby :
#
# Bocker sandbox.
#
# bocker runs as root and rewrites the host's network interfaces, routing
# table, firewall rules, cgroups and mount table. Never run it on a machine
# you care about. This VM is the sandbox: if bocker trashes something, throw
# it away with `vagrant destroy` and start over.
#
#   vagrant up                       # build the sandbox (needs libvirt or virtualbox)
#   vagrant ssh                      # log in
#   sudo bocker pull alpine 3.19     # ... and start experimenting
#   sudo /vagrant/test               # run the test suite
#
# Modernised from the original 2015 CentOS 7 box:
#   * Ubuntu 22.04, util-linux 2.37 (no more compiling unshare by hand)
#   * cgroup v2 instead of the removed cgroup v1 controllers
#   * Docker Registry v2 pull (the v1 API bocker used was shut down)
#   * base image unpacked from the Alpine mini rootfs, no Docker needed

BRIDGE_ADDR = "10.0.0.1/24"
BRIDGE_NET  = "10.0.0.0/24"

# Runs once, on the first `vagrant up` (re-run with `vagrant provision`).
setup = <<~'SHELL'
  set -euxo pipefail
  export DEBIAN_FRONTEND=noninteractive

  apt-get update -qq
  apt-get install -y -qq \
    btrfs-progs curl iproute2 iptables jq uuid-runtime ca-certificates

  # --- btrfs filesystem mounted at /var/bocker --------------------------------
  if ! mountpoint -q /var/bocker; then
    if [ ! -f /var/bocker.img ]; then
      fallocate -l 10G /var/bocker.img
      mkfs.btrfs -q /var/bocker.img
    fi
    mkdir -p /var/bocker
    grep -q '/var/bocker' /etc/fstab \
      || echo '/var/bocker.img /var/bocker btrfs loop,nofail 0 0' >> /etc/fstab
    mount /var/bocker
  fi

  # --- cgroup v2 subtree for bocker ------------------------------------------
  # Ubuntu 22.04 is cgroup v2 only. Give bocker its own subtree with the cpu
  # and memory controllers delegated so `bocker run` can apply limits.
  mkdir -p /sys/fs/cgroup/bocker
  echo '+cpu +memory' > /sys/fs/cgroup/bocker/cgroup.subtree_control || \
    echo "warning: could not delegate cpu/memory controllers" >&2

  # --- base image for `bocker init` ----------------------------------------
  # Upstream did `docker save centos | undocker -o base-image`. We just unpack
  # the Alpine mini rootfs (~3 MB, no Docker in the loop).
  if [ ! -d /root/base-image ]; then
    mkdir -p /root/base-image
    curl -fsSL https://dl-cdn.alpinelinux.org/alpine/v3.19/releases/x86_64/alpine-minirootfs-3.19.1-x86_64.tar.gz \
      | tar -xz -C /root/base-image
    echo 'alpine:3.19' > /root/base-image/img.source
  fi

  ln -sf /vagrant/bocker /usr/local/bin/bocker
  echo "Bocker sandbox ready.  Try:  sudo bocker pull alpine 3.19"
SHELL

# Runs on every boot: the bridge / NAT / forwarding state is not persistent.
netsetup = <<~SHELL
  set -euxo pipefail

  mountpoint -q /var/bocker || mount /var/bocker || true

  sysctl -q -w net.ipv4.ip_forward=1
  echo 'net.ipv4.ip_forward=1' > /etc/sysctl.d/99-bocker.conf

  ext_if="$(ip route show default | awk '/default/ {print $5; exit}')"

  if ! ip link show bridge0 >/dev/null 2>&1; then
    ip link add bridge0 type bridge
    ip addr add #{BRIDGE_ADDR} dev bridge0
    ip link set bridge0 up
  fi

  # Idempotent NAT + forwarding for traffic from the container bridge.
  iptables -t nat -C POSTROUTING -s #{BRIDGE_NET} ! -o bridge0 -j MASQUERADE 2>/dev/null \
    || iptables -t nat -A POSTROUTING -s #{BRIDGE_NET} ! -o bridge0 -j MASQUERADE
  iptables -C FORWARD -i bridge0 -o "$ext_if" -j ACCEPT 2>/dev/null \
    || iptables -A FORWARD -i bridge0 -o "$ext_if" -j ACCEPT
  iptables -C FORWARD -i "$ext_if" -o bridge0 -m state --state RELATED,ESTABLISHED -j ACCEPT 2>/dev/null \
    || iptables -A FORWARD -i "$ext_if" -o bridge0 -m state --state RELATED,ESTABLISHED -j ACCEPT
SHELL

Vagrant.configure("2") do |config|
  config.vm.box = "generic/ubuntu2204"
  config.vm.hostname = "bocker-sandbox"

  # vagrant-libvirt does not mount /vagrant on its own. virtiofs/9p/NFS all
  # need host-side cooperation that qemu:///session + SELinux make painful, so
  # use rsync: a one-way host->guest copy pushed on `vagrant up`. After editing
  # files on the host, run `vagrant rsync` (or `vagrant rsync-auto` in a spare
  # terminal for live push) to resync.
  config.vm.synced_folder ".", "/vagrant", type: "rsync",
    rsync__exclude: [".git/", ".vagrant/"]

  config.vm.provider "libvirt" do |lv|
    lv.memory = 2048
    lv.cpus = 2
  end

  config.vm.provider "virtualbox" do |vb|
    vb.memory = 2048
    vb.cpus = 2
  end

  config.vm.provision "shell", inline: setup
  config.vm.provision "shell", inline: netsetup, run: "always"
end
