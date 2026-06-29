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
inventory.ini             single inventory for every playbook
inventory.tailscale.ini   same hosts, reached over Tailscale
  *.ini.example           templates — copy, then fill in

vars/                     shared configuration (the "env" files)
  common.yml              values used by every playbook, both OSes
  arch.yml                Arch-only values (packages, Loki/Splunk, SSH key)
  ubuntu.yml              Ubuntu-only values
  *.yml.example           templates — copy, then fill in

bootstrap/                stage 1: create the dedicated `ansible` admin user
  arch/   bootstrap-arch.yml
  ubuntu/ bootstrap-ubuntu.yml

site/                     stage 2: install + enable + start the services
  arch/   site-arch.yml
  ubuntu/ site-ubuntu.yml

docs/command-privileges.yml   machine-readable map of which tasks use sudo
ansible.cfg                    defaults (sudo on; default inventory = inventory.ini)
```

One inventory serves both stages. Each playbook targets its own group
(`arch_hosts` or `ubuntu_hosts`), and `ansible_user` is the steady-state
`ansible` account. Bootstrap runs once as an existing admin instead — you
override the user per run with `-u` (see below).

**Bootstrap vs. site.** Bootstrap runs once as an existing privileged login and
creates a dedicated `ansible` user with passwordless sudo. The site playbook
then runs as that `ansible` user and does the actual install. Both the install
*and* the enable/start happen in the site stage — after a successful run every
service is running and set to start on boot.

## Configure (what to populate)

### 1. Shared variable files (`vars/`)

Copy each example, then edit:

```bash
cp vars/common.yml.example vars/common.yml
cp vars/arch.yml.example   vars/arch.yml      # if managing Arch hosts
cp vars/ubuntu.yml.example vars/ubuntu.yml    # if managing Ubuntu hosts
```

Playbooks load `common.yml` first, then the OS file, so OS values override
shared ones.

| File | Fill in |
| --- | --- |
| `vars/common.yml` | `tailscale_authkey` (leave empty to install Tailscale without joining a tailnet). Optionally adjust `fluent_bit_tail_paths`, `fluent_bit_security_tail_paths`, and `audit_rules`. |
| `vars/arch.yml` | `ansible_admin_authorized_key` (the public key string for the bootstrap user), `loki_host`, `splunk_hec_host`, `splunk_hec_token`. Adjust `loki_port`/`loki_uri`/`splunk_*` to match your endpoints. |
| `vars/ubuntu.yml` | `ansible_admin_authorized_key_file` (path to the public key), `loki_host`, `splunk_hec_host`, `splunk_hec_token`. `ubuntu_osquery_enabled` is `false` because upstream ships no ARM64 package. |

Splunk forwarding is skipped automatically on Ubuntu when `splunk_hec_host`/
`splunk_hec_token` are empty.

### 2. Inventory (`inventory.ini`)

There is one inventory at the project root. Copy the example and set your hosts
under the right group:

```bash
cp inventory.ini.example inventory.ini
```

- Put Arch hosts under `[arch_hosts]` and Ubuntu hosts under `[ubuntu_hosts]`.
- `ansible_user=ansible` (the group default) is correct for the **site** stage.
  The **bootstrap** stage connects as an existing admin instead — override it
  per run with `-u` (e.g. `-u azureuser`); you don't need a second inventory.
- `inventory.tailscale.ini` is the same inventory but with Tailscale MagicDNS
  names; select it with `-i inventory.tailscale.ini` once a host has joined the
  tailnet.

Keep private keys outside the repo and readable only by you
(`chmod 600 ~/.ssh/your_key`). Do not commit keys.

## Run

`ansible.cfg` defaults to `inventory.ini`, so you only pass `-i` to switch to
the Tailscale inventory. Bootstrap overrides the login user with `-u`.

### Arch Linux

```bash
# Stage 1 — as an existing privileged login
ansible-playbook bootstrap/arch/bootstrap-arch.yml -u your-existing-admin

# Stage 2 — as the ansible user
ansible-playbook site/arch/site-arch.yml
# or, over Tailscale:
ansible-playbook -i inventory.tailscale.ini site/arch/site-arch.yml
```

### Ubuntu

```bash
# Stage 1 — as the cloud image's default user
ansible-playbook bootstrap/ubuntu/bootstrap-ubuntu.yml -u azureuser

# Stage 2 — as the ansible user
ansible-playbook site/ubuntu/site-ubuntu.yml
# or, over Tailscale:
ansible-playbook -i inventory.tailscale.ini site/ubuntu/site-ubuntu.yml
```

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
