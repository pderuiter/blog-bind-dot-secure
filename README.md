# blog-bind-dot-secure

Companion code for the blog post
[Securing DNS Traffic with DNS over TLS (DoT) on BIND](https://blog.bsdserver.nl/securing-dns-traffic-with-dns-over-tls-dot-on-bind/).

It builds the post's "Complete Production Configuration": a BIND resolver that answers plain DNS on
port 53 and DNS over TLS on port 853.

- `tls dot-server` block with `cert.pem`, `key.pem` and `dhparam.pem` in `/etc/bind/tls`
  (a self-signed certificate by default, or your own certificate files, for example from Let's Encrypt)
- TLS 1.2/1.3 only, ECDHE cipher suites, `prefer-server-ciphers`, session tickets
- `tcp-clients 2000` and `tcp-keepalive-timeout 600` (units of 100 ms, so 60 seconds)
- `version none`, `hostname none`, `acl trusted` for query and recursion access
- `tls_log` channel for the `network` category in `/var/log/named/tls.log`
- optional: forward all queries to Cloudflare over DoT (`tls upstream-dot` with `remote-hostname
  "cloudflare-dns.com"`), which needs BIND 9.20
- optional: the post's nftables ruleset with the DoT connection rate limit
- `kdig` (knot-dnsutils) for testing

## Requirements

- Terraform 1.6+ and a Proxmox VE cluster with an API token (only if you want Terraform to create the VM)
- Ansible core 2.16+ on your workstation
- A target running Ubuntu 24.04 (the post uses the Debian layout in `/etc/bind`)
- For DoT forwarding: BIND 9.20, installed from the ISC packages (`ppa:isc/bind`) with
  `bind_install_source: isc`. Ubuntu 24.04 ships BIND 9.18, which has the DoT listener but cannot
  forward over TLS.

## Quick start

```bash
# 1. Create the VM (optional, skip if you already have a server)
cd terraform
cp terraform.tfvars.example terraform.tfvars   # fill in your values
terraform init
terraform apply
terraform output -raw ansible_inventory > ../ansible/inventory.yml

# 2. Configure BIND
cd ../ansible
ansible-galaxy collection install -r requirements.yml
ansible-playbook site.yml

# 3. Run the post's checks (plain DNS, dig +tls, kdig +tls, and the forwarding/nftables checks when enabled)
ansible-playbook ../tests/verify.yml
```

Without Terraform, copy `ansible/inventory.example.yml` to `ansible/inventory.yml` and edit it.

To enable DoT forwarding and the firewall rules, set in `ansible/group_vars/all/vars.yml`:

```yaml
bind_install_source: isc
bind_dot_forwarding: true
bind_dot_nftables: true
```

`bind_dot_nftables: true` replaces `/etc/nftables.conf` with the post's ruleset and enables the
`nftables` service. Restarting that service flushes the whole ruleset, so merge it with your own rules
first if the server already has a firewall.

## Variables

| Variable | Default | Description |
| --- | --- | --- |
| `bind_dot_listen_v4` | 127.0.0.1 and the default IPv4 address | Addresses for the port 53 and port 853 listeners |
| `bind_dot_listen_v6` | `::1` | IPv6 addresses for both listeners |
| `bind_dot_trusted_networks` | localhost, 192.168.0.0/16, 10.0.0.0/8 | Members of `acl trusted` |
| `bind_dot_cert_source` | `selfsigned` | `selfsigned` generates a key and certificate on the server, `files` copies yours |
| `bind_dot_cert_file` / `bind_dot_key_file` | `""` | Certificate (full chain) and key on the controller, for `files` |
| `bind_dot_server_name` | `dns.example.com` | Name in the self-signed certificate |
| `bind_dot_dhparam_bits` | `2048` | Size of the DH parameters |
| `bind_dot_tcp_keepalive_timeout` | `600` | `tcp-keepalive-timeout`, in units of 100 ms |
| `bind_dot_forwarding` | `false` | Forward all queries over DoT (BIND 9.20+) |
| `bind_dot_forwarders` | 1.1.1.1, 1.0.0.1 | DoT upstreams; all must present `bind_dot_upstream_hostname` |
| `bind_dot_upstream_hostname` | `cloudflare-dns.com` | `remote-hostname` checked against the upstream certificates |
| `bind_dot_forward_mode` | `only` | `forward only` or `forward first` |
| `bind_dot_nftables` | `false` | Install the post's nftables ruleset |
| `bind_install_source` | `distro` | `distro` (BIND 9.18) or `isc` (ppa:isc/bind, BIND 9.20) |

## Layout

- `terraform/` creates Ubuntu 24.04 VMs on Proxmox VE (bpg/proxmox provider) and outputs an Ansible inventory
- `ansible/roles/bind_install` installs BIND (distribution or ISC packages)
- `ansible/roles/bind_dot` writes the TLS material, the DoT configuration and the optional nftables rules
- `tests/verify.yml` runs the functional checks from the post

## License

MIT
