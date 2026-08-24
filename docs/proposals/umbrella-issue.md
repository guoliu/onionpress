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

*(recording 1 embedded here: editing a page in moss and republishing — the live `.onion` updates in one click)*

OnionPress can now work with [moss](https://github.com/Symbiosis-Lab/moss-releases) via the [OnionPress plugin](https://github.com/Symbiosis-Lab/moss-registry/tree/main/plugins/onionpress). This allow an user to right click on a folder of markdown files and media assets, preview the website generated, then publish it to Tor network, with all the controls and powers that OnionPress offers.

To make it work, I added fixes and features to a OnionPress fork, which I believe are aligned with OnionPress's design. Therefore I offered these edits back in the PRs below. Other than bug fixes, these PRs add two things:

- **Static-site generators become publishers.** The receiver is a small documented HTTP protocol (status, upload, atomic commit) so any SSG can drive OnionPress as a publish target. moss is the first client; but the protocol is compatible with any SSGs instead of moss-specific. WordPress stays exactly what it is.
- **OnionPress works with proxy now, making is usable behind GFW etc.** Tor can route through the proxy the user is already running, and the stack ships bridge and pluggable-transport support (obfs4, Snowflake) for both C Tor and Arti.

*(recording 2 embedded lower down: a first publish from moss — the onion-name step claims a memorable name, then the site comes up at `<name>.onion`)*

## Try it out

1. Install [moss](https://github.com/Symbiosis-Lab/moss-releases/releases/latest/download/moss.dmg), and enable preview features in setting.
2. Right click on a folder with md files (such one made with Obsidian), or launch moss and create one.
3. In deploy setting tab, click "+", install OnionPress plugin. After it finishes installing, click publish.

## The PRs

Each is reviewable alone; nothing later is required to accept something earlier. PR 4's branch includes 2 and 3 underneath (its feature builds on both); its own commits are the last nine.

| PR | What | Size |
|---|---|---|
| #__ | Bug fixes: port re-resolution after restart, a start that exited 1 on success, an auto-login open-redirect, dead clearnet onionname redirects, reachability tri-state in status.json, 2 GB default VM memory | S |
| #__ | Tor bridge/pluggable-transport/upstream-proxy support (C Tor **and** Arti), a watchdog that escalates on "serving" not "bootstrapped", Wayback sweep hardening | M |
| #__ | Static-first serving: Apache rules that serve a published static site ahead of WordPress, runtime-injected and self-repairing; pages served uncompressed so Wayback captures are whole | S |
| #__ | The static-publish receiver: loopback REST endpoints (status / upload / atomic commit), hardened tar extractor, headless `onionname` CLI, `--managed` unattended installs, Wayback coverage of the static site's real pages | M |

## To make it more usable

These are improvements that will make it much more usable to a broader user, but too big to propose at this stage. I'd love to talk more on these if they sounds aligned.
- **A DNS domain alongside the onion name and onion address**: sites published through OnionPress get a `.onion`; most authors also need a `example.com` their readers can reach, just like the dual life onionpress.org itself has. In moss, user can purchase and setup a domain with a few clicks; if we can find a way for Internet Archive fallback to serve that custom domain (Claude: how?), it becomes much more useful.
- **WordPress as an optional component**: a publisher that brings its own static site needs only tor + receiver + Apache (Claude: not even receiver + Apache right? We can just serve static file locally). Making WordPress optional would shrink the download and the idle footprint for that path (Claude: by how much?), while keeping today's full stack the default.
- **Wayback behind the GFW**: making the archive fallback usable where web.archive.org itself is blocked (Claude: how? what issue do we have?).
