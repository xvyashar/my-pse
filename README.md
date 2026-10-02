# My Personal Server Environment

Ansible playbook that rebuilds a personal VPS from scratch: base hardening,
Docker, an nginx reverse proxy, and app services wired to subdomains.

Intended for a machine you wipe and reprovision often. Every run is idempotent,
so re-running converges rather than duplicating.

## Requirements

- **Target**: Debian 12/13 or Ubuntu. Root password access for the very first run.
- **Control node**: Linux, macOS, or WSL. Ansible does not support Windows as a
  control node.
- **A domain you control**, with two DNS records pointing at the server's public
  IP:

  | Record | Type | Resolves to | Needed for |
  |---|---|---|---|
  | apex | `A` | `yourdomain.com` | the nginx landing page |
  | wildcard | `A` | `*.yourdomain.com` | every subdomain, now and later |

  Both are required. The wildcard is what covers `9router.yourdomain.com`, and
  it means you don't need a new record for each service you add.

  The playbook creates no DNS records. Its checks request `127.0.0.1` with a
  `Host` header, so they pass without DNS resolving — the records matter for
  real traffic to arrive, not for the run to go green.
- An SSH keypair on the control node. If you don't have one yet:

  ```bash
  ssh-keygen -t ed25519 -N "" -C "pse@ansible-control" -f ~/.ssh/id_ed25519
  ```

  The playbook installs the public half onto the new user. Keep the private half
  on the control node only. `-N ""` leaves it without a passphrase, which is what
  you want for a key an unattended playbook uses — it cannot answer an
  interactive passphrase prompt.

## Setup

```bash
cp secrets.ini.example secrets.ini    # host, port, user, passwords
cp domain.txt.example domain.txt      # your apex domain
```

Edit both. `secrets.ini` is gitignored; `secrets.ini.example` and
`domain.txt.example` are committed as templates.

> [!NOTE]
> Keep ini files EOF on `LF`

## Usage

### First run, on a freshly installed server

```bash
ansible-playbook site.yml -i inventory.ini -i secrets.ini --ask-pass
```

`--ask-pass` is only needed here. It supplies the **root** password, which comes
from your VPS provider and is not in `secrets.ini`. This run disables root login
at the end.

### Every run after that

```bash
ansible-playbook site.yml -i inventory.ini -i secrets.ini --skip-tags bootstrap
```

No prompts, no passwords. Everything authenticates as the new user over the SSH
key installed during bootstrap. Note the two `-i` flags — a comma-separated
list is parsed as host *names*, not files.

## Plays

| Play | Tag | Runs | What it does |
|---|---|---|---|
| **Bootstrap** | `bootstrap` | once | Creates the sudo user with a hashed password, installs their SSH key, gives them passwordless sudo, updates and upgrades packages, installs and enables ufw and fail2ban, then disables root login and password authentication |
| **Docker Install** | — | every run | Installs Docker Engine from Docker's apt repository (deb822 source, GPG keyring). Removes conflicting distro packages on first install only — on a re-run that would cascade into removing Docker itself |
| **Services** | — | every run | Creates `~/docker-services`, writes `compose.yaml` and the nginx vhost for the apex domain, opens port 80 in ufw, and registers `pse-services.service` so the stack starts at boot |
| **9router** | — | every run | Writes the `9router.<domain>` vhost and verifies the subdomain routes to 9router rather than falling through to the apex |

## Configuration

### `secrets.ini`

| Key | Purpose |
|---|---|
| `ansible_host`, `ansible_port` | Server address |
| `pse_user` | Login user; also the admin account for every later run |
| `pse_password` | That user's password. Hashed with sha512 on write |
| `nine_router_init_password` | `INITIAL_PASSWORD` for the 9router container |

Variable names must start with a letter or underscore. A digit-leading name
works today but is deprecated and will break in ansible-core 2.23.

### `domain.txt`

A single bare domain, no scheme, no port. `9router` is added as a subdomain, so
you need a wildcard DNS record `*.yourdomain` pointing at the server.

## Networking

Port 80 is the only container port published to the internet. nginx is the sole
entry point; 9router is bound to `127.0.0.1:20128` so other services on the host
can reach it while it stays unreachable from outside.

Docker writes its own iptables rules and bypasses ufw, so published container
ports are filtered through the `DOCKER-USER` chain instead. That chain is
rebuilt by `pse-docker-user-rules.service` on every boot and after every
`docker.service` restart, because Docker flushes it. `docker_allowed_ports` in
the Docker Install play controls what is permitted.

The playbook verifies the live chain matches that list after each run, because
iptables state is invisible to Ansible and a silently empty chain would leave
every published port open.

## Verification

Each run ends with assertions rather than assuming success:

- the live `DOCKER-USER` rules reflect `docker_allowed_ports` and still drop
  everything else
- the apex domain serves the page Ansible wrote
- `nginx -t` passes, which also proves nginx can resolve the `9router` upstream
- the `9router` subdomain does **not** return the apex landing page — the marker
  in that page is how a failed `server_name` match gets caught

## Layout

```
inventory.ini                 committed; connection settings
secrets.ini                   gitignored; host, user, passwords
domain.txt                    your apex domain
files/                        static files, copied verbatim
  pse-docker-user-rules.service
templates/                    templated files, Jinja rendered
  compose.yaml                the container stack
  docker.sources              Docker's apt source, per distro
  nginx/default.conf          apex vhost
  nginx/9router.conf          9router vhost
  nginx/index.html            landing page
  pse-docker-user-rules       DOCKER-USER rebuild script
  pse-services.service        runs `docker compose up` at boot
```

Editing anything under `templates/` or `files/` is the intended way to change
this server. Direct edits on the server are overwritten on the next run.