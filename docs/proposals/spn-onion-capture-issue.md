# DRAFT — issue for the Internet Archive: Save Page Now does not run its browser over .onion

> Status: draft, not filed. Intended as a standalone issue the umbrella issue links to, so the OnionPress PR series does not have to carry a Wayback bug report inside it. Measurements taken 2026-08-24 from a live OnionPress site we operate.

**Title: Save Page Now captures .onion pages without their embeds — the headless browser appears not to run**

## Summary

Over `.onion`, Save Page Now fetches the document and stops there. Embedded resources — stylesheets, images, scripts — are never requested, so onion snapshots replay as unstyled HTML. The same site on clearnet, built by the same generator and submitted by the same script with the same parameters in the same minute, captures 24 embeds. This is not specific to our site: DuckDuckGo's onion behaves identically, and non-HTML capture on three well-known onions stopped in the same month.

This matters to OnionPress specifically, because an OnionPress site that goes offline redirects readers to its Wayback snapshot. That fallback is the product feature; right now it delivers a page with no styling and no images.

## The designed path

On the OnionPress side, a published site is reachable only at its `.onion`, and the machine serving it is somebody's laptop, so it goes offline routinely. OnionHeaven answers for a site that is down by issuing a `302` to the site's most recent Wayback capture. For that to be worth anything, the capture has to be a whole page. OnionPress therefore sweeps the site's real URLs into Save Page Now on publish, submitting each page over Tor to the archive.org onion mirror.

On the Internet Archive side, the documented behaviour of SPN2 is that a capture is a page *and* its embeds. From the public API documentation:

> "When we capture a web page, we also try to capture its embeds. We return them with the capture result."

and, on how the fetch is performed:

> "By default SPN2 does a HTTP HEAD on the target URL to decide whether to use a headless browser or a simple HTTP GET request. `force_get` overrides this behavior."

So the intended flow is: HEAD the target, choose the headless browser for an HTML document, let the browser load the page, and record the subresources it fetches as embeds. Nothing in either system asks for anything unusual — OnionPress submits ordinary page URLs and expects the documented default.

## Where it breaks

The browser does not appear to run for `.onion` targets. Four independent observations, all either from SPN's own responses or from the access log of the server being captured:

**1. A successful capture with `capture_screenshot=1` produces no screenshot.** A screenshot can only exist if something rendered the page. Requesting one on `www.debian.org` returns a `screenshot` field alongside 24 embeds and 29 outlinks; requesting one on our onion, on a capture that returned `status: success`, returns no `screenshot` field at all.

**2. `embeds` and `outlinks` are both zero.** Both are browser-derived. A page carrying 19 `<img>` elements, a stylesheet, 4 scripts and many internal links yields `counters: {"outlinks":0,"embeds":0}` with `http_status: 200` and `duration_sec` of 7–10s. It is not timing out; it completes quickly having found nothing.

**3. The documented HEAD never arrives.** Across eight captures of eight distinct URLs, our access log holds eight `GET`s from SPN and zero `HEAD`s. If no HEAD is issued, the browser-versus-GET decision is not being made.

**4. One request per capture, with nothing behind it.** The capture arrives as a single `GET` with a Chrome user-agent, receives a normal `200`, and is never followed by a request for any subresource on the same origin.

A plain-GET path taken unconditionally over `.onion` would account for all four at once.

## Isolation

Same script, same credentials, same parameters, same container, submission itself carried over Tor in every arm — only the target's transport differs:

| target | embeds | outlinks |
|---|---|---|
| our site, clearnet | 24 | 29 |
| our site, `.onion` | **0** | **0** |
| `duckduckgo.com` | 158 | 63 |
| DuckDuckGo's `.onion` | **0** | **0** |

The DuckDuckGo pair is the important one: it is not our site, our server, or our generator.

We also checked whether `js_behavior_timeout` was responsible, since we had been sending `0`. It is not — `0`, omitted (the documented 5s default) and `30` give identical results on each transport.

## It looks like a regression

Non-HTML captures per month in 2026, from CDX, on three onions that are not ours:

| onion | Feb | Mar | Apr | May | Jun | Jul | Aug |
|---|---|---|---|---|---|---|---|
| DuckDuckGo | 83 | 83 | 420 | 172 | 332 | — | **0** |
| archive.org's own | 337 | 50 | 842 | 361 | 124 | — | **0** |
| Facebook | 86 | 28 | 156 | 19 | 43 | — | **0** |

All three carried assets at volume through June, have **no captures of any kind** in July, and capture HTML only from August onward. The `x-archive-src` header on the pre-July records reads `spn2-…`, so those were Save Page Now's own captures rather than another crawler's. Something appears to have changed in the onion path at the end of June.

## A secondary restriction, which blocks the obvious workaround

Submitting an asset directly, one URL at a time, is a supported mode and works on clearnet — `https://www.debian.org/debhome.css` and a `.png` beside it both captured on request. Over `.onion` the same submissions return `error:no-captures` without SPN ever connecting to our server.

The discriminator appears to be the file extension rather than the response. Our `/feed/`, which serves `application/rss+xml` from an extensionless URL, captures and replays with its correct content type. On the same host and in the same run, `.css` and `.jpg` URLs are refused unread. So a site could in principle make its assets archivable by serving them from extensionless URLs, which is not a reasonable thing to ask of a site.

## Reproducing

Submit any `.onion` page URL to `/save` with `capture_screenshot=1` and read `counters` and `screenshot` in the status response. Compare against any clearnet page.

One caveat for anyone reproducing this: roughly a quarter of our onion submissions return `error:gateway-timeout`, which is ordinary Tor circuit failure. Our own first run of the screenshot test failed that way in *both* arms and briefly looked like a browser-related error; only a no-screenshot control run alongside it showed otherwise. A single onion result is not evidence — retry until you get a non-timeout verdict.

## What we are not claiming

We cannot see inside SPN, so "the browser does not run" is an inference from four external observations, not a diagnosis. We do not know whether this was a deliberate change, and if capturing onion embeds is intentionally disabled we would rather know that than have it treated as a bug. We are also not reporting a Tor reachability problem: the document itself captures fine, our onion answers a HEAD in 2.5s over Tor with `200 text/html`, and the site was up throughout.
