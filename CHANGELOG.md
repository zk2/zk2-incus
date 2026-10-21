# Changelog

## Unreleased

- `zk2-health`: a running instance with an explicit `boot.autostart=false`
  is now reported - it would not come back after a node reboot. Temporary
  instances (`user.zk2.ephemeral`) are exempt.

## v0.1.0 - 2026-10-05

The first release. Every step of the README was run on real nodes, two VMs
with nested Incus: built from a fresh Debian 13, used, backed up, restored,
rebuilt from nothing and restored from the archives, and linked as a pair.

Tested with Debian 13 (kernel 6.12.111), Incus 7.0.1 from Zabbly, ZFS
2.3.9, borg 1.4.0, nginx 1.30.5 from nginx.org, OpenVPN and easy-rsa from
Debian 13.

### Nodes

- `zk2-node-build`: a node from a fresh Debian 13 - disk guards, Secure Boot
  check, ZFS mirror with the headers metapackage, Incus from Zabbly with a
  verified key, storage with refquota, bridge, DHCP routes for the VPN with
  the default route RFC 3442 requires, split DNS, host firewall in its own
  nftables table, VPN route unit. Idempotent, `--dry-run`.
- `zk2-new-ct`, `zk2-base-profile` and `cloud-init/base.yaml`: instances
  with limits, an isolated uid range, a fixed address, user, ssh policy and
  keys, shell; docker on request.
- `zk2-publish`: Incus network forwards and ACLs from git - `diff`,
  `apply`, `show`, and what exists on the node but not in git.
- `zk2-health`: kernel and ZFS module, pool devices, memory limits and OOM
  kills, instances without limits, refquota, front and OpenVPN certificates
  and the revocation list, VPN routes, split DNS, backup inventory, leftover
  snapshots, publication drift.
- `zk2-list`, `zk2-idmap`, `zk2-shell`, `zk2-ssh-policy`.

### Backup and restore

- `zk2-backup`: snapshot, read-only mount, borg; containers, VMs at file
  level, VMs whole on the monthly policy; pre-backup hooks; per-instance
  retention; `BORG_COMPACT`.
- `hooks/`: PostgreSQL, MariaDB, PostgreSQL in docker, GitLab.
- `zk2-host-backup`: the host root and a plain-text dump of its Incus state.
- `zk2-restore`: by the map in waves, one instance, a probe under another
  name, limits from the host dump, whole VMs, a stream from a peer.
- `zk2-archives`, `zk2-export`, `zk2-nightly` (`MAIL_WHEN`).

### Several nodes

- `zk2-fleet collect`, peers in `zk2-deploy`, `zk2-ssh-config`,
  `zk2-incus-peers`, `zk2-send`.

### Roles

- `containers/nginx`: the front, catch-alls, vhost generator into the fleet.
- `containers/openvpn`: `zk2-vpn-setup`, `-add`, `-revoke`, `-show`.

### Workstation

- `zk2-fleet` (`init`, `add-node`, `collect`) and `zk2-deploy` (code to
  `/opt/zk2`, data to `/etc/zk2`, container roles).

### Known limitations

- Default Incus project and ZFS storage pools only.
- cloud-init reports `ssh_pwauth: unchanged` as deprecated (removal
  announced for 27.2); new instances finish as "done, with warnings".
- No monitoring yet beyond the nightly report; Zabbix integration, more
  workstation tools and an OpenVZ 7 migration guide are planned.
