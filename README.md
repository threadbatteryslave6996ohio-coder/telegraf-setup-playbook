# server-management

Ansible playbooks that turn a fresh Arch Linux or Ubuntu server into a managed,
observable node. Each host gets:

1. Tailscale (and is joined to your tailnet when an auth key is provided)
2. Prometheus node exporter (so an external Prometheus can scrape it)
3. Fluent Bit, forwarding host logs to an external Loki instance
4. auditd, seeded with a high-signal rule set (identity, sudo, SSH, polkit)
5. osquery as a host-visibility agent (where an upstream package exists)
6. Security-related journal + audit events forwarded to an external Splunk HEC
7. Docker (Arch only)

This repo configures **client/agent software only**. It does not provision
Prometheus, Loki, or Splunk — those are external services that must already
exist.

## Layout

The work is split into two stages, each with one folder per OS. Shared
configuration — the inventory and the variable files — lives at the top level;
only the playbooks live inside the stage folders.

```
inventories/              one directory per environment
  stg/                    staging (the ansible.cfg default)
    hosts.ini             the environment's hosts
    group_vars/all/       the environment's variables
      common.yml          values used by every playbook, both OSes
      arch.yml            Arch-only values (packages, endpoints, SSH key)
      ubuntu.yml          Ubuntu-only values
      control.yml         control-plane values (repo, proxy, stack name)
      secrets.yml         secrets shared by the node and control playbooks
  prod/                   production, same layout
  example/                template — copy it to add an environment

requirements.yml          Ansible collections these playbooks need

bootstrap/                stage 1: create the dedicated `ansible` admin user
  arch/   bootstrap-arch.yml
  ubuntu/ bootstrap-ubuntu.yml

site/                     stage 2: install + enable + start the services
  arch/    site-arch.yml
  ubuntu/  site-ubuntu.yml
  control/ site-control.yml   + templates/ for .env and scrape targets
           update-setup.yml   pull the deployment repo + restart, nothing else
           tasks/             steps shared by the two control playbooks

docs/command-privileges.yml   machine-readable map of which tasks use sudo
ansible.cfg                    defaults (sudo on; default inventory = inventories/stg)
```

One inventory serves both stages. Each playbook targets its own group
(`arch_hosts` or `ubuntu_hosts`), and `ansible_user` is the steady-state
`ansible` account. Bootstrap runs once as an existing admin instead — you
override the user per run with `-e ansible_user=...` (see below).

**Bootstrap vs. site.** Bootstrap runs once as an existing privileged login and
creates a dedicated `ansible` user with passwordless sudo. The site playbook
then runs as that `ansible` user and does the actual install. Both the install
*and* the enable/start happen in the site stage — after a successful run every
service is running and set to start on boot.

## Install dependencies

```bash
ansible-galaxy collection install -r requirements.yml
```

`community.general` provides the pacman module the Arch playbooks use, and
`community.docker` brings up the control-plane stack.

## Configure (what to populate)

### 1. Per-environment variables (`inventories/<env>/group_vars/all/`)

Each environment is a self-contained inventory directory. Ansible loads the
`group_vars/all/` beside the inventory automatically, so selecting the
inventory selects the whole configuration:

```bash
ansible-playbook site/arch/site-arch.yml                        # stg (default)
ansible-playbook -i inventories/prod site/arch/site-arch.yml    # prod
```

`ansible.cfg` defaults to `inventories/stg` deliberately: a forgotten `-i`
hits staging rather than production.

To add an environment, copy the template and fill it in:

```bash
cp -r inventories/example inventories/<env>
```

The playbooks deliberately do **not** use `vars_files`. Play-level `vars_files`
outrank inventory `group_vars`, so a shared vars file would silently override
whatever the selected environment specifies — every host would get the same
control plane and the same secrets no matter which `-i` you passed. Keeping the
values in `group_vars` is what makes the environments actually separate.

| File | Fill in |
| --- | --- |
| `group_vars/all/common.yml` | `control_plane_host` — the address every agent ships to; `provision-vms.sh` fills this in for local VMs. `tailscale_authkey` (leave empty to install Tailscale without joining a tailnet). Optionally adjust `fluent_bit_tail_paths`, `fluent_bit_security_tail_paths`, and `audit_rules`. |
| `group_vars/all/arch.yml` | `ansible_admin_authorized_key` (the public key string for the bootstrap user). The Loki and Splunk endpoints derive from `control_plane_host`, so they need no editing unless your control plane is reached some other way. |
| `group_vars/all/ubuntu.yml` | `ansible_admin_authorized_key_file` (path to the public key), `loki_host`, `splunk_hec_host`, `splunk_hec_token`. `ubuntu_osquery_enabled` is `false` because upstream ships no ARM64 package. |
| `group_vars/all/control.yml` | `control_repo_owner` and `control_repo`. The dest and owning user default to the `ansible` admin account, so they usually need no editing. |
| `group_vars/all/secrets.yml` | `splunk_password`, `splunk_hec_token`, `grafana_admin_password`. `control_repo_gh_token` only for a private repo. Shared by the node and control-plane playbooks so both sides use the same HEC token. Give each environment its own values. |

Only `inventories/example/` is tracked in git; the real environment directories
hold live hosts, keys and secrets and are gitignored.

Splunk forwarding is skipped automatically on Ubuntu when `splunk_hec_host`/
`splunk_hec_token` are empty.

#### Where agents ship data: one port, service names

The control plane exposes **a single ingress port**. Every HTTP service behind
it is addressed by a service name in the path and routed by an nginx reverse
proxy, so `loki_host` and `splunk_hec_host` are the same host and the same
port:

| Destination | Endpoint |
| --- | --- |
| Loki | `http://<control-plane>/loki/api/v1/push` |
| Splunk HEC | `http://<control-plane>/services/collector` |

That is why `loki_port` and `splunk_hec_port` both default to `80` rather than
Loki's `3100` and Splunk's `8088` — those are container-internal ports the
proxy talks to, never reachable from an agent.

The HEC path is not under `/splunk/` like the Splunk UI is, because Fluent
Bit's `out_splunk` plugin hardcodes `/services/collector` and offers no way to
prefix it. The proxy routes that path directly to Splunk.

**No TLS.** `splunk_hec_tls` defaults to `false` and nothing verifies
certificates. Agents and the control plane share a private Tailscale network,
which provides transport encryption; adding TLS underneath it would mean
managing certs for a hop that is already encrypted. Splunk's HEC has its own
SSL turned off on the control-plane side for the same reason. If you ever run
this outside a trusted network, set `splunk_hec_tls: true`, set
`splunk_hec_tls_verify` (defaults to on) and terminate TLS at the proxy.

### 2. Hosts (`inventories/<env>/hosts.ini`)

Each environment has its own inventory. Set your hosts under the right group:

```bash
$EDITOR inventories/stg/hosts.ini
```

- Put Arch hosts under `[arch_hosts]` and Ubuntu hosts under `[ubuntu_hosts]`.
- `ansible_user=ansible` (the group default) is correct for the **site** stage.
  The **bootstrap** stage connects as an existing admin instead — override it
  per run with `-e ansible_user=...` (e.g. `-e ansible_user=azureuser`); you
  don't need a second inventory. Use `-e`, not `-u`: `ansible_user` set on a
  group outranks `-u` on the command line, so `-u` is silently ignored here.
- To reach hosts over Tailscale instead, keep a second environment directory
  (e.g. `inventories/stg-ts/`) whose `hosts.ini` uses MagicDNS names, and
  select it with `-i`.

Keep private keys outside the repo and readable only by you
(`chmod 600 ~/.ssh/your_key`). Do not commit keys.

## Run

`ansible.cfg` defaults to `inventories/stg`, so you only pass `-i` to target
another environment (`-i inventories/prod`). Bootstrap overrides the login user
with `-e ansible_user=...`.

### Arch Linux

```bash
# Stage 1 — as an existing privileged login
ansible-playbook bootstrap/arch/bootstrap-arch.yml -e ansible_user=your-existing-admin

# Stage 2 — as the ansible user
ansible-playbook site/arch/site-arch.yml
# or, over Tailscale:
ansible-playbook -i inventory.tailscale.ini site/arch/site-arch.yml
```

### Ubuntu

```bash
# Stage 1 — as the cloud image's default user
ansible-playbook bootstrap/ubuntu/bootstrap-ubuntu.yml -e ansible_user=azureuser

# Stage 2 — as the ansible user
ansible-playbook site/ubuntu/site-ubuntu.yml
# or, over Tailscale:
ansible-playbook -i inventory.tailscale.ini site/ubuntu/site-ubuntu.yml
```

### Control plane

Brings the whole control plane up on the host in `[control_hosts]`: installs
Docker and the GitHub CLI, clones the control-server deployment repo, renders
the stack's `.env` from `group_vars/all/secrets.yml`, registers every managed host as a
Prometheus scrape target, and starts the stack.

```bash
ansible-playbook site/control/site-control.yml
```

When it finishes, the proxy answers on `control_proxy_port` and Grafana,
Prometheus, Loki and Splunk are all reachable through it.

Re-running is safe. An existing checkout is updated with `git pull --ff-only`,
so a fast-forward conflict surfaces as a failure rather than silently
discarding edits made on the host. A changed `.env` bounces the stack so the
containers actually pick the new values up; an unchanged one leaves it running.

**Secrets.** `.env` holds the Splunk admin password and HEC token, is
gitignored, and never arrives with the clone — it is rendered on the host from
`group_vars/all/secrets.yml`. That file is also where the *agents* get their HEC token,
which is what guarantees both sides of the wire agree. A token that matches
nothing means Splunk rejects every event, and it fails quietly.

**Scrape targets.** `prometheus/targets/managed-hosts.yml` is generated from the
inventory, so every host in `[arch_hosts]` or `[ubuntu_hosts]` is scraped
without being registered by hand. Adding a node to the inventory and re-running
this play is all it takes. Prometheus is reloaded in place so the change lands
immediately rather than at the next file_sd refresh.

`gh` is used rather than a plain `git clone` so the same play works for a
private repo: set `control_repo_gh_token` in `group_vars/all/secrets.yml` and it
authenticates via `GH_TOKEN` in the task environment, which keeps the token out
of the host's `gh` config and out of the process list. A public repo needs no
token.

## Notes

- Prometheus, Loki, and Splunk HEC are external services that must already exist.
- Node exporter listens on port `9100` by default.
- The site playbook runs as a non-root `ansible` user with `become: true` so
  sudo logs show the source user.
- Custom apps are fine as long as they write logs to files Fluent Bit can read
  (see `fluent_bit_tail_paths`).

## Privilege audit

See [`docs/command-privileges.yml`](./docs/command-privileges.yml) for a
machine-readable map of which tasks run with sudo and which user initiates them.
