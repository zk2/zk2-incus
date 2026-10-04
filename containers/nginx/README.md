# nginx front

One instance on each node terminates TLS for every site and proxies to the
instances behind it. The node publishes 80 and 443 to it
(`publication/forward.yaml`); nothing else needs a public port for HTTP.

## Files

| Where | What | Deployed by |
|---|---|---|
| zk2 `containers/nginx/nginx.conf` | http context: TLS policy, proxy defaults, gzip, logs; includes `conf.d/*.conf` and `stream.d/*.conf` | `zk2-deploy NODE nginx` (FILES) |
| zk2 `containers/nginx/snippets/` | `acme.conf` (http-01 webroot), `proxy.conf` (baseline proxy headers), `maps.conf`, `maintenance.conf`, `cloudflare-real-ip.conf` | `zk2-deploy NODE nginx` (FILES) |
| fleet `nodes/NODE/nginx/conf.d/` | the vhosts of this node, `default.conf` catch-alls | `zk2-deploy NODE nginx` (NODE_FILES) |
| fleet `nodes/NODE/deploy/nginx.conf` | which instance, which files, `nginx -t`, reload | read by `zk2-deploy` |
| zk2 `containers/nginx/conf_generator` | writes a new vhost into the fleet | run on the workstation |

Vhosts live in the fleet, per node, never in zk2: each node has its own
domains and upstreams. A vhost of another node deploys fine - `nginx -t`
does not check that an upstream exists - and breaks on the first request.

## The instance

`nginx.conf` targets the nginx.org packages: they run as user `nginx` and
have the stream module built in. Debian's own `nginx` runs as `www-data`
and ships stream separately; with it, change `user` and add
`libnginx-mod-stream`.

```sh
zk2-new-ct                                # name nginx, debian/13/cloud, an address in CT_NET
incus exec nginx -- sh -c '
  apt-get install -y curl gnupg ca-certificates certbot
  curl -fsSL https://nginx.org/keys/nginx_signing.key | gpg --dearmor -o /usr/share/keyrings/nginx.gpg
  echo "deb [signed-by=/usr/share/keyrings/nginx.gpg] https://nginx.org/packages/debian trixie nginx" \
      > /etc/apt/sources.list.d/nginx.list
  apt-get update && apt-get install -y nginx
  mkdir -p /var/www/html/default
  rm -f /etc/nginx/conf.d/default.conf'    # the package's example server
```

Then on the workstation: `cp -r ../zk2-incus/examples/node/nginx nodes/NODE/`,
`cp ../zk2-incus/examples/node/deploy/nginx.conf nodes/NODE/deploy/`, commit,
`zk2-deploy NODE nginx`, and publish 80 and 443 to the instance.

`default.conf` holds two catch-alls, and both matter:

- **80** answers the ACME http-01 challenge for any name - that is how a new
  domain gets its certificate before it has a vhost - and drops the rest.
- **443 with `ssl_reject_handshake`**: without a `default_server` on 443,
  nginx falls back to the first vhost loaded, and a scan by IP reaches it.

## A new site

```sh
# 1. on the front instance: the certificate, through the catch-all on 80
incus exec nginx -- certbot certonly --webroot -w /var/www/html/default -d app.example.org

# 2. on the workstation, in the fleet
../zk2-incus/containers/nginx/conf_generator -o nodes/node1/nginx/conf.d \
    app.example.org proxy http://10.20.0.30:8080/
git add nodes/node1/nginx && git commit -m "app.example.org"
../zk2-incus/tools/zk2-deploy node1 nginx
```

The other way round - vhost first - fails `nginx -t` on the missing
certificate files. `zk2-deploy` stops before the reload then, but the
files are already on the instance: remove the vhost from the fleet and
deploy again, or fetch the certificate and reload.

## Certificate renewal

- **webroot, not the nginx plugin.** `webroot_path` in
  `/etc/letsencrypt/renewal/*.conf` must equal the `root` of
  `snippets/acme.conf`.
- **A deploy hook reloads nginx.** Without the nginx installer nobody reloads
  it, and a renewed certificate sits on disk while nginx keeps serving the
  old one from memory until the next restart:

  ```sh
  printf '#!/bin/sh\nsystemctl reload nginx\n' > /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
  chmod +x /etc/letsencrypt/renewal-hooks/deploy/reload-nginx.sh
  ```

- **`certbot renew --dry-run` does not prove renewal works.** It talks to the
  staging server with a throwaway account. Each `renewal/*.conf` names an
  `account =`; if that account directory is missing (certificates brought
  from another front without `accounts/`), every real renewal fails while
  the dry run stays green. Check once with a real renewal of the certificate
  that expires first: `certbot renew --cert-name NAME --force-renewal`, then
  confirm with `openssl s_client` that the served `notAfter` moved - that
  also proves the deploy hook.
- **The timer is `certbot.timer` on Debian, `certbot-renew.timer` on the
  RHEL family.** Disabling the wrong name fails quietly. And look for a
  second mechanism - a crontab line doing the same - that hides the first.
- **Moving certificates: `tar`, not `scp -r`.** `live/` holds symlinks into
  `archive/`; copying resolves them into plain files, and renewal breaks two
  months later, on every domain at once.

`zk2-health` warns 14 days before a certificate on the front expires
(`CERT_DAYS`, `NGINX_CT` in `node.conf`).

## Ways to break it, from loud to silent

1. `scp -r` of `/etc/letsencrypt`: renewal breaks in two months, everywhere.
2. No deploy hook: renewed certificates that nginx never loads.
3. The wrong certbot timer name: renewal runs nowhere, or twice.
4. A backend allow-list with the old front's address: 502 after a move, on
   the vhost you checked least.
5. No `default_server` on 443: some vhost answers scans by IP, nothing in
   the logs says so.
6. A `proxy_set_header` added in a location silently drops all four
   inherited `X-Forwarded-*` headers; `include snippets/proxy.conf` next to
   it puts them back.
