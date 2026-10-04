# zk2

A small toolkit for running containers on your own dedicated servers with
[Incus](https://linuxcontainers.org/incus/): one to a handful of nodes, no
Kubernetes, no Proxmox, plain shell and git.

> **Status: work in progress.** The tools are being extracted from a private
> setup that runs production workloads on several nodes. Until
> the first release, the layout below is the plan, not the contents.

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
- **Generic code, specific data.** Nothing under `bin/` knows about a
  particular node; all specifics live in a per-node directory.
- **Every rule has a reason.** Comments explain why a check exists, usually
  because something broke without it.

## Planned layout

| Path | Contents |
|---|---|
| `bin/` | node tools, installed to `/root/bin`: build, new container, publish, backup, restore, health, nightly run |
| `cloud-init/` | the base profile for new instances |
| `hooks/` | pre-backup database dumps (examples for PostgreSQL and MariaDB) |
| `containers/` | nginx front and OpenVPN gateway: configs and helper scripts |
| `zabbix/` | optional monitoring: agent installer, templates, dashboard as code |
| `tools/` | workstation tools: deploy to a node, check all vhosts, OpenVPN client control |
| `examples/node/` | a fictional node: `node.conf`, backup inventory, restore map, published ports |
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
