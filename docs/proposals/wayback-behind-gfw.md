# DRAFT — proposal issue: making the Wayback fallback usable behind the GFW

> Status: reserve draft. Open only after engagement in the umbrella issue.

**Title: Proposal: an archive fallback that works where web.archive.org is blocked**

OnionPress's Wayback integration gives every published site a preservation copy and a fallback URL. But in the places that need censorship-resilient publishing most — behind the GFW in particular — web.archive.org itself is blocked, so the fallback fails exactly when and where the primary does. I publish from behind the GFW: the *onion* path now works there (bridges/PT/proxy, in the tor PR), but the archive fallback link we hand out is unreachable for a reader on the same network unless they are already on Tor.

What exists to build on: the sweep now archives reliably on publish, and bridge/PT support means a reader who runs Tor already has a route — including to the Internet Archive's own onion mirror of web.archive.org. The gap is readers who cannot or will not run Tor.

Questions:

1. Is the Archive's onion mirror something OnionPress could bless as a canonical alternate fallback URL (archive links that resolve over Tor), and is it operationally supported enough to depend on?
2. Would the Archive consider regional/mirror read endpoints (or IPFS-style distribution of WACZ/WARC exports) that OnionPress could rotate through when archive.org is unreachable?
3. Should OnionPress's archive links be self-describing about alternatives — one fallback page listing the wayback URL, the onion-mirror URL, and a downloadable archive — so a blocked reader has every path in one place?

This one is as much an Internet Archive infrastructure conversation as an OnionPress one — which is exactly why I'd like to have it with you.
