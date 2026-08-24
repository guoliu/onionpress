# DRAFT — PR bodies for the four upstream PRs

> Status: drafts, not posted. Posting order (revised): open the PRs first, then the umbrella issue, then edit each PR body to fill `#UMBRELLA` with the real issue number. Titles are the PR titles.

---

## PR 1 — branch `upstream/fixes`

**Title: Bug fixes: launcher lifecycle, auto-login open redirect, onionname redirects, truthful status**

Part 1 of the series in #UMBRELLA. Independent bug fixes, no feature changes; each is a small commit with its own rationale and test.

- **`start` re-probed for a free port instead of resolving the running stack's** — after a restart it could point WordPress at a port nothing was listening on.
- **A successful `start` could exit 1** — the EXIT trap's cleanup result became the script's exit code, so callers that check exit status saw failure on success.
- **Auto-login `redirect_to` was an open redirect** — the value is now run through `wp_sanitize_redirect()` *before* validation (matching what `wp_redirect()` does to it anyway) and validated as a single-slash-anchored relative path. Includes bypass tests for the sanitize-then-collapse trick (`/\evil.com` → `//evil.com`).
- **The clearnet onionname page was dead code** — every clearnet visitor to a claimed name got a 302 to the `.onion` they usually cannot open; the page now resolves to the site root, and deep paths carry through.
- **Reachability was collapsed to a boolean** — it is tri-state at its source (reachable / unreachable / unknown) and status.json now says which, so "we don't know yet" no longer reads as "down".
- **A stalled plugin download hung installs for 300 s** — it now gives up and reports instead of waiting out the timeout.
- **Default VM memory raised to 2 GB** — 1 GB OOMs the stack under an ordinary WordPress + MariaDB load.
- **`pyproject.toml` puts `src/` on pytest's path** so the suite runs from a clean checkout.

Tested: the full Python suite (586 passed) plus per-fix regression tests added in the same commits.

---

## PR 2 — branch `upstream/tor-bridges`

**Title: Tor bridges, pluggable transports, and upstream-proxy support (C Tor and Arti)**

Part 2 of the series in #UMBRELLA. This is what makes OnionPress usable from behind the GFW and similar national firewalls: without a bridge or the proxy the user is already running, Tor cannot bootstrap there at all. I publish from behind the GFW through exactly this path.

- The tor image gains `obfs4proxy` and `snowflake-client` (a Dockerfile change — you would rebuild and repin your own image; our fork-built image is deliberately not part of this PR).
- The entrypoint accepts bridge lines, pluggable-transport choice, and an upstream proxy, and renders them into the config on every startup path — for both C Tor and Arti.
- docker-compose passes the settings through; `config-template.txt` documents them.
- The watchdog escalates on **"serving"**, not "bootstrapped": bootstrap 100% with a wedged transport previously counted as healthy while readers got nothing.
- A reusable Tor-bootstrap diagnostic harness (`tools/diagnose-tor-bootstrap.sh`) for debugging exactly these setups.
- Wayback sweep hardening: the sweep can no longer report a healthy run while archiving nothing (budget/lock handling, forgotten-job clearing, CDX rescue verification).

Nothing here is publisher-specific; it is the onion-service lifecycle serving any site the stack hosts.

Tested: watchdog and sweep unit suites, plus live use from a mainland-China network via user proxy + Snowflake (the field failure that motivated the serving-ladder is written up in the watchdog's comments).

One pre-existing thing worth knowing: `containers.py` and `launcher_ops.py` pin an older tor digest than docker-compose, so bridges won't reach the takeover-worker path until all three move together.

---

## PR 3 — branch `upstream/static-first-serving`

**Title: Serve a published static site ahead of WordPress**

Part 3 of the series in #UMBRELLA. Small and self-contained: Apache serves a published static site from the current-generation symlink when one exists, and falls through to WordPress for everything else — so WordPress keeps answering for admin, auto-login, and any path the static site doesn't cover.

- The conf is **runtime-injected** (`docker cp` + `a2enconf`) and repaired after a container recreate, so your WordPress image is reused unchanged — no image rebuild, no registry.
- `--apache-conf-dir` is honored on all provision paths.
- Static pages are served **uncompressed**: Wayback's replay of a gzip-encoded capture truncates the page, so compression cost every archive capture its tail. The two CDX traps that can mask this are named in the code.

Any static content placed at the current-generation path is served this way; the receiver that populates it arrives in part 4, but this serving rule stands alone (with no generation present, behavior is exactly today's).

Tested: full suite (591 passed), including conf-injection and repair invariant tests; capture wholeness confirmed against the live archive.

---

## PR 4 — branch `upstream/static-publish`

**Title: The static-publish receiver: publish any static site to OnionPress**

Part 4 of the series in #UMBRELLA. This branch includes parts 2 and 3 underneath (the feature builds on both); its own commits are the last nine.

The receiver is a WordPress mu-plugin exposing three loopback-only REST endpoints — status, upload, atomic commit — documented in `docs/static-publish-protocol.md`. A publisher uploads a site generation as a tar, then commits it; the commit is one atomic symlink flip, so readers never see a half-published site. moss is the first client, but the protocol is deliberately SSG-agnostic; `./test-receiver.sh` exercises it with plain curl.

- **Hardened by design**: loopback-allowlist trust (positive check, not a denylist), a tar extractor with tested reject paths (path traversal, symlink escape, size caps), multipart upload only.
- **Headless `onionname` CLI**: suggest / check / register, JSON output, so a publisher can claim a memorable onion name without the GUI.
- **`--managed` provisioning** for unattended installs, and an idempotent `start` (already-running is a no-op, checked before the PID lock).
- **The Wayback sweep sees the static site's real pages**: discovery via the generation's sitemap.xml (directory walk as fallback), capture state keyed by generation id so a publish retires the old generation's rows atomically; WP boilerplate posts are excluded from counts while a generation serves. SPN's per-capture `resources[]` is recorded per-URL and surfaced honestly — a run whose captures all came back bare no longer reads as healthy — and SPN's measured `.onion` limits are documented in the plugin.
- **A scripted restart revives the MenubarApp instead of killing it**: `quit`+`start` (any supervisor's recovery pair) previously left the app dead — and it is the sole writer of status.json and the sole sender of the OnionHeaven heartbeat, so the published reachability verdict froze.

Why a mu-plugin: WordPress already provides the REST server and the subsite-collision guard; the receiver adds no new daemon and no new port.

Tested: full suite (666 passed), including extractor reject-path fixtures, receiver upload tests, sweep coverage tests against a real 32-page generation, and `./test-receiver.sh` end to end.
