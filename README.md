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
- **your site** - your own private repository with one directory per node:
  `node.conf`, published ports, later the backup inventory and restore map.

`zk2-deploy`, run on your workstation, puts the code from zk2 and a node's
data from your site onto that node.

### Recommended: site next to zk2

```
~/infra/
  zk2-incus/    git clone https://github.com/zk2/zk2-incus
  my-infra/     your private repository
    zk2-site.conf
    nodes/
      node1/
        node.conf
        publication/
```

```sh
cd ~/infra
git clone https://github.com/zk2/zk2-incus
zk2-incus/tools/zk2-site init my-infra
cd my-infra && git init
../zk2-incus/tools/zk2-site add-node node1     # copies examples/node; edit it
../zk2-incus/tools/zk2-deploy --dry-run node1
../zk2-incus/tools/zk2-deploy node1
```

Updating: `git -C ~/infra/zk2-incus pull`, then `zk2-deploy` each node. zk2 never
sees your site: its `git status` stays clean, and nothing of yours can end
up in it by accident.

Do not put your site inside the zk2 checkout, and do not keep your nodes in
a fork of zk2: a fork of a public repository on GitHub is public too.

### Pinned version: zk2 as a submodule

If you want every node on a version you chose, and upgrades recorded in your
own history:

```
my-infra/
  zk2-incus/    git submodule -> github.com/zk2/zk2-incus
  zk2-site.conf
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
from, and finds your site by `zk2-site.conf` in the current directory or
above (or `ZK2_SITE`).

### On the node

| Path | What | Written by |
|---|---|---|
| `/opt/zk2/bin/` | zk2 commands | `zk2-deploy`, replaced as a whole |
| `/opt/zk2/VERSION` | which zk2 is deployed | `zk2-deploy` |
| `/usr/local/sbin/zk2-*` | links to `/opt/zk2/bin`, so commands are in `PATH` | `zk2-deploy` |
| `/etc/zk2/node.conf` | node parameters | `zk2-deploy`, from your site |
| `/etc/zk2/publication/` | published ports and ACLs | `zk2-deploy`, mirrored from your site |
| `/etc/zk2/zk2-env` | secrets: borg passphrase, mail credentials | **you**, by hand, mode 600; never in git, never overwritten |

Nightly report on the node, mailed to `ZK2_EMAIL` from `zk2-env` through
any of sendmail, msmtp or ssmtp. By default it comes every night, "OK" or
"findings" in the subject, so a missing mail is a signal too;
`MAIL_WHEN=problems` in `node.conf` limits it to nights with findings.

```
# /etc/cron.d/zk2
30 5 * * * root /usr/local/sbin/zk2-health --mail
```

Log in as root over ssh; the node needs `rsync`. The ssh alias is the node
directory name unless `SSH_HOST` in `node.conf` says otherwise.

## What is there

| Command | Where | What |
|---|---|---|
| `zk2-deploy` | workstation | deliver code and node data; deploy a container role |
| `zk2-site` | workstation | create a site, add a node from the example |
| `zk2-health` | node | what differs from how the node is meant to be: kernel without ZFS module, pool devices, memory limits and OOM kills, instances without limits, certificates, publication drift |
| `zk2-publish` | node | published ports (Incus network forwards) and ACLs from git: `diff`, `apply`, `show` |
| `zk2-idmap` | node | container uid isolation and pipe usage per host uid |

## Planned layout

| Path | Contents |
|---|---|
| `bin/` | node tools, installed to `/opt/zk2/bin`: build, new container, publish, backup, restore, health, nightly run |
| `cloud-init/` | the base profile for new instances |
| `hooks/` | pre-backup database dumps (examples for PostgreSQL and MariaDB) |
| `containers/` | nginx front and OpenVPN gateway: configs and helper scripts |
| `zabbix/` | optional monitoring: agent installer, templates, dashboard as code |
| `tools/` | workstation tools: deploy to a node, check all vhosts, OpenVPN client control |
| `examples/node/` | a fictional node, the template for `zk2-site add-node`: `node.conf`, published ports, later backup inventory and restore map |
| `docs/` | design notes: backup, node disaster recovery, migrating from OpenVZ 7 |
| `contrib/` | optional provider-specific helpers |

Commands are named `zk2-<what>`, without an extension.

## Requirements

- Debian 13 on two disks (mirrored ZFS pool for Incus)
- Incus 6.x or newer
- borg 1.2+ and an ssh-reachable borg repository for backups
- An SMTP relay for the nightly report

## License

MIT, see [LICENSE](LICENSE).
