# OnionPress Project Memory

## Meta
- **This file (`CLAUDE.md`) is the project memory.** Store all new memories and notes here so they travel with the repo.

## Upstreaming is in progress (2026-08-16 →)
- **Read the upstreaming plan before** touching receiver naming, tor image pins, the
  self-updater URLs, or opening any PR to `brewsterkahle/onionpress`. It records the
  locked decisions (4 stacked PRs, rename-in-fork-first, hardening scope) and holds the
  live status table — keep that table current. It is **not in this repo**: this fork is
  public and read by upstream, so coordination notes live privately (see `.gitignore`).
- `upstream/*` branches are frozen cherry-pick branches based on upstream main
  (`94ce1a363`). **Never merge fork main into them**; later fork work is the next wave.

## Open: bootstrap-percentage progress commit needs a joint cherry-pick, not a solo one
- `2c166899` ("report Tor's bootstrap percentage while it bootstraps," on a peer's `feat/wayback-moss-coverage` branch, not yet on `main`) does not cherry-pick cleanly alone — investigated 2026-08-14/15. It conflicts in `app/MacOS/onionpress` around `retag_pinned_images()`, a function that itself doesn't exist yet on `main`: it's introduced by an earlier, also-unmerged commit on the same peer branch. Landing the progress-bar fix requires both commits together (or peer coordination on sequencing), not a standalone pick of `2c166899`. Until this lands, the wait-phase-freeze bug this commit was meant to fix is only partially addressed by tonight's `fix/onionpress-install-progress` work on the moss side.

## Wayback/SPN over .onion: the headless browser never runs (measured 2026-08-24)

**Root cause, established 2026-08-24.** Over `.onion`, SPN does not run its headless browser — it takes the plain-GET path. Four independent legs, all on our own server or in SPN's own responses: (1) `capture_screenshot=1` on a **successful** onion capture returns no `screenshot` field, while the same param on `www.debian.org` returns one — a screenshot requires a render, so nothing rendered; (2) `embeds` and `outlinks` are both browser-derived and both `0`; (3) SPN2 documents a HEAD to choose browser-vs-GET, and across eight captures of eight distinct URLs our access log holds eight `GET`s and **zero `HEAD`s** — no choice is being made; (4) exactly one request arrives per capture, with a Chrome UA, and no subresource request ever follows it. Read what follows as consequences of that one cause, not as separate bugs.

Our onion site archives as bare unstyled HTML. Neither cause is our site, our params, or moss — a controlled paired test settled that: the SAME moss build, byte-identical stylesheet (md5 `2d5eab97…`) and favicon at identical paths, served on clearnet and on .onion, gave **clearnet 22 and 15 `resources` on two pages against onion 0 and 0**, and standalone leaf assets **clearnet 3/3 success, onion 0/5**. Transport is the only variable.

**Defect A — SPN does NO resource discovery at all over .onion.** State it that way, not as "embeds aren't captured": a clean 2x2 (2026-08-24, same script, same auth, same params, same container, same Tor proxy carrying the submission — only the target's transport differs) gave clearnet `embeds=24 outlinks=29` on both param arms against onion `embeds=0 outlinks=0` on both. **`outlinks` is zero too**, on a page carrying 19 `<img>`, a stylesheet and 4 scripts plus plenty of internal links, while `http_status` is 200 and `duration_sec` is 7–10s on both transports. SPN retrieves the document and then parses nothing. That is why per-asset submission was never going to be the workaround, and it is the sharpest form of the bug to send IA. Read `counters.{embeds,outlinks}` in the status JSON, not just `count(resources)` — the counters say whether discovery ran at all.

**Every onion result needs a retry before it means anything.** ~25% of onion submissions return `error:gateway-timeout`. The first run of the screenshot test failed that way in **both** arms, which read as a browser-related error and was not — only the no-screenshot control revealed it. n=1 on this transport is worthless; loop until you get a non-timeout verdict.

**Already ruled out — do not re-chase these.** `js_behavior_timeout` is NOT the cause: 0, omitted (SPN's 5s default) and 30 all give the identical result on each transport (onion 0/0/0, clearnet 24/24). Our server is not tripping the browser-vs-GET decision either — a HEAD on our onion homepage over Tor returns `200` with `Content-Type: text/html` in 2.5s, which is everything SPN needs to choose the browser (it just never asks). The markup is unremarkable root-relative `/assets/…` and `/_moss/style.<hash>.css`. And the page genuinely has embeds to find (43,658 bytes, 19 `<img>`, 1 stylesheet, 4 scripts). Separately, **never probe this with `force_get=1`** — SPN2 documents it as bypassing the headless browser, so it captures zero embeds by design and will fake this bug for you.

The dated-regression evidence still stands and is worth citing alongside the 2x2: SPN2 captured onion CSS/JS/fonts/images at full fidelity until ~2026-06-24, went completely silent through July, and returned in August capturing HTML only. Verified on two independent onions, non-HTML captures by month — DuckDuckGo: `202602`–`202606` = 83, 83, 420, 172, 332, then **nothing in 202607**, then `202608` html=4 non-html=**0**. IA's own onion: 270, 43, 678, 179, 117, then **nothing in 202607**, then `202608` html=1 non-html=**0**. Our site was first archived 2026-08-13 — entirely inside the broken window, so we never had a working period to observe. **Provenance IS readable**, contrary to what this file said before: CDX redacts `filename`, but the `x-archive-src` header on the `id_` replay endpoint does not (ours reads `spn2-20260824074929-wwwb-spn22.us.archive.org-8002.warc.gz`). Those other onions' assets came from **SPN2 itself**, not from a separate crawler. The capability exists and broke.

**The extension gate — SPN refuses non-document URLs over .onion by EXTENSION, before connecting.** It is ONION-SPECIFIC, verified 2026-08-24: `https://www.debian.org/debhome.css` and a `.png` beside it both captured **successfully** as direct submissions, while our `.css` and `.jpg` over onion returned `error:no-captures` in the same run. So per-asset submission is a supported SPN mode generally — it is restricted only on the onion path, consistent with that path being a reduced plain-GET pipeline. The gate reads the URL string alone: `.html`, `.xhtml`, `.rss`, `.atom`, and extensionless paths capture; `.css`, `.xml`, `.png`, `.js`, `.json`, `.txt` return `error:no-captures` "unreachable". **It is the extension, not the Content-Type** — a `.html` URL served as `text/css` captured and replays byte-exact as `text/css`, while `.css` and `.png` URLs served as `text/html` were refused. Proof it never connects: 27 submissions produced exactly 8 HTTP requests in our access log, mapping 1:1 onto the 8 successes; all 19 failures hit the server zero times. `force_get=1` — the documented flag for non-HTML targets — does **not** bypass it (CSS and JPEG both still `error:no-captures`, with a no-flag control failing identically). This exactly explains `/feed/` vs `/rss.xml`: same RSS bytes, but the extensionless directory URL captured 17 times and the `.xml` one never, unread.

**Read the two error strings differently.** `error:no-captures` is a permanent URL-based refusal — SPN never tried; retrying is pointless. `error:gateway-timeout` is transient Tor circuit failure, hit ~27% of the time even on document extensions; retry with a fresh URL. Note SPN's dedup returns the CACHED outcome including cached errors for >=75 min, so a retry needs a distinct URL. Also beware small samples: that 27% transient rate makes n=1 results treacherous, and it is what made an earlier css-vs-html test look like a Content-Type whitelist.

**Two consequences worth acting on.** (1) The production sweep's submit list (`op_wayback_static_state`, 31 URLs) contains zero non-document-extension URLs, so "our assets never captured" partly reflects almost no attempts, not only refusals. (2) `.rss`/`.atom` are absent from the `SetEnvIf` no-gzip list in `multisite.py`, so those feeds go out gzipped and will hit the replay-truncation bug documented there.

Only IA can fix either defect; escalate upstream, since OnionPress is IA's own project.

## "Wayback is blocked behind the GFW" is UNMEASURED — do not put it in a proposal

Measured from the Mac 2026-08-24. Without a proxy, `web.archive.org` is unreachable — root, CDX and replay all fail. **Through an ordinary proxy it is fine**: root 200 in 2.2s, replay 200 once you follow the redirect, SPN's status endpoint reachable (401 = correct unauthenticated answer, not a block). No truncation, redirect-loop, or TLS/SNI problem distinct from plain blocking was found. So a reader who already runs a proxy has a working Wayback, and the draft's headline claim has never been tested by anyone — a repo sweep found `web.archive.org` mentioned nowhere in moss at all, and every GFW measurement on record targets Tor, GitHub, mosspub, matters.town or Cloudflare. Drop the claim or measure it first.

The GFW is also structurally irrelevant to the publisher path: all five SPN/CDX endpoints go through `socks5h://onionheaven:9050` to IA's onion mirror, and there is no clearnet fallback in the wayback plugin (deliberately — it is a privacy call, see above). Every URL we ever submit for capture is `http://<onion>/…`, so **no clearnet snapshot of any published site exists**.

The real reader-side gap is ours, not censorship, and we can fix it without IA: the only archive URL a reader is ever handed is the takeover 302 in `onionheaven-redirect.sh`, and it points at **another .onion**. No proxy or VPN can open that. "A reader with a proxy is fine" is therefore false — but because of our own construction, not the GFW, and the honest version of that sentence is the one to send upstream.

**2. Every SPN call goes to IA's onion mirror, and it fails independently of clearnet.** `onionpress_wayback_user_status()`, `_submit_parallel()` and `_poll_parallel()` all target `web.archivep75mbjunhxc6x4j5mwjmomyxb573v42baldlqu56ruil2oiad.onion`. On 2026-08-24 that host was unreachable (HTTP 000 after 60s) while clearnet `web.archive.org` answered in 5.7s over the *same* SOCKS proxy — so submissions silently returned no job id and `user_status()` returned null, which reads exactly like "SPN is degraded" and is not. **Diagnose the mirror before concluding anything about SPN.** A clearnet fallback would be trivially easy and is deliberately NOT implemented here: both routes go through Tor, but the onion mirror stays inside the network while clearnet exits through an exit node, and that is a privacy call for the project owner, not a bug to patch. Raise it upstream rather than switching it.

## Naming Rules (IMPORTANT)
- The project is called **OnionPress** (one word, capital O and P). Never "Onion.Press", "onion.press", or "onion-press".
- Data directory: `~/.onionpress/` (not `~/.onion.press/`)
- GitHub repo: `brewsterkahle/onionpress`
- Use **"onion service"** (not "hidden service") in all user-facing text. Tor Project deprecated "hidden service" terminology.
  - Exception: file paths like `/var/lib/tor/hidden_service/` and Docker image names like `goldy/tor-hidden-service` cannot change — those are external identifiers.
- When writing new code, docs, issues, or UI text, always use "OnionPress" and "onion service".

## Design Principle: Follow OnionPress's Own Mechanisms
- This is Brewster Kahle's project (Internet Archive). Where it already has a designed mechanism for something — Save Page Now archiving, onion service lifecycle, the moss integration boundary — **extend that mechanism** rather than inventing a parallel one, even when a bespoke fix looks faster.
- When starting new work here, check upstream (`brewsterkahle/onionpress` issues and design notes) for how the original project intended a feature to work before designing from scratch. Reading the intent first is cheaper than discovering it from the code afterwards, and it is what keeps a fork's changes mergeable upstream.
- Examples already in the codebase: the Arti bridge fix (2026-08-09) extended `apply_bridge_config()`'s pattern into a new `apply_arti_bridge_config()` rather than adding a separate bridge-config path for Arti; the watchdog-escalation work extended `tor-watchdog.py`'s existing check/recover loop with more rungs rather than adding a second supervisor process.
- Applies equally to the `moss/plugins/onionpress` integration layer — see ADR-050 there ("moss owns the serving outcome; OnionPress keeps the work") for the cross-repo version of this same rule: moss delegates recovery work to OnionPress's own mechanisms and only acts once those are exhausted, rather than reimplementing them.
- Before finishing any change here, check whether it reached for a new mechanism where one already existed.

## Key Architecture
- macOS menubar app (py2app built from `src/menubar.py`)
- **Quiet launch on the moss-staged copy**: when the bundle lives under `~/.moss/stacks/` (`onionpress.platform.is_quiet_launch`, override `ONIONPRESS_QUIET=1/0`), the MenubarApp shows no launch splash, never auto-opens a browser, ignores the browser-open half of reopen events, and never runs its own setup wizard — moss owns install/start there and the menu bar icon is the only affordance. A standalone `/Applications` install keeps the full startup UX. Wiring is pinned by `tests/test_install_invariants.py::TestMossManagedQuietLaunch`; don't add a new startup window or auto-open path without routing it through `self.quiet_launch`.
- **The launcher has its own half of that gate, and it is the one moss actually calls.** Quiet launch shipped in Python only, so `app/MacOS/onionpress` went on raising an osascript modal ("OnionPress is already starting up") in the middle of a moss install on 2026-08-18 — moss had run that `start` itself, and there was nobody at the keyboard to dismiss it. `is_quiet_launch()` now exists in the shell too, deriving `APP_BUNDLE` from the script's own path and mirroring the Python rule exactly; `tests/test_launcher_quiet_launch.py` asserts every case against *both* implementations so they cannot drift, and the invariant test above fails on any ungated `osascript` in the launcher. **Anything the managed copy needs to say goes to stderr and the log** — moss drains both and narrates them in its advisory UI. Do not add a dialog here on the assumption a human is watching.
- Launcher shell script at `app/MacOS/onionpress` (assembled into `OnionPress.app/Contents/MacOS/` at build time)
- **Anything the launcher backgrounds that is meant to outlive it must be `disown`ed, not just `nohup`ed.** `nohup` makes a child ignore SIGHUP; it does **not** remove the job from the shell's job table, and `start` installs `trap 'rm -f "$PIDFILE"; kill $(jobs -p) …' EXIT INT TERM HUP` after `ensure_menubar_running()` has already backgrounded the MenubarApp. So for two days the `start` that revived the app SIGTERMed it on its way out: a moss install would finish green and ~35s later OnionPress tore the whole stack down, with nothing in any log (the trap is silent) and moss wrongly suspected. It looked intermittent only because a `start` that finds the app alive backgrounds nothing, and a `start` that finds the receiver answering exits *before* the trap is installed — so only a cold start, i.e. the install, could do it. Fixed in `840d204d`; pinned by `tests/test_menubar_revival.py::test_the_app_survives_the_start_arm_finishing`, which extracts the trap from the launcher rather than retyping it. Related trap: `~/onionpress-backups/takeover-check.sh` blamed the OnionHeaven takeover flag for the same deaths — both land where the app's ~90s startup ends, which is also where `start` exits, so the correlation was timing.
- Docker containers (tor, wordpress, mariadb) run inside Colima VM
- Logs at `~/.onionpress/onionpress.log` and `~/.onionpress/launcher.log`
- User-visible content at `~/OnionPress/` (backups, Creations)

## Repo Layout
- `src/menubar.py` — py2app entry point (the only flat module; everything else lives in the package)
- `src/onionpress/` — shared Python package (all non-entry-point code)
- `app/` — macOS .app bundle source (assembled into `OnionPress.app/` at build time)
  - `app/Info.plist` — canonical version source #2
  - `app/MacOS/` — launcher scripts, Swift wrapper source
  - `app/Resources/` — docker configs, plugins, icons, templates
- `OnionPress.app/` — **gitignored build output**, assembled by `build/build-dmg-simple.sh`

## Why py2app
- Modern Macs do NOT ship a usable Python — `/usr/bin/python3` is just a shim that prompts to install Xcode CLI Tools
- Apple removed Python 2 in macOS 12.3 and has no commitment to shipping Python 3 long-term
- py2app bundles the Python interpreter + all dependencies into a self-contained .app so the user never needs to know Python is involved
- This is essential for a consumer app — cannot ask non-technical users to install Xcode Command Line Tools

## Build & Release Process
- MenubarApp built with py2app via `setup.py` (extracted from `build/build-dmg-simple.sh` lines 228-276)
- **After editing ANY `src/` file that the MenubarApp uses, you MUST rebuild the MenubarApp** via `build/rebuild-menubar.sh`. The py2app bundle contains compiled `.pyc` files — editing `src/` alone does NOT update the running app.
- **py2app entry point**: `MenubarApp/Contents/Resources/menubar.py` is the ONLY copy that matters at runtime — py2app `exec()`s it from `__boot__.py`. The copies in `lib/python3.14/` and `scripts/` are unused build artifacts. When hot-patching the installed app without a full rebuild, only update `Contents/Resources/menubar.py` (and its `__pycache__/` pyc).
  - `src/menubar.py` — main app (entry point)
  - `src/onionpress/` — shared package: backup, key_manager, setup_window, onion_proxy, onion_auth, onionheaven, updater, install_native_messaging, native_messaging_host, power, health, docker, tor, colima, platform, config, containers, ui_helpers, settings_ui, browser, log_rotation, analytics_sharing, onionnames_*
  - `setup.py` — py2app config. Adding a new submodule to `onionpress` just means one new line in the `includes` list — no build-script change needed. Build scripts only copy the whole `onionpress` package into site-packages.
- **Release via GitHub releases only** (`gh release create`). Do NOT upload to Internet Archive.
- **Cut releases with `build/release.sh`** — it builds BOTH artifacts and uploads both to one release, so the Linux `.deb` can't be forgotten. (It was: v2.4.101–v2.4.106 shipped only the `.dmg`, which 404'd the README's `releases/latest/download/onionpress.deb` link — that link resolves to the *Latest* release's asset of that name.) Bump + commit first, then run it. Cross-platform rule it enforces: the `.dmg` can only be built on macOS, so on **macOS** it builds `.dmg`+`.deb` and creates/updates the release with both; on **Linux** it builds only the `.deb` and *refuses to create* a release (a `.dmg`-less "Latest" would break the Mac link) — it only attaches the `.deb` to an existing release that already carries the matching `.dmg`. Always run the Mac side first.
- **Version bumping**: run `build/bump-version.sh X.Y.Z` — it updates all version locations automatically. The 2 canonical sources are `src/menubar.py` (`self.version`) and `app/Info.plist` (`CFBundleShortVersionString`). Derived locations (`src/onionpress/__init__.py`, `setup.py` which reads menubar.py dynamically, MenubarApp plist) are updated by the bump script or at build time. The quit log in menubar.py uses `self.version` dynamically. Docker containers get the version via `ONIONPRESS_VERSION` env var from the launcher script (which reads Info.plist).
- **py2app vs setuptools 81+ incompatibility** — setuptools 81 (released 2026-02-06) removed `dry_run` from `distutils.spawn()`, which py2app 0.28.9 still uses. The build script (`build/build-dmg-simple.sh`) handles this automatically: it tries the build first, and falls back to `setuptools<81` only if py2app fails. Once py2app ships a fix, the fallback stops being needed. Track upstream: https://github.com/ronaldoussoren/py2app/issues/557

## Security
- **Database passwords are randomly generated per install** — never use defaults or hardcoded passwords. The `ensure_secrets` function generates unique passwords with `openssl rand` on first run, saved to `~/.onionpress/secrets`.
- Do not commit or log database passwords.

## Colima VM Sandboxing
- The VM has two narrow mounts: `~/.onionpress/shared:w` and `~/OnionPress:w`
- This limits blast radius if a container is compromised — attacker can only see vanity keys and user's published content (which is public anyway)
- `~/.onionpress/` stays as-is for app state (no spaces in path — required by Colima/Lima/Docker socket paths)
- `~/OnionPress/` holds user-visible backups and Creations
- All other container data uses Docker named volumes (which live inside the VM)
- **Do not move app state to `~/Library/Application Support/`** — the space in the path breaks Colima/Docker socket paths (104-char Unix socket limit + space handling issues)
- **Diffdisk cap is 20 GiB on new installs** (issue #230) via `--disk` to the first `colima start` in `app/MacOS/launcher.sh` and `src/onionpress/colima.py`. The cap is set at VM creation and is immutable — existing installs keep their original 100 GiB cap. Dangling Docker images auto-prune after each successful image pull (`update_docker_images()` in `src/menubar.py` and the bash launchers), keeping real usage well under 5 GiB in practice. The cap is configurable via `VM_DISK=N` in `~/.onionpress/config` *before* first launch; when the OnionHeaven hub vanity key is present at first start, the cap auto-bumps to 100 GiB for takeover-container headroom (hub is the edge case; normal nodes never need more than 20).
- **Do not add additional `--mount` flags without considering security implications**

## Multi-User Support (v2.4.11+)
- Multiple macOS users can run OnionPress simultaneously from the same `/Applications/OnionPress.app`
- Each user gets their own `~/.onionpress/` data dir, Colima VM, and Docker containers
- **Port offsets**: second user auto-detects port 8080 is taken and offsets all ports by +10000 (18080/19050/19077), third user by +20000, etc. Max ~5 users.
- **Detection uses socket bind test**, not `lsof` — `lsof` only sees the current user's processes and cannot detect ports bound by other users
- **Port detection must happen in the MenubarApp's `__init__`** (Python socket bind), not in the shell scripts — the MenubarApp launches first and needs the correct ports before the `onionpress` script runs
- **Module-level constants in `onionpress.onion_proxy`** (`PROXY_PORT`, `PHP_PROXY_PORT`) are set at import time. The MenubarApp must update these globals after detecting the offset: `onion_proxy.PROXY_PORT = self.proxy_port`
- The `onionpress` shell script also has detection as a fallback (for standalone use), but respects pre-set `ONIONPRESS_PORT_OFFSET` env var from the MenubarApp
- **`LSMultipleInstancesProhibited` must NOT be in any Info.plist** — macOS enforces it across ALL users sharing the same app bundle, not just per-user
- **`pgrep` in the launcher must use `-u $(whoami)`** to restrict to the current user's processes
- **PID lock file** (`~/.onionpress/onionpress.pid`) prevents the same user from double-launching; cleaned up via `trap` on EXIT/INT/TERM/HUP
- **Container-internal ports are NOT offset** — Docker networking (`onionpress-tor:9050`, `wordpress:80`) is isolated per-VM. Only host-side port mappings change.
- `OnionPress.app/` is gitignored — it is assembled at build time from `app/` source. Never commit build output.

## Colima Networking Gotcha
- **SOCKS proxy (port 9050) does NOT work through Colima VM port forwarding** — connections are accepted then immediately closed
- **For ANY communication over Tor from the Mac, always use `docker exec` into the tor container** — this is reliable
  - Do NOT use `curl --socks5-hostname 127.0.0.1:9050` from the Mac host — it will fail
- This applies to future mirror system communication (health checks, challenge-response, etc.)
- **`wget` inside the tor container CANNOT fetch external .onion addresses** — it doesn't support SOCKS proxies, so it can't resolve .onion via Arti. `wget` only works for internal container-to-container requests (e.g., `wget http://wordpress:80/`).
- **To fetch external .onion addresses, use `socat` with SOCKS4A** inside the tor container:
  - `printf "GET / HTTP/1.1\r\nHost: <address>.onion\r\nConnection: close\r\n\r\n" | socat -t 10 - SOCKS4A:127.0.0.1:<address>.onion:80,socksport=9050`
  - Or install `curl` in the tor container and use `curl --socks5-hostname 127.0.0.1:9050`
- WordPress container has `curl`
- Test onion service path (internal): `docker exec onionpress-tor wget -q -O /dev/null http://wordpress:80/`
