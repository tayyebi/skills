---
name: testing-network
description: Use when verifying network reachability and connectivity for a provisioned instance — covers IP assignment and format, ICMP reachability and sustained packet loss, a port-state matrix (expected-open vs. expected-closed), forward and reverse DNS resolution, MTU/fragmentation, routing sanity, IPv6 parity, and — most importantly — correlating network reachability with control-plane state and billing/metering so an instance is never billed while it is actually unreachable.
---

# Network Reachability Testing

Confirm that what the control plane says about an instance ("running," "has this IP") matches what the network actually shows, and that billing tracks the latter rather than blindly trusting the former. The highest-value check in this space isn't any single ping or port scan — it's the correlation between reachability and metering.

## When to use

- Any provisioned instance (VM, container, managed service) that gets a network-routable address you're expected to verify.
- Whenever you need to distinguish "the control plane thinks it's running" from "it is actually reachable" — these silently diverge more often than either layer's owners expect.
- Pairs with `testing-ssh` (owns auth/cipher depth once port 22 is confirmed open — this skill only does a passive banner check as corroboration), `testing-lifecycle` (state-transition polling this skill's checks hang off of), and `testing-region-coverage` (the same checks run across a matrix of regions).

## Core pattern

**Validate the assigned address itself.** Confirm an address is present, correctly formatted, and — critically — actually public/routable rather than an internal/private-range address leaking out of an API that should only ever hand back externally reachable addresses.

**Corroborate reachability with more than one signal.** ICMP alone is a weak proof of health: it can be rate-limited or filtered even on a perfectly healthy instance, and it can occasionally succeed against a host whose actual service ports are down. Pair a ping-loss measurement (both a short burst and a longer sustained run, since transient loss and sustained loss mean different things) with a TCP-level check on the port(s) the instance is actually expected to serve.

**Build a port-state matrix, not ad hoc checks.** Enumerate the ports that should be open (the instance's actual service ports) and a fixed list that should be closed (legacy/insecure services: Telnet, FTP, unauthenticated database ports, SMB, etc.) and assert both sides of the matrix in one pass. An "expected closed" list catches a hardening regression; an "expected open" list catches a firewall misconfiguration blocking the feature that's supposed to work.

**Check DNS both directions.** If the product assigns a hostname, forward resolution must return the same address the API reports. Reverse (PTR) resolution is commonly optional — treat its absence as informational, not a failure, but if present, assert it doesn't leak an obviously internal name.

**Probe MTU when it matters.** For anything sensitive to fragmentation (encapsulated traffic, certain VPN/tunnel setups), a quick DF-bit ping sweep at a few payload sizes establishes the largest unfragmented payload the path supports — cheap insurance against a class of bugs that only shows up under real traffic, not synthetic pings.

**Correlate network state against control-plane state, explicitly.** This is the check that matters most: when the control plane reports "running," the instance should be ping- and port-reachable; when it reports "stopped," it should not be. Any divergence — reachable-while-stopped or unreachable-while-running — is the actual bug signature to hunt for, and it's invisible if you only test network and control-plane state separately.

**Chase every "unreachable while it shouldn't be" finding into the billing ledger.** Whenever an instance that the control plane reports as active turns out to be unreachable, immediately check whether usage/metering entries are still accruing for it. An instance customers can't use but are still being charged for is a materially worse bug than the reachability failure alone — always pair the two checks.

## Pitfalls

- **Trusting ICMP alone as a health signal.** Rate-limiting and filtering produce false negatives; pair it with a TCP-level check.
- **Ad hoc port checks instead of an explicit open/closed matrix.** Without a fixed "must be closed" list, a hardening regression (an unintentionally exposed database or management port) can ship unnoticed.
- **Testing network and control-plane state in isolation.** The bug that matters is the *divergence* between them, which only appears when you check both together.
- **Finding "unreachable while active" and not checking billing.** The reachability bug and the billing bug are two different severities; always check both.
- **Assuming IPv6 parity without testing it** when the product claims dual-stack support — the IPv4 path working proves nothing about IPv6.
- **Treating PTR absence as a failure.** Reverse DNS is commonly optional; don't over-assert on it.

## Checklist

- [ ] Assigned address present, correctly formatted, and actually public/routable
- [ ] ICMP reachability checked at both a short burst and a sustained run, with a loss threshold
- [ ] TCP-level check on the instance's real service port(s) as a second reachability signal
- [ ] Port-state matrix asserted: expected-open ports open, a fixed expected-closed list actually closed
- [ ] Forward DNS (if a hostname is assigned) matches the API-reported address; PTR checked if present, not required
- [ ] MTU/fragmentation probed if the product's traffic pattern is fragmentation-sensitive
- [ ] Network reachability explicitly correlated against control-plane state (running ⇒ reachable, stopped ⇒ not)
- [ ] Every "unreachable while supposedly active" finding cross-checked against the billing/metering ledger
- [ ] IPv6 parity tested independently when dual-stack is claimed
