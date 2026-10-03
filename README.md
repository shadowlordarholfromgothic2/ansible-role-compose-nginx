# `compose_nginx` role

Runs **nginx as a Docker Compose stack** in front of the other compose stacks on
the same host, for example the ones deployed by
[`compose_service`](https://github.com/shadowlordarholfromgothic2/ansible-role-compose-service).

```text
client ──:443──▶ nginx ──docker network "proxy"──▶ nexus:8081
                (/opt/nginx)                    └─▶ nexus:8082   (docker registry)
                (/etc/nginx)                    └─▶ gitlab:80    (any other stack)
```

nginx and the backends share a docker network, so a backend is addressed by
its **container name** and does not need to publish any port. Each site comes
down to four settings: the **backend**, the **domain name(s)**, the
**certificate**, and the **docker network** the backend shares with nginx.

The configuration in `/etc/nginx` is rendered from the role's templates. Your
own **snippets** plug into it at http, server and location level. A site can
also bring a template of its own.

The role does not install Docker, and it does not obtain certificates from an
ACME CA. It can generate self-signed ones, or pick up files that something
else (certbot, …) puts in place.

## Requirements

* ansible-core ≥ 2.14 (tested on 2.21)
* Collection `community.docker` ≥ 3.6.0 (for `docker_compose_v2`), also listed
  in [`requirements.yml`](requirements.yml).
* Docker Engine and the **`docker compose` plugin ≥ 2.18.0** on the target. The
  role does *not* install Docker. It checks for the plugin up front and fails
  with a clear message if it is missing or too old.
* `openssl` on the target, but only for self-signed certificates. It is already
  there wherever `ca-certificates` is installed.
* nginx ≥ 1.27.3 in the image (default `1.30.5-alpine`) for the runtime DNS
  lookup of upstreams, see [Backends](#backends).
* `become: true`, because files are written to `/opt` and `/etc` as root.

## What the role does

1. **Validates input.** Ansible checks every variable against
   [`meta/argument_specs.yml`](meta/argument_specs.yml) before the first task:
   types, allowed values, required keys, and no unknown keys in sites,
   locations, certificates and snippets. The role then checks what the spec
   cannot: names are unique and valid as file names, backends look like
   URLs, and every certificate and snippet a site names exists.
2. **Preflight.** `docker compose version` must work and be ≥ `nginx_min_compose_version`.
3. **Teardown** (only for `nginx_state: absent`), see [Removing nginx](#removing-nginx).
4. **Docker network.** Creates `nginx_docker_network` (and `nginx_docker_networks_extra`)
   if it is missing.
5. **Config.** Writes certificates, `mime.types`, snippets, `nginx.conf`, the
   upstreams, the default server and one file per site, then deletes `*.conf`
   files that no site or snippet owns. Every changed file notifies a **reload**.
6. **Compose file.** Renders `/opt/nginx/docker-compose.yml` and validates it
   with `docker compose config -q`.
7. **Config test.** Runs `nginx -t` in a one-off container of the same compose
   service, so it also works before nginx has ever run. If the test fails,
   the play stops before the reload and the running nginx keeps its old config.
8. **Converge.** Runs `docker compose up -d`, then the pending reload
   (`nginx -s reload`, graceful).
9. **Readiness.** Polls `http://127.0.0.1/healthz` until nginx answers.

On the managed host:

```text
/opt/nginx/                     # nginx_compose_dir, safe to delete and recreate
└── docker-compose.yml          # 0640
/etc/nginx/                     # nginx_config_dir, mounted read-only as /etc/nginx
├── nginx.conf
├── mime.types
├── conf.d/
│   ├── _upstreams.conf         # one upstream per backend host:port
│   ├── _default.conf           # catch-all for unknown names + /healthz
│   ├── nexus.conf              # one file per site
│   └── nexus-docker.conf
├── snippets/                   # nginx_snippets
│   └── allow-lan.conf
└── certs/                      # 0700
    ├── nexus.crt
    └── nexus.key               # 0600
```

The role mounts the whole directory rather than single files. When Ansible
replaces a file it creates a new inode, and a single-file bind mount would
keep pointing at the old one, so nginx would never see the change.

## Variables

See [`defaults/main.yml`](defaults/main.yml) for the full commented list, or
print the reference generated from the argument spec:

```bash
ansible-doc -t role compose_nginx
```

| Variable                       | Default                         | Purpose |
|--------------------------------|---------------------------------|---------|
| `nginx_sites`                  | `[]`                            | Virtual hosts, see [Sites](#sites) |
| `nginx_certificates`           | `[]`                            | Certificates, see [Certificates](#certificates) |
| `nginx_snippets`               | `[]`                            | Your config pieces, see [Snippets](#snippets) |
| `nginx_site_defaults`          | see defaults                    | Per-site settings applied under each site's own keys |
| `nginx_docker_network`         | `proxy`                         | Network shared with the backends, created if missing |
| `nginx_docker_networks_extra`  | `[]`                            | More networks for nginx to join (created if missing) |
| `nginx_state`                  | `present`                       | `present` / `stopped` / `absent` |
| `nginx_image` / `nginx_version`| `nginx` / `1.30.5-alpine`       | Image, pin the tag |
| `nginx_pull`                   | `missing`                       | `always` / `missing` / `never` / `policy` |
| `nginx_compose_dir`            | `/opt/nginx`                    | Compose project dir |
| `nginx_config_dir`             | `/etc/nginx`                    | Config dir on the host |
| `nginx_compose_project`        | `nginx`                         | Compose project and container name |
| `nginx_compose_template`       | `docker-compose.yml.j2`         | Compose template; an absolute path to use your own |
| `nginx_bind_address`           | `""` (all)                      | Host address the ports are published on |
| `nginx_http_port` / `nginx_https_port` | `80` / `443`            | Published host ports |
| `nginx_extra_volumes`          | `[]`                            | Extra bind mounts (compose short syntax) |
| `nginx_extra_hosts`            | `host.docker.internal:host-gateway` | So backends outside docker are reachable |
| `nginx_min_compose_version`    | `2.18.0`                        | Oldest `docker compose` plugin accepted |
| `nginx_http_snippets`          | `[]`                            | Snippets included in `http {}` |
| `nginx_resolver`               | `127.0.0.11 valid=10s ipv6=off` | Docker's embedded DNS |
| `nginx_upstream_resolve`       | `true`                          | Look backends up at runtime, see [Backends](#backends) |
| `nginx_upstream_keepalive`     | `16`                            | Idle keepalive connections per upstream |
| `nginx_ssl_protocols` / `nginx_ssl_ciphers` | Mozilla intermediate | TLS settings for all sites |
| `nginx_worker_processes` / `nginx_worker_connections` | `auto` / `1024` | |
| `nginx_error_log_level`        | `notice`                        | Error log level (`docker logs`) |
| `nginx_keepalive_timeout`      | `65s`                           | Client keepalive timeout |
| `nginx_server_tokens`          | `false`                         | Show the nginx version in headers and error pages |
| `nginx_default_status`         | `444`                           | Answer for unknown host names (444 = close) |
| `nginx_health_path`            | `/healthz`                      | Health endpoint on the default server |
| `nginx_selfsigned_days`        | `3650`                          | Lifetime of self-signed certificates |
| `nginx_purge_unmanaged`        | `true`                          | Delete `*.conf` in `conf.d/` and `snippets/` that nothing owns |
| `nginx_wait_timeout` / `nginx_wait_delay` | `60` / `3`           | Readiness probe |

The role registers `nginx_compose_result` (the output of `docker_compose_v2`).

### Several services on one host

`nginx_sites`, `nginx_certificates` and `nginx_snippets` are **merged** with
every variable named `nginx_sites_<x>`, `nginx_certificates_<x>` and
`nginx_snippets_<x>`. Each service's group_vars can then bring its own sites,
and a host in several service groups gets all of them. With a single
`nginx_sites` list, the last group would silently win.

```yaml
# group_vars/nexus/nginx.yml
nginx_sites_nexus: [...]
# group_vars/gitlab/nginx.yml
nginx_sites_gitlab: [...]
```

Do not give any other variable one of these prefixes. Site, certificate and
snippet names must be unique across all of them, and the role checks this.

The argument spec only knows the three main lists, so Ansible does not check
the `_<x>` variables before the first task. The role's own checks still cover
every entry, wherever it came from.

## Usage

Install the role, for example from `requirements.yml` in the playbook
repository:

```yaml
# requirements.yml
collections:
  - name: community.docker

roles:
  - name: compose_nginx
    src: git+https://github.com/shadowlordarholfromgothic2/ansible-role-compose-nginx.git
    version: main               # or a release tag, e.g. 1.0.0
```

```bash
ansible-galaxy install -r requirements.yml
```

Describe the sites:

```yaml
# group_vars/nexus/nginx.yml
nginx_certificates_nexus:
  - name: nexus
    self_signed: true

nginx_sites_nexus:
  - name: nexus
    server_name: nexus.lab.local
    backend: http://nexus:8081
    certificate: nexus
```

Run nginx **before** the stacks it proxies, because it creates the network they
join:

```yaml
- hosts: nexus
  become: true
  roles:
    - role: compose_nginx
    - role: compose_service     # the backend stack
```

## Sites

```yaml
nginx_sites:
  - name: nexus                     # file conf.d/nexus.conf, ^[a-z0-9][a-z0-9_.-]*$
    server_name: nexus.lab.local    # domain name, or a list of them
    backend: http://nexus:8081      # scheme://container_name:port[/path]
    certificate: lab                # from nginx_certificates; omit for plain HTTP
```

A site with a `certificate` listens on 443 (HTTP/2), and its port 80 only
redirects to HTTPS. Set `https_redirect: false` to serve both. Without a
certificate it listens on 80 only.

Every key of [`nginx_site_defaults`](defaults/main.yml) can be overridden per site:

| Key                       | Default | |
|---------------------------|---------|---|
| `https_redirect`          | `true`  | Port 80 redirects to HTTPS (when there is a certificate) |
| `http2`                   | `true`  | |
| `hsts`                    | `false` | Only turn it on once HTTPS works for good, because browsers remember it |
| `client_max_body_size`    | `100m`  | `0` = unlimited |
| `proxy_buffering`         | `true`  | `false` streams responses straight to the client |
| `proxy_request_buffering` | `true`  | `false` streams uploads straight to the backend |
| `proxy_connect_timeout`   | `10s`   | |
| `proxy_read_timeout`      | `60s`   | |
| `proxy_send_timeout`      | `60s`   | |
| `proxy_headers`           | `{}`    | Extra `proxy_set_header` name → value |
| `snippets`                | `[]`    | Snippet names included in `server {}` |
| `config`                  | `""`    | Raw lines for `server {}` |
| `locations`               | `[]`    | More locations, see below |

Overriding `nginx_site_defaults` itself replaces the whole dict, so copy all
keys if you change it.

A site accepts only the keys above plus `name`, `server_name`, `backend`,
`certificate` and `template`. Anything else (a typo such as `server_names` or
`proxy_buffer`) fails the argument validation.

Every proxied location sends `Host` (as the client sent it, port included),
`X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`, `X-Forwarded-Host` and the
websocket `Upgrade`/`Connection` headers. Websockets work without extra
settings.

### Locations

```yaml
    locations:
      - path: /v2/                  # anything `location` accepts: "= /x", "~ \.php$", ...
        backend: http://nexus:8082  # optional, proxies like the site does
        snippets: [allow-lan]       # optional
        proxy_headers: {}           # optional, merged over the site's
        config: |                   # optional raw lines
          proxy_read_timeout 900s;
```

A `/` entry replaces the generated `location /`. In that case the site needs
no `backend` of its own.

### Your own template

When a site does not fit the generated shape (static files, odd rewrites, …),
give it a template:

```yaml
  - name: legacy
    server_name: legacy.lab.local
    template: "{{ playbook_dir }}/files/nginx/legacy.conf.j2"   # absolute path
```

The role renders it to `conf.d/legacy.conf`. It sees the site as `site`, so it
can use `site.server_names`, and for its backends `_nginx_backends[site.backend].upstream`.

## Backends

A backend stack joins the shared network as an external network. nginx then
reaches it by container name:

```yaml
# compose/<service>/docker-compose.yml.j2
services:
  nexus:
    container_name: nexus           # the host name nginx uses
    networks: [proxy]
    # no `ports:` needed; publish on 127.0.0.1 only if the host itself needs it
networks:
  proxy:
    name: "{{ nginx_docker_network | default('proxy') }}"
    external: true
```

* **Run this role first.** It creates the network, and `docker compose up`
  of a stack that declares an external network that does not exist fails.
* **Use the container name**, which is unique per host. A compose *service*
  name such as `app` or `db` is registered on every network the container joins,
  so two stacks that both have an `app` would share one DNS name on the
  `proxy` network.
* nginx starts even while a backend is down or not deployed yet. Every
  backend host:port becomes an `upstream` with `server … resolve`, so nginx
  looks the name up through Docker's DNS **at runtime**. Until the backend
  exists it answers `502`. It picks the backend up within ~30 s (nginx caches
  the failed lookup), and follows a recreated container to its new IP within
  the resolver's `valid=10s`. Without `resolve`
  (`nginx_upstream_resolve: false`), nginx would refuse to start while any
  backend name is unknown, and would keep the old IP until a reload.
* Backends that are not containers: `http://host.docker.internal:8080` for a
  service on the docker host itself, or any DNS name or IP reachable from the
  container.
* `https://` backends work too. nginx sends SNI with the backend's host name.

## Certificates

```yaml
nginx_certificates:
  # PEM content: the certificate followed by its intermediates, and the key.
  # lookup('file') decrypts vault-encrypted files transparently.
  - name: lab
    cert: "{{ lookup('ansible.builtin.file', 'files/certs/lab.crt') }}"
    key: "{{ vault_lab_tls_key }}"

  # Self-signed, generated on the host.
  - name: nexus
    self_signed: true
    domains: [nexus.lab.local]      # optional, default: the server names of its sites

  # Neither: something else (a certbot deploy hook, …) puts
  # certs/external.crt and .key in place. The role only checks that they exist.
  - name: external
```

Several sites can share one certificate (a wildcard, say). Keys are written
with mode `0600` and `no_log`.

A self-signed certificate is only regenerated when it is missing or when its
names no longer match `domains`. Clients that were told to trust it keep
trusting it between runs. Delete the `.crt` to force a new one.

## Snippets

Snippets are your own config pieces, written to `snippets/<name>.conf`. They
take effect only where they are named:

```yaml
nginx_snippets:
  - name: allow-lan
    content: |
      allow 192.168.0.0/16;
      deny all;
  - name: security-headers
    src: "{{ playbook_dir }}/files/nginx/security-headers.conf.j2"   # a template

nginx_http_snippets: [rate-limits]  # http {} level, e.g. limit_req_zone, map, log_format

nginx_sites:
  - name: admin
    server_name: admin.lab.local
    backend: http://admin:8080
    snippets: [allow-lan, security-headers]       # server {} level
    locations:
      - path: /api/
        backend: http://admin:8080
        snippets: [rate-limit-api]                # location {} level
```

For a one-off, use a site's or location's `config` instead, which takes raw lines.

## Example: Nexus

Sonatype Nexus with its Docker registry on a host name of its own:

```yaml
nginx_certificates_nexus:
  - name: nexus
    self_signed: true                    # SANs: both domains below

nginx_sites_nexus:
  - name: nexus                          # UI, REST, Maven/npm/PyPI/raw repositories
    server_name: nexus.lab.local
    backend: http://nexus:8081
    certificate: nexus
    client_max_body_size: 0              # raw repos and docker layers are large
    proxy_buffering: false               # stream downloads ...
    proxy_request_buffering: false       # ... and uploads, no spooling to disk
    proxy_read_timeout: 300s
    proxy_send_timeout: 300s

  - name: nexus-docker                   # docker registry API (/v2/) on its own name
    server_name: docker.lab.local
    backend: http://nexus:8082           # the Docker connector of a docker repository
    certificate: nexus
    client_max_body_size: 0
    proxy_buffering: false
    proxy_request_buffering: false
    proxy_read_timeout: 900s
    proxy_send_timeout: 900s
```

Why these settings:

* **Sonatype's recommendations.** Unlimited body size, no buffering in either
  direction, and long timeouts. Without them, large uploads fail with `413`, or
  nginx spools multi-GB artifacts to its own disk first.
* **`Host` and `X-Forwarded-Proto`**, which the role always sends, are what Nexus
  builds absolute URLs from: the docker token realm, blob upload locations and
  links in the UI. Also set the *Base URL* capability in Nexus to
  `https://nexus.lab.local`.
* **Docker needs its own host name.** The registry API has to sit at `/v2/`
  of a host, and a Nexus docker repository answers it on a connector port
  (8082 here). Alternatively, keep a single name and route only the API with a
  location on the main site:
  ```yaml
      locations:
        - path: /v2/
          backend: http://nexus:8082
  ```
* **Self-signed.** Docker clients must trust the certificate: copy
  `/etc/nginx/certs/nexus.crt` to
  `/etc/docker/certs.d/docker.lab.local/ca.crt` on each client.

## Removing nginx

```bash
ansible-playbook playbook.yml -e nginx_state=absent
```

This runs `docker compose down` and deletes `/opt/nginx`. **`/etc/nginx`
(including the certificates) and the docker network are kept**, because other
stacks may still be attached to the network. The play goes on, so roles after
this one still run.

`nginx_state: stopped` keeps the container but stops it.

## Tags

| Tag      | Tasks |
|----------|-------|
| `always` | Input validation and the `docker compose` preflight |
| `config` | Network, directories, certificates, config files, compose file, `nginx -t` |
| `deploy` | Network, teardown, `docker compose up`, reload, probe |
| `verify` | Probe only |

```bash
ansible-playbook playbook.yml --tags config      # re-render + test + reload, no compose up
```

## Notes

* **Changes are reloaded, not restarted.** `/etc/nginx` is a bind mount, so
  `docker compose up` cannot see a config change. Every config file therefore
  notifies `nginx -s reload`, which is graceful. Changes to the compose file
  (image tag, ports, …) recreate the container through `docker compose up`.
* **The config test protects the running nginx.** A broken config fails the
  play before the reload. The files on disk are already the new, broken ones,
  so fix the variables and run again before the container restarts.
* **Unmanaged files are deleted.** With `nginx_purge_unmanaged: true`,
  removing a site or snippet from the variables removes its file. A hand-made
  `*.conf` in `conf.d/` or `snippets/` is removed too. `certs/` is never purged.
* **Only `mime.types` is shipped.** Mounting `/etc/nginx` hides the image's
  `fastcgi_params`, `uwsgi_params` and the `modules` link. Put what you need into a snippet.
* **The network survives the stack.** The role creates it with the docker CLI
  rather than in nginx's compose file, so `docker compose down` here never
  pulls it away from the backends.
* **Check mode works**, also on a fresh host. Steps that need files which a
  dry run has not written (the config test, the readiness probe, the compose
  dry run on a first deploy) are skipped.

## Testing

A Molecule scenario installs Docker in a Debian 13 container, deploys nginx
*before* its backend exists, starts a [`whoami`](https://github.com/traefik/whoami)
backend on the shared network, then checks idempotence, file permissions,
HTTPS with a self-signed certificate and its SANs, the redirect, the forwarded
headers, snippets, extra locations, a site merged from `nginx_sites_*` and the
catch-all. It also runs the role in check mode, once before the first deploy
and once after it. CI runs it, together with both linters, on every pull
request:

```bash
pip install -r requirements-dev.txt     # same pinned versions as CI
yamllint --strict .
ansible-lint
molecule test
```

ansible-lint and Molecule install the collections and test roles from
[`requirements.yml`](requirements.yml) on their own.

## Releases

PRs are squash-merged, so the PR title becomes the commit on `main`. It must be
a [Conventional Commit](https://www.conventionalcommits.org/) (a check enforces
this), because it decides the next version and the changelog entry:

| PR title                                              | Next release | Changelog       |
|-------------------------------------------------------|--------------|-----------------|
| `feat!: …` or a `BREAKING CHANGE:` footer             | major        | ⚠ Breaking      |
| `feat: …`                                             | minor        | Features        |
| `fix:` / `perf:` / `revert:` / `docs: …`              | patch        | own section     |
| `ci:` / `test:` / `refactor:` / `build:` / `style:` / `chore: …` | none | not listed |

After each merge, [release-please](https://github.com/googleapis/release-please)
opens or updates a release PR with the version bump and the new `CHANGELOG.md`
entry. Merging that PR tags the release (e.g. `1.2.0`) and publishes a GitHub
release.

PRs opened with the default `GITHUB_TOKEN` do not trigger workflows, so the
release PR gets no CI checks. If branch protection requires them, add a
fine-grained PAT as the `RELEASE_PLEASE_TOKEN` secret.

## License

MIT
