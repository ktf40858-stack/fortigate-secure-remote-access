# Hardening and common misconfigurations

The design is only as good as the things it turns off. This is the list, and the reasoning.

## Cipher and protocol posture

| Turn off | Why |
|---|---|
| TLS 1.0 / 1.1 on the SSL VPN portal | Deprecated, broken record protocol; the portal is internet-facing |
| IKEv1 / aggressive mode | Aggressive mode transmits a hash of the PSK that an offline attacker can crack |
| DES, 3DES | 3DES is retired (Sweet32); DES is trivially broken |
| MD5, SHA-1 for integrity | Collision-broken; use SHA-256+ |
| DH groups 1, 2, 5 | Too small; 14 is the floor, 19/20 preferred |

## The misconfigurations that show up in every audit

**1. Flat SSL VPN policy (`ssl.root -> LAN, any/any`).** The most common finding. It makes the
VPN a bridge onto the whole network. Fixed here by per-group, per-resource policies with no
LAN-wide rule.

**2. Wide IPsec selectors (`0.0.0.0/0`).** A site-to-site tunnel that bridges both LANs entirely.
Fixed by scoping phase-2 selectors to the subnets that must communicate.

**3. Self-signed portal certificate left in place.** Users are trained to click through the
browser warning, which is the exact warning that would flag a real MITM. Replace `Fortinet_SSL`
with a CA-signed cert.

**4. No MFA on remote access.** Remote access is the most-phished credential. Password-only is
one email from compromise. FortiToken or RADIUS/SAML MFA on every remote account.

**5. SSL VPN on the default port 10443 with no geo/IP restriction and no lockout.** The portal
gets brute-forced from the internet within hours of going live. Mitigations:

```
config vpn ssl settings
    set login-attempt-limit 3
    set login-block-time 60
end
# and a geo-IP / trusted-source restriction on the portal's local-in policy where the
# user population is geographically bounded
```

**6. Administrative access exposed on the WAN interface.** The management GUI/SSH should never
be reachable from `port1`. Disable admin access on the untrust interface entirely:

```
config system interface
    edit "port1"
        set allowaccess ping          # no https/ssh/http admin on the WAN side
    next
end
```

## Logging

Every firewall policy in this repo sets `logtraffic all`. On remote access specifically, the
logs are the only visibility into who connected, from where, and what they reached — there is no
physical port to look at. A remote-access policy without full logging is a control you cannot
audit and an incident you cannot reconstruct.

## The one-line summary for each

- SSL VPN: **the firewall policy is the access control, not the tunnel** — keep it per-role.
- IPsec: **the selectors are the boundary** — scope them, never `0.0.0.0/0` for a branch.
- Both: **MFA, modern ciphers, CA-signed cert, full logging** — the four that are non-negotiable.
