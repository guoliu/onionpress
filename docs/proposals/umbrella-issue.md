# DRAFT — umbrella tracking issue for brewsterkahle/onionpress

> Status: draft, not posted. Posting sequence (user sign-off required):
> 1. Personal note to Brewster (not GitHub): ~3 sentences + the update
>    recording + ONE link (this issue). Offer a call/live demo.
> 2. Post this issue with recording 1 embedded at top (≤10 MB embeds
>    inline; otherwise a guoliu/onionpress release asset + GIF teaser in
>    the issue). Never a moss-repo link (private → 404 for him).
> 3. Open PR 1 the same day. Open PRs 2–4 same-day-numbered or after first
>    response — judgment call.
> 4. The three collaboration proposals stay a PARAGRAPH here; open them as
>    individual issues only when he engages with that thread. Drafts live in
>    this directory, ready.

---

**Title: Static-site publishing for OnionPress — working demo + PR series**

*(recording 1 embedded here: editing a page in moss and republishing —
the live `.onion` updates in one click)*

The video is [moss](https://mosspub.com) — a static-site publishing app —
updating a site that is served from a live `.onion` through OnionPress.
Everything it uses is offered back in the PRs below — we kept nothing
needed to reproduce it.

Two things these PRs add to OnionPress:

- **Static-site generators become publishers.** The receiver is a small
  documented HTTP protocol — status, upload, atomic commit — so any SSG
  can drive OnionPress as a publish target. moss is the first client;
  nothing in the protocol is moss-specific, and WordPress stays exactly
  what it is.
- **OnionPress works from behind the GFW.** Tor can route through the
  proxy the user is already running, and the stack ships bridge and
  pluggable-transport support (obfs4, Snowflake) for both C Tor and Arti.

*(recording 2 embedded lower down: a first publish from moss — the
onion-name step claims a memorable name, then the site comes up at
`<name>.onion`)*

## Try it yourself (~5 minutes)

1. Install OnionPress from our fork's release (or run your own build with
   the PRs applied) and start it.
2. `./test-receiver.sh` — publishes a fixture site over the loopback
   receiver and verifies it is served at the site root ahead of WordPress.
3. The wire protocol any SSG can implement: `docs/static-publish-protocol.md`.

## The PRs (suggested review order)

Each is reviewable alone; nothing later is required to accept something
earlier. PR 4's branch includes 2 and 3 underneath (its feature builds on
both); its own commits are the last nine.

| PR | What | Size |
|---|---|---|
| #__ | Bug fixes: port re-resolution after restart, a start that exited 1 on success, an auto-login open-redirect, dead clearnet onionname redirects, reachability tri-state in status.json, 2 GB default VM memory | S |
| #__ | Tor bridge/pluggable-transport/upstream-proxy support (C Tor **and** Arti), a watchdog that escalates on "serving" not "bootstrapped", Wayback sweep hardening | M |
| #__ | Static-first serving: Apache rules that serve a published static site ahead of WordPress, runtime-injected and self-repairing; pages served uncompressed so Wayback captures are whole | S |
| #__ | The static-publish receiver: loopback REST endpoints (status / upload / atomic commit), hardened tar extractor, headless `onionname` CLI, `--managed` unattended installs, Wayback coverage of the static site's real pages | M |

A design note on all of it: where OnionPress already had a mechanism —
Save Page Now archiving, the onion-service lifecycle, provisioning — we
extended that mechanism rather than building a parallel one.

## What we deliberately did NOT send

Our fork also carries a repointed self-updater, a fork-built tor image
(superseded by the one-line Dockerfile change in the bridges PR — you'd
rebuild and repin your own image), and fork CI. None of it is in these PRs.
Two pre-existing pins worth knowing: `containers.py` and `launcher_ops.py`
pin an older tor digest than docker-compose, so bridges won't reach the
takeover-worker path until all three move together (flagged in the bridges
PR).

## Where we'd like to go together

Three directions we'd love your read on — happy to open any of these as its
own issue if it interests you:

- **A DNS domain alongside the onion name and onion address**: sites
  published through OnionPress get a `.onion`; most authors also need a
  `example.com` their readers can reach — the dual life onionpress.org
  itself has.
- **WordPress as an optional component**: a publisher that brings its own
  static site needs only tor + receiver + Apache. Making WordPress
  optional would shrink the download and the idle footprint for that path,
  while keeping today's full stack the default.
- **Wayback behind the GFW**: making the archive fallback usable where
  web.archive.org itself is blocked.
