# DRAFT — umbrella tracking issue for brewsterkahle/onionpress

> Status: draft, not posted. Posting sequence (revised 2026-08-24, user sign-off required):
> 1. Open the 4 PRs first, so the issue can cite real numbers.
> 2. Post this issue with recording 1 embedded at top (≤10 MB embeds inline; otherwise a guoliu/onionpress release asset + GIF teaser in the issue), then edit each PR body to link the issue.
> 3. Guo sends the personal note to Brewster: ~3 sentences + the update recording + ONE link (this issue). Offer a call/live demo.

---

**Title: Static-site publishing for OnionPress — working demo + PR series**

*(recording 1 embedded here: editing a page in moss and republishing — the live `.onion` updates in one click)*

OnionPress can now work with [moss](https://github.com/Symbiosis-Lab/moss-releases) via the [OnionPress plugin](https://github.com/Symbiosis-Lab/moss-registry/tree/main/plugins/onionpress). This allows a user to right-click on a folder of markdown files and media assets, preview the website generated, then publish it to the Tor network, with all the controls and powers that OnionPress offers.

To make it work, I added fixes and features to an OnionPress fork, which I believe are aligned with OnionPress's design, so I'm offering them back in the PRs below. Other than bug fixes, these PRs add two things:

- **Static-site generators become publishers.** The receiver is a small documented HTTP protocol (status, upload, atomic commit) so any SSG can drive OnionPress as a publish target. moss is the first client, but nothing in the protocol is moss-specific. WordPress stays exactly what it is.
- **OnionPress works with the user's proxy now, making it usable behind the GFW and similar firewalls.** Tor can route through the proxy the user is already running, and the stack ships bridge and pluggable-transport support (obfs4, Snowflake) for both C Tor and Arti.

*(recording 2 embedded lower down: a first publish from moss — the onion-name step claims a memorable name, then the site comes up at `<name>.onion`)*

## Try it out

1. Install [moss](https://github.com/Symbiosis-Lab/moss-releases/releases/latest/download/moss.dmg), and enable preview features in settings.
2. Right-click on a folder with md files (such as one made with Obsidian), or launch moss and create one.
3. In the deploy settings tab, click "+" and install the OnionPress plugin. After it finishes installing, click publish.

## The PRs

Each is reviewable alone; nothing later is required to accept something earlier. PR 4's branch includes 2 and 3 underneath (its feature builds on both); its own commits are the last nine.

| PR | What | Size |
|---|---|---|
| #__ | Bug fixes: port re-resolution after restart, a start that exited 1 on success, an auto-login open-redirect, dead clearnet onionname redirects, reachability tri-state in status.json, 2 GB default VM memory | S |
| #__ | Tor bridge/pluggable-transport/upstream-proxy support (C Tor **and** Arti), a watchdog that escalates on "serving" not "bootstrapped", Wayback sweep hardening | M |
| #__ | Static-first serving: Apache rules that serve a published static site ahead of WordPress, runtime-injected and self-repairing; pages served uncompressed so Wayback captures are whole | S |
| #__ | The static-publish receiver: loopback REST endpoints (status / upload / atomic commit), hardened tar extractor, headless `onionname` CLI, `--managed` unattended installs, Wayback coverage of the static site's real pages | M |

## To make it more usable

These are improvements that will make it much more usable to a broader set of users, but too big to propose at this stage. I'd love to talk more on these if they sound aligned.

- **A DNS domain alongside the onion name and onion address**: sites published through OnionPress get a `.onion`; most authors also need a `example.com` their readers can reach, just like the dual life onionpress.org itself has. In moss, a user can purchase and set up a domain in a few clicks. The archive fallback could serve that domain too: archiving under the clearnet URL also sidesteps the current gaps in Save Page Now's `.onion` capture path (see the note below), and the domain's edge can redirect to the newest snapshot when the home machine is offline — the same role the takeover already plays for the onion, applied at the DNS layer.
- **WordPress as an optional component**: a publisher that already runs its own local server (moss serves the site it previews) needs only the tor container — the static server and receiver matter when the site should keep serving after the publisher app quits, or live on another machine. Making WordPress optional would cut most of the container download (WordPress 257 MB + MariaDB 100 MB compressed, vs 86 MB for the tor image) and most of the 2 GB VM it is sized for, while keeping today's full stack the default.

## One issue with Save Page Now found in the process

An OnionPress site that goes offline falls back to its Wayback snapshot, so how well Save Page Now captures a `.onion` decides what readers actually see. Right now they see the page unstyled: the HTML is archived, and none of the CSS or images are.

Over `.onion`, SPN fetches the document and discovers nothing inside it — `embeds` and `outlinks` both come back `0` where the same site on clearnet gives 24 and 29, submitted by the same script with the same parameters in the same minute. It is not our site: DuckDuckGo's onion behaves identically, and non-HTML capture on three well-known onions stopped in the same month. Our best reading is that the headless browser is not running for onion targets.

Filed separately as #__. Onion capture visibly worked two months ago, so it seemed worth reporting.
