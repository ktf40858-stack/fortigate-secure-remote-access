# IPsec site-to-site design

## What the design commits to, and why

| Choice | Value | Why |
|---|---|---|
| IKE version | IKEv2 | IKEv1 aggressive mode leaks the PSK hash to a passive listener; IKEv2 has no aggressive mode, cleaner rekey, and MOBIKE |
| Phase 1 encryption | AES-256-GCM / AES-256 + SHA-256 | No DES, 3DES, MD5 or SHA-1 — all broken or deprecated |
| DH group | 14 (2048-bit MODP) minimum | Group 1/2/5 are too small; 14 is the floor, 19/20 (ECP) better |
| PFS | enabled | Compromise of one session key does not expose past or future sessions |
| Phase 2 lifetime | 3600 s | Frequent rekey limits the data encrypted under any one key |
| DPD | on-idle | Detects a dead peer and tears the tunnel down instead of blackholing |

None of these are defaults you can leave alone. FortiOS will happily negotiate weaker proposals
for interoperability; pinning the strong ones is the work.

## The selectors are the security boundary

The most important lines in the config are the phase-2 selectors:

```
set src-subnet 10.30.0.0 255.255.255.0     # HQ offers ONLY its server subnet
set dst-subnet 10.50.0.0 255.255.255.0     # to ONLY the branch subnet
```

A site-to-site tunnel is often built with `0.0.0.0/0` selectors — "just connect the two sites" —
which bridges the entire two networks and makes a compromise at the small branch a compromise of
all of HQ. Scoping the selectors to the subnets that actually need to talk means the branch can
reach HQ's servers and nothing else. The tunnel is not a bridge between LANs; it is a controlled
path between two named subnets.

The firewall policy then narrows it further: the branch reaches `hq-servers`, not all of
10.30.0.0/24 if that subnet later grows to hold more than servers. Selectors and policy are two
independent controls on the same boundary, and both are scoped.

## The rekey and DPD story

Two failure modes an interviewer probes:

- **Rekey mismatch.** If the two ends disagree on phase-2 lifetime or PFS, the tunnel comes up,
  passes traffic, then drops when the first rekey fails — an intermittent outage that looks like
  a flaky link. Both ends here use the same lifetime and both enable PFS with the same DH group.
- **Dead peer, live tunnel.** Without DPD, if the remote gateway reboots, the local end keeps
  sending into a tunnel that no longer exists — traffic is blackholed until the SA times out.
  `set dpd on-idle` probes an idle peer and tears down a dead SA so traffic can reroute or the
  tunnel can rebuild.

## Verifying it

```
# phase 1 up?
diagnose vpn ike gateway list

# phase 2 SA installed, and which selectors?
diagnose vpn tunnel list

# is traffic actually using the tunnel (not falling to a default route)?
diagnose sniffer packet any 'host 10.50.0.10' 4
```

The selector check in `diagnose vpn tunnel list` is the one that catches the classic mistake:
the tunnel is up, but the selectors are `0.0.0.0/0` because someone left them wide, and the
"secure branch tunnel" is actually bridging everything. Reading `src`/`dst` in that output and
confirming they match the intended subnets is the proof the scoping worked.
