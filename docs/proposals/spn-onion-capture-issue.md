# DRAFT — issue for the Internet Archive: SPN stopped capturing embeds over .onion at the end of June

> Status: draft, not filed. Standalone so the OnionPress PR series does not have to carry it. Written for readers who built Save Page Now — it states only what I measured, not how SPN is supposed to work. Measurements 2026-08-24, from a live OnionPress site I operate.

**Title: Save Page Now captures .onion pages without their embeds — regression dated to end of June 2026**

## What I think you do not know

Onion captures have been document-only since roughly 2026-06-25. Before that they carried embeds at volume. I can date it on three onions that are not mine, one of which is yours, and I can add server-side observations from the origin being captured — which is the part nobody outside can normally supply.

## The dating

Non-HTML captures per month in 2026, from CDX:

| onion | Feb | Mar | Apr | May | Jun | Jul | Aug |
|---|---|---|---|---|---|---|---|
| DuckDuckGo | 83 | 83 | 420 | 172 | 332 | — | **0** |
| `archivep75mb…onion` (yours) | 337 | 50 | 842 | 361 | 124 | — | **0** |
| Facebook | 86 | 28 | 156 | 19 | 43 | — | **0** |

July has **no captures at all** on any of the three, then August resumes HTML-only. `x-archive-src` on the pre-July records reads `spn2-…`, so these were SPN's captures rather than another crawler's.

## What I see from the origin

I control the server being captured, so these are direct observations rather than inferences from the API:

- **The HEAD never arrives.** Eight captures of eight distinct URLs produced eight `GET`s and zero `HEAD`s in my access log. The only HEADs present are my own `curl` probes.
- **One request, nothing behind it.** Each capture is a single `GET` with a Chrome UA, answered `200`, never followed by a request for any of the page's 19 images, its stylesheet, or its 4 scripts — all same-origin.
- **The origin is healthy.** It answers a HEAD over Tor in 2.5s with `200 text/html`, and was up throughout.

## What I see from the API

- `counters` comes back `{"outlinks":0,"embeds":0}` with `http_status: 200` and `duration_sec` of 7–10s. Fast, successful, empty.
- `capture_screenshot=1` on a **successful** onion capture returns no `screenshot` field. The same parameter on `www.debian.org` returns one.

## The isolation

One script, one credential, identical parameters, submission carried over Tor in every arm. Only the target's transport differs:

| target | embeds | outlinks |
|---|---|---|
| my site, clearnet | 24 | 29 |
| my site, `.onion` | **0** | **0** |
| `duckduckgo.com` | 158 | 63 |
| DuckDuckGo's `.onion` | **0** | **0** |

The DuckDuckGo pair is the one that matters — not my site, my server, or my generator.

I also ruled out `js_behavior_timeout`, since I had been sending `0`: values `0`, omitted, and `30` give identical results on each transport.

## My reading, which you are better placed to confirm or reject

No headless browser runs for onion targets. Embeds and outlinks are browser-derived and both zero; a screenshot needs a render and none is produced on a success; and the HEAD that selects browser-versus-GET is absent because there is no selection happening. A plain-GET path taken unconditionally over `.onion` accounts for all four observations with one cause.

I cannot see inside SPN, so that is an inference. If onion embed capture was disabled deliberately, I would much rather know that than have this treated as a bug.

## A second restriction, which blocks the obvious workaround

Direct per-asset submission works on clearnet — `https://www.debian.org/debhome.css` and a `.png` beside it both captured on request. Over `.onion` the same submissions return `error:no-captures` without ever reaching my server.

The discriminator looks like the file extension, not the response: my `/feed/`, serving `application/rss+xml` from an extensionless URL, captures and replays with its correct content type, while `.css` and `.jpg` on that same host in the same run are refused unread.

## If you try to reproduce this

Roughly a quarter of my onion submissions return `error:gateway-timeout` — ordinary circuit failure. My own first screenshot run failed that way in *both* arms and looked like a browser-related error until a no-screenshot control run alongside it showed otherwise. A single onion result proves nothing here; retry until you get a non-timeout verdict.

## Why I care

An OnionPress site is served from someone's laptop and goes offline routinely; OnionHeaven answers by redirecting readers to the newest Wayback capture. That fallback is the feature. Today it delivers an unstyled page with no images. I have a way around it — publish to a clearnet domain and archive that instead — so this is not urgent for me. But onion capture visibly worked two months ago, and losing it quietly seemed worth telling you about.
