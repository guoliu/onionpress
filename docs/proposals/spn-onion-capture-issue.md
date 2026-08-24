# Issue for the Internet Archive: SPN captures .onion pages without their embeds

> Filed as [internetarchive/wayback#303](https://github.com/internetarchive/wayback/issues/303) on 2026-08-24. Measurements from a live OnionPress site I operate. This file is the source of truth for the issue body — edit here, then `gh issue edit 303 --repo internetarchive/wayback --body-file <this file>` minus the title line and this note.

**Title: Save Page Now captures .onion pages without their embeds — regression dated to end of June 2026**

I publish a static site through [OnionPress](https://onionpress.org), a Tor publishing stack that serves a site from the author's own machine over a `.onion`. When that machine goes offline, readers are redirected to the newest Save Page Now capture, so that capture is what they actually see. Right now they get the HTML with no CSS and no images.

Everything below goes through the SPN2 API — `POST /save` with an `Authorization: LOW key:secret` header — submitted over Tor to the archive.org onion mirror (`archivep75mb…onion`), because the publishing host has no clearnet route. The same script and the same credential produce correct captures for clearnet targets, which is the control throughout.

## What I see

Over `.onion`, SPN returns a successful capture with nothing in it: `counters` comes back `{"outlinks":0,"embeds":0}`, with `http_status: 200` and `duration_sec` of 7–10s.

I control the origin being captured, so I can also say what arrives there:

- Eight captures of eight distinct URLs produced eight `GET`s and **zero `HEAD`s** in my access log. The docs say SPN does a HEAD on the target to decide between the headless browser and a plain GET.
- Each capture is a single `GET` with a Chrome UA, answered `200`, never followed by a request for any of the page's 19 images, its stylesheet, or its 4 scripts — all same-origin.
- `capture_screenshot=1` on a **successful** onion capture returns no `screenshot` field. The same parameter on `www.debian.org` returns one.
- The origin was up throughout, and answers a HEAD over Tor in 2.5s with `200 text/html`.

## When it started

Non-HTML captures per month in 2026, from CDX:

| onion | Feb | Mar | Apr | May | Jun | Jul | Aug |
|---|---|---|---|---|---|---|---|
| DuckDuckGo | 83 | 83 | 420 | 172 | 332 | — | **0** |
| `archivep75mb…onion` (yours) | 337 | 50 | 842 | 361 | 124 | — | **0** |
| Facebook | 86 | 28 | 156 | 19 | 43 | — | **0** |

July has no captures at all on any of the three, then August resumes HTML-only. `x-archive-src` on the pre-July records reads `spn2-…`, so these were SPN's captures rather than another crawler's.

## Isolating it

One script, one credential, identical parameters, submission carried over Tor in every arm. Only the target's transport differs:

| target | embeds | outlinks |
|---|---|---|
| my site, clearnet | 24 | 29 |
| my site, `.onion` | **0** | **0** |
| `duckduckgo.com` | 158 | 63 |
| DuckDuckGo's `.onion` | **0** | **0** |

The DuckDuckGo pair is the one that matters: it rules out my site, my server, and my generator.

I also ruled out `js_behavior_timeout`, since I had been sending `0` — values `0`, omitted, and `30` give identical results on each transport.

## What I think is happening

No headless browser runs for onion targets. Embeds and outlinks are browser-derived and both zero; a screenshot needs a render and none is produced on a success; and the HEAD that selects browser-versus-GET never arrives. A plain-GET path taken unconditionally over `.onion` accounts for all four observations with one cause.

I can't see inside SPN, so that is an inference. If onion embed capture was disabled deliberately, I would rather know that than have this treated as a bug.

## Per-asset submission is refused too

Submitting assets directly works on clearnet — `https://www.debian.org/debhome.css` and a `.png` beside it both captured on request. Over `.onion` the same submissions return `error:no-captures` without ever reaching my server.

The discriminator looks like the file extension rather than the response: my `/feed/`, serving `application/rss+xml` from an extensionless URL, captures and replays with the correct content type, while `.css` and `.jpg` on that same host in the same run are refused unread.

## If you reproduce this

About a quarter of my onion submissions return `error:gateway-timeout` — ordinary circuit failure. My own first screenshot run failed that way in *both* arms and looked like browser-related evidence until a no-screenshot control run alongside it showed otherwise. A single onion result proves nothing here; retry until you get a non-timeout verdict.

Happy to run further tests from the origin side, which is the part that is normally hard to observe. And if Save Page Now isn't tracked in this repo, point me at the right place and I'll move this.
