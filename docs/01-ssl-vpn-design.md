# SSL VPN design

## The tunnel is not the access

The single idea this whole design rests on: an SSL VPN tunnel authenticates a user and gives
them an IP on the inside. It does **not**, by itself, decide what they can reach. That decision
is the firewall policy that runs after the tunnel terminates. Conflating the two — "they got
through the VPN, so they're in" — is what produces flat remote-access that an audit flags and an
attacker loves.

So the design has three separable layers, and each is tightened on its own:

1. **Authentication** — who are you (password + MFA)
2. **Portal** — what gets routed into your tunnel (split-tunnel address list)
3. **Firewall policy** — what you are actually permitted to reach, by group

## Per-role portals

Employees and contractors get different portals because they have different trust and different
needs. The contractor portal routes exactly one subnet into the tunnel; the employee portal
routes two. This is defence in depth against a misconfiguration: even if the firewall policy
were wrong, the contractor's client would not have a route to anything but the one app.

Mapping is by group, in `authentication-rule` — a user lands in the portal their group dictates,
automatically. Add a user to `grp-contractors` and they get the contractor portal with no
per-user setup.

## MFA is not optional

A remote-access tunnel protected by a password alone is one phishing email away from being an
attacker's tunnel — and remote access is the single most phished credential there is, because it
is internet-facing by definition. FortiToken (FortiGate's built-in TOTP) or an external RADIUS/
SAML MFA is bound to each account:

```
config user local
    edit "employee1"
        set two-factor fortitoken
        set fortitoken <SERIAL>
    next
end
```

The design treats MFA as a baseline requirement, not a hardening option. A remote-access repo
that shows password-only auth is demonstrating the wrong thing.

## Split tunnel: the trade-off, stated

| | Split tunnel | Full tunnel |
|---|---|---|
| Client internet traffic | goes direct, not through HQ | backhauled through HQ |
| HQ inspects client's general browsing | no | yes |
| Load on HQ link | low | high |
| Right when | endpoint is managed/trusted, you only care about corp resource access | endpoint is untrusted, you want HQ's filtering on everything |

This lab uses split tunnel because the resources are specific and the point is *least-privilege
access to them*, not policing the user's whole internet. That is a choice with a reason, and the
reason is what matters — the wrong answer in an interview is not "split" or "full", it is not
knowing why.

## Modern TLS only

```
set ssl-min-proto-ver tls1-2
set ciphersuite TLS-AES-256-GCM-SHA384 TLS-AES-128-GCM-SHA256
```

TLS 1.0 and 1.1 are disabled, and the cipher list is restricted to AEAD suites. The SSL VPN
portal is internet-facing, so its TLS posture is part of the perimeter, not an afterthought —
the same scanners that rate a website's TLS rate this. And the certificate is CA-signed in
production, not the built-in self-signed `Fortinet_SSL`, which trains users to click through the
warning that is supposed to protect them.

## Where SSL VPN fits versus ZTNA

Fortinet, like the rest of the industry, is moving remote access from SSL VPN toward **ZTNA** —
per-application access brokered on identity and device posture, with no network-level tunnel at
all. SSL VPN gives a user an IP on the inside; ZTNA gives a user a connection to one application
and nothing else, re-checked continuously. This lab builds the SSL VPN design well *and* names
its successor, because "we still run SSL VPN, here is how we'd move to ZTNA" is a more credible
thing to say than pretending SSL VPN is the endpoint. That successor is the subject of the
[zero-trust-sase-architecture](https://github.com/ktf40858-stack/zero-trust-sase-architecture) repo.
