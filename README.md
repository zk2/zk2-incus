# zk2

A small toolkit for running containers on your own dedicated servers with
[Incus](https://linuxcontainers.org/incus/): one to a handful of nodes, no
Kubernetes, no Proxmox, plain shell and git.

The project and its commands are called zk2 (`zk2-deploy`, `zk2-publish`,
...); the repository is `zk2-incus`.

> **Status: work in progress.** The tools are being extracted from a private
> setup that runs production workloads on several nodes. Until the first
> release, expect missing pieces; see [What is there](#what-is-there).

## What it is for

- One node or a few, each a dedicated server you rent or own.
- Containers (and occasionally VMs) managed by Incus on a ZFS mirror.
- An nginx front and an OpenVPN gateway as ordinary containers on the node.
- Everything that matters - published ports, backup policy, restore map -
  described in git, and the node brought to match the description.

## Principles

- **Git is the source of truth.** Port forwards, ACLs, the backup inventory
  and the restore map are files; scripts compare the node with them
  (`diff`) before changing anything (`apply`).
- **No provider lock-in.** Backups go to any borg repository reachable over
  ssh; mail goes through any SMTP relay; the base OS is a stock Debian
  installed however your provider installs it.
- **A backup you never restored is a hope.** Restores are scripted, driven by a
  map, and rehearsed.
- **Generic code, specific data.** zk2 holds code only and knows nothing
  about your nodes; your nodes live in your own repository. Updating zk2 is
  a `git pull` that never conflicts with your data.
- **Every rule has a reason.** Comments explain why a check exists, usually
  because something broke without it.

## How it is used

Two repositories:

- **zk2** - this one, code only. You clone it and never edit it.
- **your fleet** - your own private repository with one directory per node:
  `node.conf`, published ports, later the backup inventory and restore map.

`zk2-deploy`, run on your workstation, puts the code from zk2 and a node's
data from your fleet onto that node.

### Recommended: fleet next to zk2

```
~/infra/
  zk2-incus/    git clone https://github.com/zk2/zk2-incus
  my-fleet/     your private repository
    zk2-fleet.conf
    nodes/
      node1/
        node.conf
        publication/
```

```sh
cd ~/infra
git clone https://github.com/zk2/zk2-incus
zk2-incus/tools/zk2-fleet init my-fleet
cd my-fleet && git init
../zk2-incus/tools/zk2-fleet add-node node1     # copies examples/node; edit it
../zk2-incus/tools/zk2-deploy --dry-run node1
../zk2-incus/tools/zk2-deploy node1
```

Updating: `git -C ~/infra/zk2-incus pull`, then `zk2-deploy` each node. zk2 never
sees your fleet: its `git status` stays clean, and nothing of yours can end
up in it by accident.

Do not put your fleet inside the zk2 checkout, and do not keep your nodes in
a fork of zk2: a fork of a public repository on GitHub is public too.

### Pinned version: zk2 as a submodule

If you want every node on a version you chose, and upgrades recorded in your
own history:

```
my-fleet/
  zk2-incus/    git submodule -> github.com/zk2/zk2-incus
  zk2-fleet.conf
  nodes/...
```

```sh
git submodule add https://github.com/zk2/zk2-incus
git -C zk2-incus checkout v0.1.0 && git add zk2-incus && git commit -m "zk2 v0.1.0"
zk2-incus/tools/zk2-deploy node1
```

Upgrading is a commit in your repository ("zk2 v0.1.0 -> v0.2.0"), and
rolling back is reverting it.

In both layouts `zk2-deploy` delivers the code of the zk2 checkout it is run
from, and finds your fleet by `zk2-fleet.conf` in the current directory or
above (or `ZK2_FLEET`).

### On the node

| Path | What | Written by |
|---|---|---|
| `/opt/zk2/bin/` | zk2 commands | `zk2-deploy`, replaced as a whole |
| `/opt/zk2/hooks/` | generic pre-backup hooks | `zk2-deploy`, replaced as a whole |
| `/opt/zk2/cloud-init/` | first boot template of new instances | `zk2-deploy`, replaced as a whole |
| `/opt/zk2/VERSION` | which zk2 is deployed | `zk2-deploy` |
| `/usr/local/sbin/zk2-*` | links to `/opt/zk2/bin`, so commands are in `PATH` | `zk2-deploy` |
| `/etc/zk2/node.conf` | node parameters | `zk2-deploy`, from your fleet |
| `/etc/zk2/publication/` | published ports and ACLs | `zk2-deploy`, mirrored from your fleet |
| `/etc/zk2/inventory.conf` | what is backed up, how often, with which hook | `zk2-deploy`, from your fleet |
| `/etc/zk2/restore.conf` | restore map: archive, name, address, wave | `zk2-deploy`, from your fleet |
| `/etc/zk2/hooks/` | your own pre-backup hooks | `zk2-deploy`, mirrored from your fleet |
| `/etc/zk2/zk2-env.dist` | template of the secrets file | `zk2-deploy` |
| `/etc/zk2/zk2-env` | secrets: borg repository and passphrase, report address, instance passwords | **you**, from the template, mode 600; never in git, never overwritten |

### A new node

Install Debian 13 the way your provider does it: the OS on its own small
RAID, the disks or partitions meant for instances left empty. Then:

```sh
apt install rsync                               # on the node
../zk2-incus/tools/zk2-fleet add-node node1     # in your fleet; edit node.conf:
                                                #   ZFS_DISKS, SSH_PORT, CT_NET, ...
../zk2-incus/tools/zk2-deploy node1
```

On the node:

```sh
zk2-node-build --dry-run && zk2-node-build      # ZFS mirror, Incus, network, firewall
cp /etc/zk2/zk2-env.dist /etc/zk2/zk2-env && chmod 600 /etc/zk2/zk2-env
editor /etc/zk2/zk2-env                         # BORG_*, ZK2_EMAIL, instance passwords
editor /etc/msmtprc                             # your SMTP relay
(set -a; . /etc/zk2/zk2-env; borg init --encryption=repokey-blake2)
zk2-base-profile                                # first boot setup of new instances
zk2-publish apply
zk2-new-ct                                      # first instance
```

zk2-node-build refuses disks that carry any signature unless `--force`, and
its firewall lets in nothing but ssh on `SSH_PORT`: change the sshd port and
`node.conf` together.

Keep the passphrase and `borg key export --paper` outside the node: without
them no archive can be read.

The nightly run backs up the instances, then the host, then checks health,
and mails one report to `ZK2_EMAIL` from `zk2-env` through any of sendmail,
msmtp or ssmtp. By default the mail comes every night, "ok" or "FAILED" in
the subject, so a missing mail is a signal too; `MAIL_WHEN=problems` in
`node.conf` limits it to nights with problems.

```
# /etc/cron.d/zk2
0 3 * * * root /usr/local/sbin/zk2-nightly
```

### On the day of a disaster

Build the node anew, deploy it, put `zk2-env` back (from your password
manager, or from the host archive of another copy), then:

```sh
zk2-archives                                    # what there is, how fresh
mkdir -p /root/restore && cd /root/restore
(set -a; . /etc/zk2/zk2-env; borg extract "::$(borg list --glob-archives '_host@*' --last 1 --format '{archive}')" var/lib/zk2/host-state)
zk2-restore wave 1 --config /root/restore/var/lib/zk2/host-state/instances.json
zk2-restore wave 2 --config /root/restore/var/lib/zk2/host-state/instances.json
```

`--config` gives the instances back their limits and root sizes, which live
in the Incus database and not in the archives.

Log in as root over ssh; the node needs `rsync`. The ssh alias is the node
directory name unless `SSH_HOST` in `node.conf` says otherwise.

## What is there

| Command | Where | What |
|---|---|---|
| `zk2-deploy` | workstation | deliver code and node data; deploy a container role |
| `zk2-fleet` | workstation | create a fleet, add a node from the example |
| `zk2-node-build` | node | build a node on fresh Debian 13: ZFS mirror, Incus from Zabbly, bridge, DHCP routes for the VPN, split DNS, host firewall |
| `zk2-new-ct` | node | new instance by a short dialog: limits, isolated uid range, fixed address, cloud-init, docker if asked |
| `zk2-base-profile` | node | the Incus profile base: user, passwords, ssh policy and keys, shell, packages - from `cloud-init/base.yaml` |
| `zk2-nightly` | node | backup, host backup and health in order, one report |
| `zk2-backup` | node | snapshot, read-only mount, borg: containers, VMs (files, or whole on the monthly policy), pre-backup database dumps, per-instance retention |
| `zk2-host-backup` | node | the node's own root file system and a plain-text dump of its Incus state |
| `zk2-restore` | node | restore by the map, in waves; one instance from a given archive; a probe under another name; limits from the host dump; whole VMs |
| `zk2-archives` | node | what the repository holds, what is stale or missing, how to restore |
| `zk2-health` | node | what differs from how the node is meant to be: kernel without ZFS module, pool devices, memory limits and OOM kills, instances without limits, certificates, VPN routes, split DNS, backup inventory, leftover snapshots, publication drift |
| `zk2-publish` | node | published ports (Incus network forwards) and ACLs from git: `diff`, `apply`, `show` |
| `zk2-idmap` | node | container uid isolation and pipe usage per host uid |

## Planned layout

| Path | Contents |
|---|---|
| `bin/` | node tools, installed to `/opt/zk2/bin`: build, new container, publish, backup, restore, health, nightly run |
| `cloud-init/` | the base profile for new instances |
| `hooks/` | pre-backup database dumps: PostgreSQL, MariaDB, PostgreSQL in docker, GitLab |
| `containers/` | nginx front and OpenVPN gateway: configs and helper scripts |
| `zabbix/` | optional monitoring: agent installer, templates, dashboard as code |
| `tools/` | workstation tools: deploy to a node, check all vhosts, OpenVPN client control |
| `examples/node/` | a fictional node, the template for `zk2-fleet add-node`: `node.conf`, published ports, later backup inventory and restore map |
| `docs/` | design notes: backup, node disaster recovery, migrating from OpenVZ 7 |
| `contrib/` | optional provider-specific helpers |

Commands are named `zk2-<what>`, without an extension.

## Requirements

- Debian 13 (kernel 6.12+), and two disks or partitions for the ZFS mirror
- Incus 7 (zk2-node-build installs it from the Zabbly repository)
- borg 1.2+ and an ssh-reachable borg repository for backups
- An SMTP relay for the nightly report

## License

MIT, see [LICENSE](LICENSE).
