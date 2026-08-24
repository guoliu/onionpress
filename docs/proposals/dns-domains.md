# DRAFT — proposal issue: a DNS domain alongside the onion name and onion address

> Status: reserve draft. Open only after engagement in the umbrella issue.

**Title: Proposal: a clearnet domain alongside the `.onion` for published sites**

onionpress.org itself lives a dual life: a normal DNS domain for everyone, and an onion address for people who need it. Sites published *through* OnionPress currently only get the second half. For most authors the `.onion` is the resilient mirror, not the front door — they still need a `example.com` their readers can reach. In moss, a user can already purchase and set up a domain in a few clicks; the gap is a blessed story for how that domain and the OnionPress machine relate.

Two things make this more than a convenience:

1. **Archive fidelity.** We measured that Save Page Now over `.onion` captures zero embeds and cannot capture non-HTML resources at all — so archiving under a clearnet URL is what makes the Wayback copy of a published site *complete*, not just reachable.
2. **Fallback that works in any browser.** With captures under `example.com`, the fallback URL (`web.archive.org/web/2/example.com/…`) opens without Tor. The domain's edge — a CDN worker, the publisher's deploy host, or plain DNS failover with a low TTL — can health-check the origin and redirect to the newest snapshot when the machine is offline: the same role the takeover already plays for the onion, applied at the DNS layer.

Questions I'd like your read on:

1. Is dual-homing in scope for OnionPress itself? Onion-Location headers already point clearnet visitors to the onion — should the reverse exist: a supported path from a registered domain to a machine running OnionPress?
2. The naming CLI (in the receiver PR) registers memorable onion names headlessly. Should the same flow optionally drive DNS (a registrar-style API, or documented A/AAAA + reverse-proxy guidance), or is DNS strictly the publisher app's problem?
3. Certificates and NAT: for a home machine, the realistic clearnet path is a small tunnel/edge (user-owned VPS or a service). Would you want OnionPress to document/bless one pattern, or stay onion-only and leave clearnet to integrators?
