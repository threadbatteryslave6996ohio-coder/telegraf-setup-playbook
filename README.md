# telegraf-setup-playbook

Ansible playbook that:

1. Installs Tailscale
2. Joins the host to your tailnet when an auth key is provided
3. Installs Telegraf
4. Installs Docker
5. Configures Telegraf to send metrics to a remote InfluxDB v2 endpoint

## Usage

Edit the variables in [`site.yml`](./site.yml), then run:

```bash
ansible-playbook -i your-inventory.ini site.yml
```

Required Telegraf variables:

- `telegraf_output_url`
- `telegraf_output_token`
- `telegraf_output_org`
- `telegraf_output_bucket`

Optional Tailscale variable:

- `tailscale_authkey`

This playbook targets Arch Linux hosts only.
