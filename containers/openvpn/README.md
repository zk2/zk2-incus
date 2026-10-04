# OpenVPN gateway

One instance on each node gives you - and only you - a way into the instance
network: ssh, databases, admin pages that are never published. Split tunnel:
clients reach the instance network through the tunnel and everything else
directly.

| What | Where |
|---|---|
| `bin/zk2-vpn-setup` | sets up the server in the instance, idempotent |
| `bin/zk2-vpn-add NAME` | issues a client: certificate and a ready `.ovpn` |
| `bin/zk2-vpn-revoke NAME` | revokes a client for real |
| `bin/zk2-vpn-show [NAME]` | the whole picture, and what is wrong |

`zk2-deploy NODE openvpn` puts them into `/usr/local/bin` of the instance
(`examples/node/deploy/openvpn.conf`).

## Setting it up

```sh
zk2-new-ct                                   # name openvpn, debian/13/cloud, VPN_CT_IP
# in the fleet: cp ../zk2-incus/examples/node/deploy/openvpn.conf nodes/NODE/deploy/
zk2-deploy NODE openvpn
incus exec openvpn -- zk2-vpn-setup --remote PUBLIC_IP --pool VPN_NET --route CT_NET --port 1194
```

Then on the node side, all of it from `node.conf` and the publication:

- `VPN_NET` (the pool) and `VPN_CT_IP` (this instance) in `node.conf`, then
  `zk2-deploy NODE` and `zk2-node-build` - it routes the pool through this
  instance on the node and, by DHCP, inside every instance;
- UDP 1194 to this instance in `publication/forward.yaml` with
  `snat: true`, then `zk2-publish apply`.

**Without the DHCP routes VPN clients cannot use instances**, and the symptom
misleads: ping works, the port opens, sshd logs the connection, and the
session hangs. Replies would go to the node, which masquerades them as new
outgoing connections. `zk2-node-build` sets the routes, `zk2-health` checks
them. Instances created before VPN_NET was set get the route at their next
DHCP renewal or restart.

The server keeps client addresses: instances see the real `10.20.100.x` of
each client in their logs. Only traffic leaving the node is masqueraded.

## Clients

```sh
incus exec openvpn -- zk2-vpn-add laptop     # -> /root/vpn_configs/laptop.ovpn
incus file pull openvpn/root/vpn_configs/laptop.ovpn .
incus exec openvpn -- rm /root/vpn_configs/laptop.ovpn
```

The `.ovpn` holds the client's private key. Move it over a secure channel
and do not leave copies around.

## Revoking

```sh
incus exec openvpn -- zk2-vpn-revoke laptop
```

Revoking is not deleting files: the server checks clients by the CA's
signature, and a client whose files were merely deleted on the server keeps
connecting until its certificate expires. A real revocation is `revoke` in
the index, a reissued `crl.pem`, and `crl-verify` in `server.conf`. The
server's own current certificate is refused (it looks like any client's in a
list). An open session lasts until the client reconnects; restarting
`openvpn-server@server` cuts it, and every other client too.

### The revocation list, the one thing that breaks by itself

| What | Result |
|---|---|
| `crl.pem` mode 600 | the server re-reads it on every connection as `nobody` - and refuses EVERYONE |
| `crl.pem` inside the closed `/etc/openvpn/server` | `nobody` cannot even stat it; the list loaded at start stays, and revocations silently do not apply until a restart |
| an expired list (180 days by default) | OpenVPN refuses EVERY client |

That is why the list lives in `/etc/openvpn/crl.pem`, mode 644. The third
row is the only thing in this role that breaks on its own, months after
the last change: `zk2-vpn-revoke` prints the next date, `zk2-vpn-show` and
`zk2-health` warn ahead (`CRL_DAYS`), and any `easyrsa gen-crl` followed by
copying the file renews it.

## Where things live

```
/etc/openvpn/easy-rsa/pki     the PKI: index.txt, issued/, private/, the CA key
/etc/openvpn/server/          closed: ca.crt, the server pair, ta.key, server.conf
/etc/openvpn/crl.pem          the revocation list, OPEN on purpose
/etc/openvpn/zk2-vpn.conf     the public address clients connect to
/root/vpn_configs/            issued .ovpn files until they are handed over
```

The PKI directory is the source of truth. Issue and revoke only through the
scripts: a certificate signed past easy-rsa is not in the index, and there is
nothing to revoke it with (`zk2-vpn-show` finds such certificates).

One PKI per node: a CA shared by two servers gives colliding client serial
numbers, and a revocation on one node means nothing obvious on the other.
The instance's backup carries the PKI - losing it means reissuing every
client.
