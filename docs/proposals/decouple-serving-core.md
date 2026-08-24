# DRAFT — proposal issue: WordPress as an optional component

> Status: reserve draft. Open on brewsterkahle/onionpress only after Brewster engages with this thread in the umbrella issue. Frame as questions, not a design — invite his direction.

**Title: RFC: making WordPress optional for publishers that bring their own static site**

Today, publishing a static site through OnionPress means running the full stack — WordPress + MariaDB included — even when the publisher never uses WordPress. The irreducible core for that path is small: the tor container, a static file server, and the receiver's three endpoints (status / upload / atomic commit). Apache and the receiver mu-plugin are today's implementations because WordPress provided a web server and a REST framework for free — a serving-only profile could collapse them into one small process in the tor container's pod.

What that would buy, measured on the current images: WordPress is 257 MB and MariaDB 100 MB compressed, against 86 MB for the tor image — so a serving-only profile cuts most of the container download, and most of the 2 GB VM those services are sized for. For a publisher on a slow or filtered connection (exactly the users the bridges work serves), that is the difference between a ten-minute install and an afternoon.

Questions I'd like your read on before proposing anything concrete:

1. Would you want a "serving-only" profile in this repo (a compose profile / flag that skips WordPress+DB), or is WordPress-always part of the product identity, with lighter integrations belonging elsewhere?
2. If a profile: what should own the admin surface WordPress provides today (settings, onion naming, the auto-login door)? The menubar app already does most lifecycle work.
3. The receiver currently lives as a WordPress mu-plugin because that gave it a REST server and the subsite-collision guard for free. A standalone receiver (tiny PHP, or a static binary in the tor pod) drops the WordPress dependency — worth it, or complexity in the wrong place?

Context: the static receiver + static-first serving PRs work today with WordPress present; this issue is about the next step for publishers that don't need it. I'm happy to prototype whichever direction you prefer.
