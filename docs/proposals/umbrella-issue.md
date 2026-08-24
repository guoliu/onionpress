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

## One thing we found in Save Page Now, since the archive fallback depends on it

An OnionPress site that goes offline falls back to its Wayback snapshot, so how well Save Page Now captures a `.onion` decides what readers actually see. Right now they see the page unstyled: the HTML is archived, and none of the CSS or images are.

The cause is not that Tor is slow or unreachable. **Over `.onion`, SPN fetches the document and then discovers nothing inside it** — `counters.embeds` and `counters.outlinks` both come back `0`, while `http_status` is `200` and the capture finishes in 7–10s. Measured 2026-08-24 with one script, one set of credentials, identical parameters, varying only the target:

| target | embeds | outlinks |
|---|---|---|
| our site, clearnet | 24 | 29 |
| our site, `.onion` | **0** | **0** |
| duckduckgo.com | 158 | 63 |
| DuckDuckGo's `.onion` | **0** | **0** |

It is not specific to our site or our generator — DuckDuckGo's own onion behaves identically. Since we run the server being captured, we can add two things from our own access log that may narrow it down.

First, each capture arrives as a **single** request with a Chrome user-agent, gets a normal `200`, and is never followed by a request for any of the page's 19 images, its stylesheet, or its 4 scripts. So it looks like discovery rather than reachability — the document comes through fine.

Second, the documented behaviour is that "SPN2 does a HTTP HEAD on the target URL to decide whether to use a headless browser or a simple HTTP GET request". **We never see that HEAD.** Across eight captures of eight distinct URLs, our log holds eight `GET`s and zero `HEAD`s from SPN (the only HEADs present are our own `curl` probes). If the decision step is being skipped over `.onion` and the plain-GET path taken unconditionally, that would explain the zero embeds, the zero outlinks and the single request together. We cannot see your side, so we offer that as a hypothesis rather than a diagnosis.

It also looks like a regression rather than a limitation, and it is not just us. Non-HTML captures per month in 2026, from CDX, on three onions none of which are ours:

| onion | Feb | Mar | Apr | May | Jun | Jul | Aug |
|---|---|---|---|---|---|---|---|
| DuckDuckGo | 83 | 83 | 420 | 172 | 332 | — | **0** |
| archive.org's own | 337 | 50 | 842 | 361 | 124 | — | **0** |
| Facebook | 86 | 28 | 156 | 19 | 43 | — | **0** |

All three carried assets at volume through June, have **no captures at all** in July, and since August capture HTML only (4, 1 and 3 documents respectively, zero non-HTML). Provenance on the pre-July ones reads `spn2-…` in the `x-archive-src` header, so these were Save Page Now's own captures, not another crawler's. Something appears to have changed in the onion path at the end of June.

There is a second thing, which blocks the obvious workaround. Submitting an asset directly, one URL at a time, is a supported mode — `https://www.debian.org/debhome.css` and a `.png` beside it both captured fine when we tried them. Over `.onion` the same submissions return `error:no-captures` without SPN ever connecting to our server. The discriminator appears to be the file extension rather than the response: our `/feed/`, which serves `application/rss+xml` from an extensionless URL, captures and replays with its correct content type, while `.css` and `.jpg` on the same host are refused unread.

Neither is urgent for us — we can route the fallback through a clearnet domain, and that is a better answer anyway. But onion capture visibly worked two months ago, so we thought it was worth reporting.
