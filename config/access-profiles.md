# Role-to-access matrix

This is the heart of the design. The VPN tunnel is just an authenticated pipe; **this table is
the access control**, enforced by the identity-based firewall policies after the tunnel.

## The matrix

| Role | Portal | May reach | On | May NOT reach |
|---|---|---|---|---|
| Employee | portal-employees | app-server (10.30.0.10), file-server (10.30.0.30) | HTTPS, SMB | the rest of the LAN, management, other user subnets |
| Contractor | portal-contractors | app-server (10.30.0.10) only | HTTPS | the file server, the LAN, everything else |
| Branch (site-to-site) | — | hq-servers (10.30.0.0/24) | ALL | the HQ user subnets, HQ management |

## Why role, not just "VPN user"

The tempting shortcut is one rule: `ssl.root -> LAN, allow, VPN-users`. It is one line and it
is a flat tunnel — every remote user, employee or contractor, compromised or not, can reach the
entire internal network. That single rule is the most common remote-access finding in an audit.

Instead, access is bound to the user's **group** on the firewall policy itself
(`set groups "grp-contractors"`). Two things follow:

1. A contractor whose laptop is compromised gives the attacker reach to exactly one application.
   The blast radius is the contractor's job, not the network.
2. Access is provisioned by group membership. Onboarding a contractor is adding them to
   `grp-contractors`; there is no per-user rule to write and none to forget to remove when they
   leave. Deprovisioning is removing them from the group — access disappears everywhere at once.

## Split tunnelling as a per-role decision

| Role | Split tunnel | Reasoning |
|---|---|---|
| Employee | on, routes corp subnets only | Their own internet traffic does not transit HQ — less load, less liability. They are trusted and managed. |
| Contractor | on, routes the one app subnet only | They should touch as little as possible; their general internet is none of HQ's business and none of HQ's risk. |

The alternative — **full tunnel**, all traffic through HQ — is the right call when the remote
endpoint is untrusted and you want HQ's web filtering and inspection applied to everything it
does, at the cost of backhauling all its internet traffic. Both are defensible; the point is
that it is a **decision with a rationale**, recorded here, not a default nobody examined. An
interviewer who asks "split or full tunnel?" is testing whether you know it is a trade-off.

## What enforces each column

| Column | Enforced by |
|---|---|
| Which portal | `config vpn ssl settings > authentication-rule`, group -> portal |
| What is routed into the tunnel | portal `split-tunneling-routing-address` |
| What is actually reachable | the identity-based `firewall policy` rules (10, 11) |
| Threat inspection on allowed flows | `utm-status`, `av-profile`, `ips-sensor` on each policy |

Note that routing and reachability are enforced separately and both matter. Split-tunnel
routing decides what the client *sends* into the tunnel; the firewall policy decides what it is
*allowed to reach*. A permissive firewall policy behind a tight split tunnel is still a hole —
the client can be reconfigured, the firewall cannot be from the client side. The firewall
policy is the real boundary.
