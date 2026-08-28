# Bocker
Docker implemented in around 100 lines of bash.

  * [Sandbox (start here)](#sandbox-start-here)
  * [Prerequisites](#prerequisites)
  * [Example Usage](#example-usage)
  * [Functionality: Currently Implemented](#functionality-currently-implemented)
  * [Functionality: Not Yet Implemented](#functionality-not-yet-implemented)
  * [What changed from upstream](#what-changed-from-upstream)
  * [License](#license)

## Sandbox (start here)

Bocker runs as **root** and rewrites your network interfaces, routing table,
firewall rules, cgroups and mount table. **Do not run it on a machine you care
about** — the upstream author's warning stands: *"I can make no guarantees that
it won't trash your system"*.

Use the bundled VM. A container runtime (Docker, Podman, distrobox, toolbox) is
**not** a safe substitute: bocker manipulates the host kernel's netfilter,
bridges and cgroup hierarchy directly, so a privileged container would put your
real host at risk and an unprivileged one can't run bocker at all. A throwaway
VM is the right blast radius — a mistake costs you a `vagrant destroy`.

```sh
vagrant up                      # needs vagrant + libvirt (or VirtualBox)
vagrant ssh
sudo bocker pull alpine 3.19
sudo bocker run img_XXXXX /bin/echo 'Hello from bocker'
```

The repo is copied to `/vagrant` in the VM with rsync (one-way). After editing
`bocker` on the host, run `vagrant rsync` to push the changes — or keep
`vagrant rsync-auto` running in another terminal.

Run the test suite inside the VM:

```sh
sudo -i
cd /vagrant && ./test
```

Verified on a Fedora host with `vagrant` + `vagrant-libvirt` (qemu:///session):
all 8 tests pass.

## Prerequisites

If you are not using the Vagrant box, you need:

* btrfs-progs
* curl
* iproute2
* iptables
* jq
* uuid-runtime (`uuidgen`)
* util-linux >= 2.25.2  (any current distro is fine; 22.04 ships 2.37)
* coreutils >= 7.5
* A cgroup **v2** hierarchy (the default on every current distro)

Additionally your system needs:

* A btrfs filesystem mounted under `/var/bocker`
* A cgroup v2 subtree at `/sys/fs/cgroup/bocker` with the `cpu` and `memory`
  controllers delegated
* A network bridge called `bridge0` with an IP of `10.0.0.1/24`
* IP forwarding enabled in `/proc/sys/net/ipv4/ip_forward`
* A firewall rule masquerading traffic from `10.0.0.0/24` onto a physical
  interface

The included `Vagrantfile` builds all of this on Ubuntu 22.04.

## Example Usage

```
$ bocker pull alpine 3.19
#################################################################### 100.0%
Created: img_5514

$ bocker images
IMAGE_ID        SOURCE
img_5514        alpine:3.19

$ bocker run img_5514 cat /etc/alpine-release
3.19.1

$ bocker ps
CONTAINER_ID       COMMAND
ps_5514            cat /etc/alpine-release

$ bocker logs ps_5514
3.19.1

$ bocker rm ps_5514
Removed: ps_5514

$ bocker run img_5514 which curl
which: no curl in (/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin)

$ bocker run img_5514 apk add --no-cache curl
(1/5) Installing ca-certificates ...
(5/5) Installing curl ...
OK: 12 MiB in 19 packages

$ bocker ps
CONTAINER_ID       COMMAND
ps_5502            apk add --no-cache curl
ps_5418            which curl

$ bocker commit ps_5502 img_5514
Removed: img_5514
Created: img_5514

$ bocker run img_5514 which curl
/usr/bin/curl

$ bocker run img_5514 cat /proc/1/cgroup
0::/

$ cat /sys/fs/cgroup/bocker/ps_5188/cpu.weight
100

$ cat /sys/fs/cgroup/bocker/ps_5188/memory.max
512000000

$ BOCKER_CPU_WEIGHT=200 \
    BOCKER_MEM_LIMIT=1024 \
    bocker run img_5514 sleep 30 &

$ cat /sys/fs/cgroup/bocker/ps_5219/cpu.weight
200

$ cat /sys/fs/cgroup/bocker/ps_5219/memory.max
1024000000
```

## Functionality: Currently Implemented

* `docker build` †
* `docker pull`
* `docker images`
* `docker ps`
* `docker run`
* `docker exec`
* `docker logs`
* `docker commit`
* `docker rm` / `docker rmi`
* Networking
* Quota Support / CGroups

† `bocker init` provides a very limited implementation of `docker build`

## Functionality: Not Yet Implemented

* Data Volume Containers
* Data Volumes
* Port Forwarding

## What changed from upstream

The original 2015 project targeted CentOS 7 and has bit-rotted. This fork keeps
the ~100-line spirit but makes it run today:

* **VM**: CentOS 7 → Ubuntu 22.04. util-linux 2.37 ships in the distro, so the
  hand-compilation of `unshare` is gone.
* **cgroups**: cgroup v1 (`cpu,cpuacct,memory` via `libcgroup-tools`) →
  cgroup v2. `bocker run` now writes `cpu.weight` / `memory.max` under
  `/sys/fs/cgroup/bocker/<id>` directly. The `BOCKER_CPU_SHARE` env var is now
  `BOCKER_CPU_WEIGHT` (range 1–10000, default 100).
* **pull**: Docker Registry **v1** API (shut down years ago) → Registry **v2**
  with token auth and multi-arch manifest handling. Requires `jq`.
* **base image**: `docker save centos | undocker` → the Alpine mini rootfs is
  unpacked straight into `~/base-image`, no Docker in the loop.
* **firewall**: no more `iptables --flush`; NAT/forward rules are added
  idempotently and scoped to `10.0.0.0/24`, and the external interface is
  detected from the default route instead of being hard-coded.
* **exec**: `bocker exec` now finds the container process with `ps -eo`
  (all processes) instead of `ps o`, which only lists the current session
  and made `exec` fail when bocker was driven non-interactively.

## License

Copyright (C) 2015 Peter Wilmott

This program is free software: you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

This program is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with this program.  If not, see <http://www.gnu.org/licenses/>.
