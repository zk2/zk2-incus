# zk2

A small toolkit for running containers on your own dedicated servers with
[Incus](https://linuxcontainers.org/incus/): one to a handful of nodes, no
Kubernetes, no Proxmox, plain shell and git.

The project and its commands are called zk2 (`zk2-deploy`, `zk2-publish`,
...); the repository is `zk2-incus`.

> **Status: v0.1.0, the first release.** The tools come from a private setup
> that runs production workloads on several nodes. Every step below was run
> on real nodes (Debian 13, Incus 7.0.1, ZFS 2.3.9, borg 1.4), including a
> node rebuilt from nothing and restored from its archives. See
> [CHANGELOG.md](CHANGELOG.md) and [Limitations](#limitations).

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
  `node.conf`, published ports, the backup inventory, the restore map, the
  vhosts of its front, and the public keys its peers need.

`zk2-deploy`, run on your workstation, puts the code from zk2 and a node's
data from your fleet onto that node. `zk2-deploy NODE ROLE` puts the files
of a container role (nginx, openvpn) into its instance.

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
        inventory.conf
        restore.conf
```

```sh
cd ~/infra
git clone https://github.com/zk2/zk2-incus
zk2-incus/tools/zk2-fleet init my-fleet
cd my-fleet && git init
../zk2-incus/tools/zk2-fleet add-node node1     # copies examples/node; edit it
```

zk2 never sees your fleet: its `git status` stays clean, and nothing of
yours can end up in it by accident. Do not put your fleet inside the zk2
checkout, and do not keep your nodes in a fork of zk2: a fork of a public
repository on GitHub is public too.

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
| `/etc/zk2/peers/` | the other nodes of the fleet: addresses, keys, certificates | `zk2-deploy`, built from your fleet |
| `/etc/zk2/zk2-env.dist` | template of the secrets file | `zk2-deploy` |
| `/etc/zk2/zk2-env` | secrets: borg repository and passphrase, report address, instance passwords | **you**, from the template, mode 600; never in git, never overwritten |

Every setting of `node.conf` is listed, with its default, in
[examples/node/node.conf](examples/node/node.conf).

## A new node

### 1. The machine

Install Debian 13 the way your provider does it: the OS on its own small
RAID, the disks or partitions meant for instances left empty. Turn **Secure
Boot off**: the ZFS module is built by dkms and signed with a key the
firmware does not know, so the kernel would refuse it (zk2-node-build checks
and stops before changing anything). On the node: `apt install rsync`.

### 2. Access from the workstation

zk2-deploy logs in as root over ssh, by the node's directory name unless
`SSH_HOST` in `node.conf` says otherwise:

```
# ~/.ssh/config
Host node1
    HostName 203.0.113.10
    Port 22
    User root
    IdentityFile ~/.ssh/id_ed25519
    IdentitiesOnly yes
```

`ssh-copy-id node1`, then `ssh node1 true` must pass without a password.

### 3. The fleet entry and the build

In your fleet, edit `nodes/node1/node.conf` (`PUBLIC_IP`, `SSH_PORT`,
`ZFS_DISKS`, `CT_NET`, ...), commit, then:

```sh
../zk2-incus/tools/zk2-deploy --dry-run node1
../zk2-incus/tools/zk2-deploy node1
ssh node1 'zk2-node-build --dry-run && zk2-node-build'   # ZFS mirror, Incus, network, firewall
```

zk2-node-build refuses disks that carry any signature unless `--force`, and
its firewall lets in nothing but ssh on `SSH_PORT`: change the sshd port and
`node.conf` together. It is idempotent: run it again after a failure.

### 4. The backup repository

Any borg repository reachable over ssh: a server of yours, a storage box,
BorgBase, rsync.net. On the node, a key of its own:

```sh
ssh-keygen -t ed25519 -N '' -f /root/.ssh/id_ed25519_borg -C "borg@$(hostname)"
cat /root/.ssh/id_ed25519_borg.pub
```

Authorize it on the repository side. On a server of yours, a dedicated user
with this line in its `authorized_keys` lets the key run borg in one
directory and nothing else:

```
command="borg serve --restrict-to-path /srv/borg/node1",restrict ssh-ed25519 AAAA... borg@node1
```

Hosted services have their own way to add a key.

### 5. Secrets, mail, repository

On the node:

```sh
cp /etc/zk2/zk2-env.dist /etc/zk2/zk2-env && chmod 600 /etc/zk2/zk2-env
editor /etc/zk2/zk2-env           # BORG_REPO, BORG_RSH, BORG_PASSPHRASE, ZK2_EMAIL, instance passwords
editor /etc/msmtprc               # your SMTP relay (msmtp-mta is installed by zk2-node-build)
(set -a; . /etc/zk2/zk2-env; borg init --encryption=repokey-blake2)
```

Keep the passphrase and `borg key export --paper` outside the node: without
them no archive can be read.

### 6. Instances, front, VPN

```sh
zk2-base-profile                  # first boot setup of new instances (user, ssh policy, shell)
zk2-publish apply                 # published ports from the fleet
zk2-new-ct                        # an instance, by a short dialog
```

The front and the VPN are ordinary instances with roles of their own:
[containers/nginx/README.md](containers/nginx/README.md) and
[containers/openvpn/README.md](containers/openvpn/README.md).

### 7. The nightly run

It backs up the instances, then the host, then checks health, and mails one
report to `ZK2_EMAIL`. By default the mail comes every night, "ok" or
"FAILED" in the subject, so a missing mail is a signal too;
`MAIL_WHEN=problems` in `node.conf` limits it to nights with problems.

```
# /etc/cron.d/zk2
0 3 * * * root /usr/local/sbin/zk2-nightly
```

Run `zk2-nightly --no-mail` once by hand, and `zk2-archives` after it.

## Day to day

All changes start in the fleet and reach the node with `zk2-deploy`.

- **A new instance:** `zk2-new-ct` on the node, then in the fleet a line in
  `inventory.conf` (backup policy - zk2-health reports an instance missing
  from it) and in `restore.conf` (its address and wave).
- **A published port:** edit `publication/forward.yaml`, `zk2-deploy NODE`,
  then `zk2-publish diff` and `zk2-publish apply` on the node.
- **A new site on the front:** see [containers/nginx](containers/nginx/README.md#a-new-site).
- **A VPN client:** `incus exec openvpn -- zk2-vpn-add NAME`, see
  [containers/openvpn](containers/openvpn/README.md#clients).
- **Updating zk2:** `git -C zk2-incus pull` (or bump the submodule), then
  `zk2-deploy` each node; `/opt/zk2/VERSION` and the nightly report show
  what a node runs.
- **What is going on:** `zk2-list`, `zk2-health`, `zk2-archives`.

## More than one node

Nodes of one fleet can reach each other: root over ssh, the Incus API for
`incus copy`, and `zk2-send` to deploy an instance from one node's archive
on another. Give every node its own `CT_NET`.

```sh
# on the workstation, in the fleet: public keys and certificates of each node
../zk2-incus/tools/zk2-fleet collect node1
../zk2-incus/tools/zk2-fleet collect node2
git add nodes && git commit -m "peers"
for n in node1 node2; do ../zk2-incus/tools/zk2-deploy $n; done

# on each node
zk2-ssh-config apply              # ssh PEER works, host keys checked from the fleet
zk2-incus-peers apply             # twice on each node the first time: the first
                                  # pass hands out trust, the second adds remotes

# then, for example
zk2-send web node2                # node2 needs a line for web in its restore.conf
```

A node trusted this way gets root on its peers. Leave a node out of the
fleet, or its key out of `nodes/NODE/ssh`, to keep it apart.

## On the day of a disaster

What you need: the borg repository, your fleet repository, a zk2 checkout,
and the borg passphrase from your password manager. Everything else - the
real `zk2-env`, the limits of every instance - comes back from the archives.
The whole sequence below was rehearsed on a test node rebuilt from nothing.

1. Build the node anew: [A new node](#a-new-node), steps 1 to 3.

2. A new borg key. The old one died with the node; authorize the new one on
   the repository side (step 4) and drop the old line.

3. A temporary `zk2-env` of three lines - enough to read the repository:

   ```sh
   cat > /etc/zk2/zk2-env <<'EOF'
   BORG_REPO=ssh://...
   BORG_RSH="ssh -i /root/.ssh/id_ed25519_borg -o IdentitiesOnly=yes"
   BORG_PASSPHRASE="..."
   EOF
   chmod 600 /etc/zk2/zk2-env
   zk2-archives          # what there is, how fresh; confirm the repository's host key
   ```

4. The real `zk2-env` and the state dump from the newest host archive:

   ```sh
   mkdir -p /root/restore && cd /root/restore
   (set -a; . /etc/zk2/zk2-env
    borg extract "::$(borg list --glob-archives '_host@*' --last 1 --format '{archive}')" \
        etc/zk2/zk2-env var/lib/zk2/host-state)
   cp etc/zk2/zk2-env /etc/zk2/zk2-env && chmod 600 /etc/zk2/zk2-env
   ```

5. Profile, published ports, then the instances wave by wave:

   ```sh
   zk2-base-profile
   zk2-publish apply
   zk2-restore wave 1 --config /root/restore/var/lib/zk2/host-state/instances.json
   zk2-restore wave 2 --config /root/restore/var/lib/zk2/host-state/instances.json
   ```

   `--config` gives the instances back their limits and root sizes, which
   live in the Incus database and not in the archives.

6. Check and carry on: `zk2-health`, then `zk2-nightly --no-mail` - new
   archives go into the same repository, and the history continues.

7. With peers: the rebuilt node has new keys and certificates.
   `zk2-fleet collect NODE`, commit, `zk2-deploy` every node of the fleet,
   then `zk2-ssh-config apply` and `zk2-incus-peers apply` on each.

## What is there

| Command | Where | What |
|---|---|---|
| `zk2-fleet` | workstation | create a fleet, add a node from the example, collect a node's public keys and certificates for its peers |
| `zk2-deploy` | workstation | deliver code and node data; deploy a container role |
| `zk2-node-build` | node | build a node on fresh Debian 13: ZFS mirror, Incus from Zabbly, bridge, DHCP routes for the VPN, split DNS, host firewall |
| `zk2-base-profile` | node | the Incus profile base: user, passwords, ssh policy and keys, shell, packages - from `cloud-init/base.yaml` |
| `zk2-new-ct` | node | new instance by a short dialog: limits, isolated uid range, fixed address, cloud-init, docker if asked |
| `zk2-publish` | node | published ports (Incus network forwards) and ACLs from git: `diff`, `apply`, `show` |
| `zk2-nightly` | node | backup, host backup and health in order, one report |
| `zk2-backup` | node | snapshot, read-only mount, borg: containers, VMs (files, or whole on the monthly policy), pre-backup database dumps, per-instance retention |
| `zk2-host-backup` | node | the node's own root file system and a plain-text dump of its Incus state |
| `zk2-archives` | node | what the repository holds, what is stale or missing, how to restore |
| `zk2-restore` | node | restore by the map, in waves; one instance from a given archive; a probe under another name; limits from the host dump; whole VMs; a stream from a peer |
| `zk2-export` | node | a one-off snapshot of a whole instance (incus export into borg): save, list, check by importing, load |
| `zk2-health` | node | what differs from how the node is meant to be: kernel without ZFS module, pool devices, memory limits and OOM kills, instances without limits, certificates, VPN routes, split DNS, backup inventory, leftover snapshots, publication drift |
| `zk2-list` | node | table of instances: backup policy, address, memory and disk against their limits, swap, nesting, snapshots, OS; `--deep` adds failed units and services whose executable is missing |
| `zk2-ssh-config` | node | root's ssh to the peer nodes (aliases, host keys from the fleet, authorized keys) and a `borg` alias |
| `zk2-incus-peers` | node | the Incus API between the nodes: firewall set, listener, trust, remotes with checked fingerprints |
| `zk2-send` | node | deploy an instance from this node's archive onto a peer node, with its configuration |
| `zk2-ssh-policy` | node | the ssh policy of node.conf (password login, root login) in existing instances: drop-in or in-place edit, cloud-init stub replaced, verified with sshd -t and sshd -T |
| `zk2-shell` | node | the zk2 shell setup (history, prompt) in existing instances, and no foreign PS1 overriding it; `--check` only looks |
| `zk2-idmap` | node | container uid isolation and pipe usage per host uid |
| `zk2-vpn-setup`, `-add`, `-revoke`, `-show` | openvpn instance | the OpenVPN gateway, see [containers/openvpn](containers/openvpn/README.md) |
| `conf_generator` | workstation | a new nginx vhost into the fleet, see [containers/nginx](containers/nginx/README.md) |

Every command answers `--help`.

## Layout

| Path | Contents |
|---|---|
| `bin/` | node tools, installed to `/opt/zk2/bin` |
| `tools/` | workstation tools: `zk2-fleet`, `zk2-deploy` |
| `cloud-init/` | the first boot template of new instances |
| `hooks/` | pre-backup database dumps: PostgreSQL, MariaDB, PostgreSQL in docker, GitLab ([hooks/README.md](hooks/README.md)) |
| `containers/nginx/` | the front: nginx.conf, snippets, vhost templates and generator |
| `containers/openvpn/` | the VPN gateway scripts |
| `examples/node/` | a fictional node, the template for `zk2-fleet add-node` |
| `examples/zk2-env.dist` | the secrets template |

Planned: optional monitoring with Zabbix (agent, templates, dashboard as
code), more workstation tools (vhost check, VPN client control, SMART
survey), design notes, and a guide for moving containers from OpenVZ 7.

## Requirements

- **Nodes:** Debian 13 (kernel 6.12+) with Secure Boot off, two disks or
  partitions for the ZFS mirror, root ssh, `rsync`. zk2-node-build installs
  the rest: ZFS, Incus 7 from the Zabbly repository, borg 1.4, jq, msmtp,
  nftables.
- **Workstation:** bash, ssh, rsync, git, openssl.
- **Elsewhere:** a borg repository reachable over ssh, an SMTP relay.

## Limitations

- Instances of the default Incus project, on ZFS storage pools only;
  zk2-backup refuses other drivers by name.
- One public address per node is the tested setup; several `forward*.yaml`
  files are supported by zk2-publish but not rehearsed.
- cloud-init 22.2+ reports `ssh_pwauth: unchanged` in the profile as
  deprecated (removal announced for 27.2); new instances finish cloud-init
  as "done, with warnings". See the comment in `cloud-init/base.yaml`.
- No monitoring yet beyond the nightly report.

## License

MIT, see [LICENSE](LICENSE).
