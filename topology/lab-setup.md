# Lab setup — FortiGate VM evaluation

## What you need

| | |
|---|---|
| Firewall | FortiGate VM64 evaluation image (Fortinet support portal) — runs on ESXi/KVM/Workstation/Hyper-V |
| Eval licence | The 15-day VM eval, or the free perpetual eval with throughput caps |
| For the site-to-site lab | a second FortiGate VM as the branch |
| Endpoints | a couple of small VMs (an internal app host, a file host) + FortiClient on a "remote" VM |
| Cost | none for the eval period |

Alternatives: FortiGate VMs on the **AWS/Azure marketplace** (pay-as-you-go, real public IPs for
a genuine internet-facing SSL VPN test), or the **Fortinet FNDN / demo labs** through the NSE
training program. The CLI is identical.

## SSL VPN lab

1. Import `config/ssl-vpn.conf`, replacing `<PLACEHOLDER>` secrets and the interface names to
   match your VM.
2. Install FortiClient on a VM outside the FortiGate's LAN (a second host-only network standing
   in for "the internet").
3. Connect as `employee1`, then as `contractor1`, and prove the difference:

```
# as employee1 - both work
curl -k https://10.30.0.10       # app server  -> OK
smbclient -L //10.30.0.30        # file server -> OK

# as contractor1 - only the app works
curl -k https://10.30.0.10       # app server  -> OK
smbclient -L //10.30.0.30        # file server -> DENIED (no route, no policy)
ping 10.30.0.50                  # anything else -> DENIED
```

The contractor being unable to reach the file server, while the employee can, over the same VPN,
is the demonstration. Confirm the denies in **Log & Report > Forward Traffic**.

## IPsec site-to-site lab

1. Two FortiGate VMs, one as HQ (198.51.100.1 in the lab), one as branch (203.0.113.1), on a
   shared "WAN" segment.
2. Import the HQ half of `config/ipsec-site-to-site.conf` on one, the branch half on the other.
   The PSK must match.
3. Bring it up and verify:

```
diagnose vpn ike gateway list      # phase 1 established
diagnose vpn tunnel list           # phase 2 SA + selectors - CHECK the selectors are scoped
```

4. From a host on the branch LAN, reach an HQ server and confirm the scoping:

```
ping 10.30.0.10        # HQ server -> OK, crosses the tunnel
ping 10.10.0.20        # HQ user subnet -> DENIED, not in the selectors or policy
```

The branch reaching HQ's servers but not HQ's users is the proof that the selectors and policy
scoped the tunnel instead of bridging the two LANs.

## Note on the eval throughput cap

The free perpetual eval caps throughput and disables some UTM signature updates. It is fine for
proving the configuration and the policy behaviour, which is what this lab is about. For a
performance test, the time-limited full eval or the marketplace PAYG image removes the cap.
