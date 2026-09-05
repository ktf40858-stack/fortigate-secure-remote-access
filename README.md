# FortiGate Secure Remote Access

Two remote-access designs on FortiGate (FortiOS): **SSL VPN** for roaming users and **IPsec
site-to-site** for a branch, both built least-privilege — access tied to identity and to the
specific resources a role needs, not a flat tunnel into the LAN. Backs the Fortinet NSE
certification with configuration a hiring manager can actually read.

> Configuration is expressed as FortiOS CLI. Designed against the FortiGate VM evaluation
> licence. No real PSKs, no real public IPs, no real user passwords — all `<PLACEHOLDER>`.

---

## What "secure" remote access means here

A remote-access VPN that drops the user onto the LAN with full reachability is a convenience
feature, not a security control — it means a compromised laptop at a coffee shop is a host
inside your network. This repo builds the opposite:

- **The tunnel authenticates the user**, ideally with MFA, not just a shared secret.
- **Firewall policy after the tunnel is least-privilege by role** — a contractor reaches the
  one app they need, an employee reaches more, neither reaches everything.
- **Split tunnelling is a deliberate decision**, documented, not a default.
- **The branch tunnel carries only the subnets that must cross it**, phase-2 selectors scoped.

## The two designs

### 1. SSL VPN — roaming users

```
   remote user (FortiClient)  ==TLS tunnel==>  FortiGate  --role policy-->  internal apps
        MFA (FortiToken)                        SSL VPN         |
                                                portal          +-- employees: app + file server
                                                                +-- contractors: one app only
```

Full config: [`config/ssl-vpn.conf`](config/ssl-vpn.conf) · design:
[`docs/01-ssl-vpn-design.md`](docs/01-ssl-vpn-design.md)

### 2. IPsec site-to-site — a branch

```
   Branch FortiGate  ==IKEv2/IPsec==>  HQ FortiGate
   10.50.0.0/24                        10.30.0.0/24 (servers only, scoped selectors)
```

Full config: [`config/ipsec-site-to-site.conf`](config/ipsec-site-to-site.conf) · design:
[`docs/02-ipsec-design.md`](docs/02-ipsec-design.md)

## What is in here

| Path | Contents |
|---|---|
| [`config/ssl-vpn.conf`](config/ssl-vpn.conf) | SSL VPN portals, per-role, with split tunnel and firewall policy |
| [`config/ipsec-site-to-site.conf`](config/ipsec-site-to-site.conf) | IKEv2 phase-1/phase-2 both ends, scoped selectors, policy |
| [`config/access-profiles.md`](config/access-profiles.md) | The role-to-access matrix, the heart of the least-privilege design |
| [`docs/01-ssl-vpn-design.md`](docs/01-ssl-vpn-design.md) | Why per-role portals, split-tunnel trade-off, MFA |
| [`docs/02-ipsec-design.md`](docs/02-ipsec-design.md) | IKEv2 choices, PFS, selector scoping, the DPD and rekey story |
| [`docs/03-hardening.md`](docs/03-hardening.md) | Cipher choices, what to disable, the common misconfigurations |
| [`topology/lab-setup.md`](topology/lab-setup.md) | Building it on the FortiGate VM eval |

## The principles it demonstrates

1. **Identity before reachability.** The tunnel is not the access — it is the authenticated
   pipe. Access is the firewall policy that runs *after* the tunnel, keyed to the user's group.
2. **Least privilege per role.** Employees and contractors get different portals and different
   policy. No single "VPN users can reach the LAN" rule.
3. **Scope the crypto and the selectors.** IPsec phase-2 selectors carry only the subnets that
   must cross; PFS on; modern ciphers only; legacy IKEv1/aggressive mode off.
4. **MFA on remote access is not optional.** A remote tunnel protected by a password alone is
   one phishing email from being an attacker's tunnel.

## Author

Kodjo Apedoh — Network & Cloud Security · Arlington, VA
CCNA · **Fortinet NSE** · Palo Alto SASE & Cloud Security · [LinkedIn](https://www.linkedin.com/in/kodjo-apedoh-03030990/) · [Other labs](https://github.com/ktf40858-stack)

## License

MIT — see [LICENSE](LICENSE). Lab and educational use only.
