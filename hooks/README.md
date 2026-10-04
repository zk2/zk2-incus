# Pre-backup hooks

A hook runs on the node before `zk2-backup` snapshots an instance, with the
instance name as its only argument. It leaves a consistent database dump on
the instance's disk, so the snapshot carries it.

The snapshot alone is already crash-consistent: PostgreSQL and InnoDB replay
their journal on start, as after a power cut. A dump adds a second, logical
copy of the same data: you can check it without restoring the instance, pull
a single table out of it, and load it into a newer database version.

Name the hook in the third column of `inventory.conf`:

```
# name      policy   hook
db          daily    dump-postgres
shop        daily    dump-mariadb
```

Hooks are looked up in `/etc/zk2/hooks/` first (your own, from
`nodes/<node>/hooks/` in your fleet) and then in `/opt/zk2/hooks/` (these).
A hook with the same name in your fleet wins.

| Hook | Dumps |
|---|---|
| `dump-postgres` | all databases and roles of PostgreSQL in the instance (`pg_dumpall` as `postgres`) |
| `dump-mariadb` | all databases of MariaDB or MySQL in the instance (root over the unix socket) |
| `dump-docker-postgres` | PostgreSQL running in a docker container inside the instance |
| `dump-gitlab` | the database of an omnibus GitLab |

Dumps land in `/var/backups/zk2/` inside the instance (`DEST` overrides).

Rules every hook here follows, and yours should too:

- **No compression.** A compressed dump changes as a whole on every change,
  and borg would store a new blob of its full size every night. A plain one
  changes line by line; borg keeps the difference and compresses it itself.
- **Write to `.tmp`, then `mv`.** An interrupted dump must not replace the
  previous good one.
- **`</dev/null` on every incus call.** The Incus client waits for the end
  of stdin before talking to the daemon; a hook run over ssh without a
  terminal otherwise hangs forever.
- **No stored credentials.** Local socket authentication (peer, unix_socket,
  trust inside the official postgres image) instead of passwords in files.
- **Non-zero exit on failure.** `zk2-backup` reports it and still takes the
  snapshot.
