# telegraf-setup-playbook

Ansible playbook that configures managed Arch Linux servers to send telemetry to external systems:

1. Installs Tailscale
2. Joins the host to your tailnet when an auth key is provided
3. Installs Prometheus node exporter
4. Installs Fluent Bit
5. Installs auditd
6. Installs osquery
7. Installs Docker
8. Enables node exporter so an external Prometheus server can scrape metrics from the host
9. Forwards host logs to an external Loki instance
10. Forwards security-related journal events and audit logs to an external Splunk HEC endpoint

## Usage

For a clean privilege split, bootstrap a dedicated `ansible` admin user first, then run the main playbook as that user with sudo.

1. Copy [`group_vars/all.yml.example`](./group_vars/all.yml.example) to `group_vars/all.yml`.
2. Copy [`group_vars/arch_hosts.yml.example`](./group_vars/arch_hosts.yml.example) to `group_vars/arch_hosts.yml`.
3. Fill in `ansible_admin_authorized_key` and the environment-specific settings in `group_vars/arch_hosts.yml`.
4. Run the bootstrap playbook as an existing privileged account:

```bash
ansible-playbook -u your-existing-admin bootstrap-ansible.yml
```

5. Copy `inventory.example.ini` to `inventory.ini` and set the target host addresses.
6. Run the main playbook:

```bash
ansible-playbook site.yml
```

If you keep the default `ansible.cfg`, you can omit `-i inventory.ini` and just run `ansible-playbook site.yml`.

`group_vars/all.yml` holds shared defaults. `group_vars/arch_hosts.yml` holds the host- or environment-specific values.

This repo does not provision Prometheus, Loki, Splunk, or any other control-plane service. It only configures the servers that send data.

This playbook targets Arch Linux hosts only.

## Notes

- Prometheus, Loki, and Splunk HEC are external services that must already exist.
- This playbook installs the client-side exporter only.
- The main playbook should be executed as a non-root `ansible` SSH user with `become: true` so sudo logs show the source user.
- Node exporter listens on port `9100` by default.
- Fluent Bit tails the configured log paths and forwards them to Loki over the tailnet.
- Fluent Bit also forwards selected security-related systemd journal events and audit logs to Splunk HEC.
- `auditd` is seeded with a small high-signal rule set for identity, sudo, SSH, and polkit changes.
- `osquery` is enabled as a host visibility agent for later investigation and detection work.
- Custom apps are fine as long as they write logs to files that Fluent Bit can read.
- `ansible.cfg` sets the default SSH user to `ansible` and enables sudo by default.

## Privilege Audit

See [`docs/command-privileges.yml`](./docs/command-privileges.yml) for a machine-readable map of which tasks run with sudo and which user initiates them.
