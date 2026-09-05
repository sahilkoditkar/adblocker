# Ad Blocking Across Platforms — Complete Research

**Research dates: 2026-08-28 to 2026-09-02.** Consolidated from eight research passes: the platform survey, the final recommendation, `adblock-rust` integration surfaces, the Android lite-browser plan, DNS provider comparison, the "personal SWG" concept, the AABrowser shell assessment, and the WebView-interception / Android-developer-verification verification pass.

Goal: map every practical way to block ads on PC (Windows/macOS/Linux), Android, and iOS — browser-level, system-level, and network-level — and decide what, if anything, is worth building.

Where later research corrected earlier conclusions, **only the corrected version appears below**; the corrections themselves are listed in §14.

---

## Table of contents

1. [TL;DR, final recommendation, and the recommended stack](#1-tldr-final-recommendation-and-the-recommended-stack)
2. [How ad blocking works — the three layers](#2-how-ad-blocking-works--the-three-layers)
3. [Brave and `adblock-rust` internals](#3-brave-and-adblock-rust-internals)
4. [Filter lists, and why DNS/domain blocking can't block everything](#4-filter-lists-and-why-dnsdomain-blocking-cant-block-everything)
5. [Browser extensions — MV2/MV3 state of play](#5-browser-extensions--mv2mv3-state-of-play)
6. [System-wide blocking, per OS](#6-system-wide-blocking-per-os)
7. [DNS providers compared, and AdGuard's business model](#7-dns-providers-compared-and-adguards-business-model)
8. [Where `adblock-rust` can actually be integrated](#8-where-adblock-rust-can-actually-be-integrated)
9. [The Android lite WebView browser](#9-the-android-lite-webview-browser)
10. [The "personal SWG" concept](#10-the-personal-swg-concept)
11. [Android developer verification](#11-android-developer-verification)
12. [Hard limits nobody escapes](#12-hard-limits-nobody-escapes)
13. [Build options, ranked](#13-build-options-ranked)
14. [Source notes, corrections, and uncertainty flags](#14-source-notes-corrections-and-uncertainty-flags)

---

## 1. TL;DR, final recommendation, and the recommended stack

### The answer in one line

**Router-level AdGuard DNS is the right base layer and genuinely solves most of the *web* ad problem network-wide — but that's ~80% of *requests*, not 80% of perceived annoyance (YouTube: 0%). Layer a browser-side blocker on top for the rest. No custom browser build is required on desktop; on Android, a WebView-based browser with `adblock-rust` is the build-worthy project.**

### The seven findings that matter

1. **There is no single mechanism that blocks all ads on all platforms.** Every real solution is a *stack*: DNS/domain blocking = widest coverage, lowest precision; browser-level filtering = highest precision, narrowest coverage; on-device MITM proxy = in between, defeated by cert pinning and QUIC/ECH.
2. **Domain-only blocking structurally cannot block** first-party ads and server-side ad insertion (YouTube, Twitch, Facebook), anything needing path granularity, or leftover empty boxes. Details in §4.
3. **Brave doesn't use an extension** — it compiles EasyList/uBO-syntax filters in a Rust engine (`adblock-rust`, MPL-2.0, reusable) wired into Chromium's network stack. That engine, or Ghostery's TypeScript equivalent, is the starting point for anything browser-side (§3).
4. **Extensions are no longer "the" reliable way on Chrome.** MV2 is fully removed as of mid-2026; full-power extensions now exist only on **Firefox** (desktop *and* Android). On Chrome the ceiling is uBlock Origin Lite (§5).
5. **Cross-OS "block everywhere" is possible, but only at DNS granularity** — router DHCP on the LAN, Android Private DNS (DoT) and iOS encrypted-DNS profiles off-LAN (§6, §7).
6. **The full engine cannot be bolted onto stock Chrome via an extension** — MV3 has no blocking `webRequest`. The only full-power path into unmodified Chrome is a **local MITM proxy**, and the maintained open-source one is now **Zen** (§8).
7. **Android WebView interception works** and reaches **~78–83%** of the full engine — with shipping precedent (Edsuns/AdblockAndroid runs Brave's engine through `shouldInterceptRequest`). This is the clearest product gap (§9).

### The recommended stack, assembled

1. **Now, zero build**: AdGuard DNS IPs (`94.140.14.14` / `94.140.15.15`) on the router + Android Private DNS / iOS profile pointed at `dns.adguard-dns.com` for off-network + **Brave** on desktop (or Chrome + uBO Lite in Complete mode + **Zen** if staying on Chrome) + Brave or Cromite on Android.
2. **Build option A (recommended first)**: the Android lite WebView browser with `adblock-rust` — fills a real market gap, small scope (§9).
3. **Build option B (desktop)**: a Rust local filtering proxy — a Privaxy revival on current `adblock-rust` — full-power blocking for stock Chrome and every desktop app, competing with Zen and AdGuard desktop; merges naturally with the threat-feed concept in §10.
4. **Not worth building**: our own DNS resolver product (AdGuard Home already exists and is excellent — *run* it, don't rewrite it), a Chrome MV3 extension (ceiling already occupied by uBO Lite), or a Chromium fork (a 2-week rebase treadmill).

### Layer 1 — Router DNS, the cross-platform base

**Setup**: router WAN/DHCP DNS fields take **IP addresses, not hostnames**. AdGuard's public IPs are free, no signup, no query cap. `dns.adguard-dns.com` is the DoT/DoH hostname — that goes in **Android Private DNS** and iOS/macOS profiles, not in the router. NextDNS at a router needs its linked-IP dance (dynamic-IP households need a DDNS updater); AdGuard's fixed IPs are simpler. Routers running OpenWrt/Asuswrt-Merlin can use an encrypted upstream, or run **AdGuard Home on the router itself** (GL.iNet ships it in the GUI) for custom lists and per-client rules with no caps.

**What it solves**: third-party display/programmatic ads and trackers in *every* app on *every* device — browsers, mobile apps, smart TVs. Useful surprise: Chrome's "automatic" Secure DNS **auto-upgrades AdGuard's IPs to AdGuard's own DoH** (they are in Chromium's provider list), so filtering survives Chrome's encrypted DNS by default. The bypass case is a user manually selecting Google/Cloudflare in `chrome://settings/security`.

**What it does not solve**: YouTube ads (essentially 0% — see §4a), cosmetic leftovers and "disable your ad blocker" walls, hardcoded-DNS devices (Roku ships hardcoded 8.8.8.8 — needs router NAT-redirect of port 53 plus blocking 853, which consumer firmware can't do but OpenWrt/Merlin/pfSense can), and devices off the home network.

**Honest "80%" assessment**: fair for general web browsing measured in requests/bytes (Pi-hole installs typically sinkhole 15–25% of all DNS queries; academic work confirms DNS filtering works but under-performs full extension blockers). Optimistic measured in *ads you actually see*, because the most annoying category — first-party video ads — is in the 0% bucket.

### Layer 2 — In-browser, per platform

| Where | Best option without building anything | Notes |
|---|---|---|
| Desktop, if browser choice is free | **Brave** (or Firefox + uBO) | Full engine built in; nothing to add |
| **Desktop, staying on Chrome** | **uBO Lite (Complete mode) + the DNS layer**, or a **local MITM proxy (Zen)** for full power | See §8.2 |
| Android | Brave (heavy) or **our WebView browser** (§9) | Chrome Android has no extensions |
| iOS | AdGuard/Brave content blockers + AdGuard DoT profile | Platform ceiling; nothing better is possible |

---

## 2. How ad blocking works — the three layers

Every blocker does some subset of:

| Layer | What it does | Requires | Examples |
|---|---|---|---|
| **Network filtering** | Cancel/redirect ad requests before they load | Seeing the request URL + context (initiator, resource type) | uBO, Brave Shields, AdGuard proxy |
| **DNS filtering** | Make ad domains not resolve (sinkhole → `0.0.0.0`/NXDOMAIN) | Controlling the resolver | Pi-hole, AdGuard Home, NextDNS, hosts file |
| **Cosmetic + scriptlet filtering** | Hide leftover DOM elements (`##.ad-slot`), inject JS to defuse ad SDKs and anti-adblock detectors | Running code inside the page | uBO content scripts, Brave's isolated-world injection |

DNS sees only hostnames. Network filtering sees full URLs plus request context. Cosmetic/scriptlet filtering sees the DOM. The further down this list, the cleaner the result — and the harder it is to deploy outside a browser.

---

## 3. Brave and `adblock-rust` internals

Brave Shields is **not** an extension — it's native browser code, which is exactly why it escapes all Manifest V3 restrictions.

**Engine — [`adblock-rust`](https://github.com/brave/adblock-rust)** (crate `adblock`, v0.13.3 as of Aug 2026, MPL-2.0):

- Parses ABP/uBO filter syntax (EasyList etc.) into `NetworkFilter` structs with a bitmask for options (`$script`, `$third-party`, `$important`, anchors, …); `$domain=` values stored as hashes.
- Matching uses **tokenization plus a reverse index**, not bloom filters: each filter is bucketed under its *least frequent* token (frequency histogram computed across all filters); at request time the URL is tokenized the same way, so only the few filters sharing a token get evaluated. Brave reported ~5.7 µs per request on EasyList+EasyPrivacy, ~69× faster than their old C++ bloom-filter engine.
- Filter buckets are evaluated in precedence order: `$important` → normal filters → exceptions (`@@`) → `$redirect`/`$removeparam`/`$csp`. Engines serialize to a binary DAT for fast startup.
- It natively ingests plain hosts lists (`FilterFormat::Hosts`), so one engine covers ad rules, domain lists, and URL-path threat rules. It has **no IP concept** — IP feeds need a separate CIDR/longest-prefix matcher (§10).

**Chromium integration**: the hook is `OnBeforeURLRequest_AdBlockTPPreWork` in [`brave_ad_block_tp_network_delegate_helper.cc`](https://github.com/brave/brave-core/blob/master/browser/net/brave_ad_block_tp_network_delegate_helper.cc), inside Brave's proxying URL loader. It gets the real initiator origin, frame tree, and Chromium resource type, computes third-partyness, calls `AdBlockEngine::ShouldStartRequest()` on a task runner, and on match blocks or substitutes a stub response. Being in the browser also enables CNAME uncloaking and seeing requests extensions can't.

**Cosmetic/scriptlet side**: rules are split hostname-specific vs generic; generic rules are keyed by class/id, so the renderer reports which classes/ids actually exist on the page (via Mojo `HiddenClassIdSelectors()`) and only matching selectors come back. Injection runs in an isolated world before page scripts; procedural operators (`:has-text()`, `:upward()`, …) are evaluated by a bundled in-page observer script. Scriptlets are uBO-compatible ([brave/adblock-resources](https://github.com/brave/adblock-resources)).

**Lists**: a catalog of ~91 lists in [`adblock-lists`](https://github.com/brave/adblock-lists)' `list_catalog.json` — defaults include uBO filters, EasyList, EasyPrivacy, URLhaus, cookie-notice lists, plus Brave's own. Delivered as signed CRX components via Chromium's component updater, so list updates need no store review.

**The public API surface** (this is what makes it host-agnostic — it is a pure classifier and selector provider, not a network stack):

```rust
// network decision — host must supply full request context
Request::new(url, source_url /* initiator */, request_type, method)
engine.check_network_request(&request) -> BlockerResult { should_block, redirect, rewritten_url, .. }

// cosmetics — host must inject the returned CSS/JS into pages
engine.url_cosmetic_resources(url) -> { hide_selectors, injected_script (scriptlets), .. }
engine.hidden_class_id_selectors(classes, ids, ..)  // generic rules, fed by a page-side MutationObserver

engine.serialize() / deserialize()  // precompiled binary "DAT" for fast startup
```

An embedder therefore needs, in decreasing order of importance: (1) a **per-request hook** before the request goes out, carrying URL + initiator + resource type, with power to cancel/redirect — without this, cosmetics only; (2) **CSS/JS injection at document start** — without this, network blocking only and broken layouts remain; (3) a DOM feedback channel reporting page class/ids, for cheap generic cosmetic filtering. If the host can't supply the initiator (`source_url`), third-party classification degrades (an unparseable source is treated as third-party) — this is exactly what hurts proxy and WebView hosts.

**Bindings**: Rust crate (canonical, active); npm **`adblock-rs`** (version-locked to the crate, but a **Neon native module, not WASM** — installing it compiles Rust); WASM works via `wasm-bindgen` but there is **no official published WASM package** (Brave's demo: [adblock-rust-dashboard](https://github.com/brave-experiments/adblock-rust-dashboard)); PyPI `adblock` (python-adblock) is **abandoned** (last release 2022). A `content-blocking` cargo feature converts lists to Apple Safari JSON, lossily (§8.3).

**Reported embedders**: Mozilla (experimental, disabled-by-default prototype landed ~March 2026), Ladybird, Waterfox, Perplexity Comet.

**Comparable embeddable engines**:

- **[`@ghostery/adblocker`](https://github.com/ghostery/adblocker)** (TypeScript, MPL-2.0) — same reverse-index idea, ~99% uBO filter compatibility, ready adapters for WebExtensions, **Electron, Puppeteer, Playwright**, Node. Easiest to embed in a JS app.
- **AdGuard CoreLibs** — *not* actually open (repo is README-only). **[AdGuard DnsLibs](https://github.com/AdguardTeam/DnsLibs)** *is* open (Apache-2.0, C++ with Java/C# bindings) but DNS-level only.

**Even Brave can't block everything**: server-side ad insertion (§4a), first-party ads (which Brave *by policy* doesn't target except in Aggressive mode — [blocking policy](https://github.com/brave/brave-browser/wiki/Blocking-goals-and-policy)), roughly a day of lag in the anti-adblock arms race, and a much weaker iOS version (the App Store confines it to WebKit content-blocker JSON, a converted "Slim List").

---

## 4. Filter lists, and why DNS/domain blocking can't block everything

### 4.0 The two kinds of list

**URL/DOM-aware lists (EasyList family)** — usable only by engines that see full requests and the DOM:

- EasyList, EasyPrivacy (dual GPLv3 / CC BY-SA 3.0), uBlock filters (GPLv3), AdGuard filters (GPLv3), Fanboy lists (CC BY 3.0), Peter Lowe's (non-commercial).
- Syntax power: full-URL patterns (`||cdn.example.com/assets/ads/*`), options like `$third-party`, `$script`, `$domain=`, exceptions `@@`, and DOM rules — `##selector` hiding, `#?#` procedural filters, `##+js(...)` scriptlets.

**Domain-only lists (hosts-style)** — the only kind DNS blockers can use:

- [StevenBlack/hosts](https://github.com/StevenBlack/hosts) (aggregator, ~2k–168k domains depending on variant), [OISD](https://oisd.nl/) ("zero breakage" philosophy; ABP-syntax `\|\|domain^` only since 2024), [HaGeZi](https://github.com/hagezi/dns-blocklists) (tiered Light→Ultimate, ~42k–269k entries), AdAway, 1Hosts. Mostly GPL-3.0.
- One bit of information per hostname. Notably, **HaGeZi's own README says DNS blockers "can't catch everything"** and recommends pairing with a browser content blocker.

Licensing caveat: the FSF notes it's unclear filter lists are copyrightable at all, so the licenses are partly nominal — but attribution is cheap, keep it.

### 4.a First-party ads and server-side ad insertion (SSAI) — the 0% bucket

YouTube serves ad video segments and content segments from the same `*.googlevideo.com` hosts and the same API origin — blocking the domain kills playback entirely. Twitch's SureStream (since 2016) stitches ad segments *into the same HLS manifest* as the stream: same CDN hostname, same URL shape. Facebook/Instagram serve feed ads first-party with randomized, obfuscated markup; Spotify is the same story.

Against SSAI, **no network-layer blocker of any kind helps** — not DNS, not a proxy, not declarative rules. Only client-side player patching works (scriptlets rewriting `ytInitialPlayerResponse` and similar), which is exactly the layer DNS and MV3-declarative blockers do not have. NextDNS's own documentation says DNS blocking cannot touch YouTube ads. **This single fact drives most of the architecture decisions in this document** and is referenced from §1, §5, §6, and §12 rather than repeated.

### 4.b No path granularity

DNS resolves names; the path and query never reach the resolver and are encrypted under HTTPS anyway. `cdn.site.com/ads/x.js` and `cdn.site.com/app.js` are indistinguishable at DNS — a shared CDN is all-or-nothing.

### 4.c No cosmetic filtering

A sinkholed request still leaves the `<div>`/`<iframe>` in the DOM: empty boxes, broken carousels, "content failed to load" placeholders.

### 4.d CNAME cloaking — the one place DNS is stronger

Trackers hide behind first-party aliases (`analytics.publisher.com` CNAME → `tracker.evil.net`). Naive domain lists miss it; uBO uncloaks only on Firefox (Chromium exposes no DNS API to extensions); recursive DNS blockers actually *see* the CNAME chain.

### 4.e DNS bypass

Apps and devices with hardcoded resolvers (Chromecast and Roku → 8.8.8.8) or built-in DoH clients ignore your DNS entirely. Countermeasures: NAT-redirect all port 53, block port 853 (DoT then falls back to plain DNS), blocklist known DoH endpoints — inherently incomplete, since DoH is indistinguishable HTTPS on 443.

### 4.f The arms race

Measured research (CV-Inspector, NDSS 2021; Nithyanand et al., FOCI '16) shows circumvention uses randomized high-entropy subdomains and paths — exactly the case static domain lists lose and URL/scriptlet rules counter. Hence the universal recommendation: **DNS blocking for breadth, browser blocker for depth.**

---

## 5. Browser extensions — MV2/MV3 state of play

**Manifest V2 (the uBlock Origin model)**: blocking `chrome.webRequest.onBeforeRequest` runs the extension's *code* synchronously per request → arbitrary logic, dynamic per-site filtering, `$redirect` stubs, header-based rules; content scripts at `document_start` do cosmetic and procedural filtering; **scriptlet injection** patches page globals before site code runs (the number-one anti-adblock weapon); filter lists update as data, with no store review.

**Manifest V3 (Chrome today)**: `declarativeNetRequest` — a static rule table matched by the browser; the extension never sees requests. Limits: 30k static rules guaranteed (plus a shared pool to ~330k), 30k dynamic (only ~5k "unsafe" block/redirect dynamic rules), 1k capped RE2-only regex. Lost: dynamic filtering, response-header rules, per-site switches, `$csp`, most regex, `$removeparam` combinations, the engine's exception/`$important` precedence semantics, and most scriptlet power. **MV2 is gone from Chrome**: disabling began Oct 2024, enterprise policy override ended with Chrome 138–139 (July 2025), last flag workarounds stripped in Chrome 150–151 (mid-2026), Web Store purge reported for Aug 31, 2026. uBlock Origin **Lite** is the MV3 fallback — no custom filters, no element picker, cosmetic/scriptlet filtering only with per-site opt-in.

**Firefox is the holdout**: it ships both APIs and has said MV2/`webRequest` is not deprecated → **full uBO works on Firefox desktop and Firefox for Android** (open extension ecosystem since Firefox 120, Dec 2023). This is the only mobile configuration with desktop-grade blocking.

**Safari**: declarative-only Content Blocker JSON (`WKContentRuleList`), 150k rules per blocker (was 50k); AdGuard for Safari ships **six** content-blocker extensions to reach ~900k rules. The blocker never sees traffic (privacy by design), so no request-time logic.

**Chrome on Android**: no extensions, and none planned for phones — the "Desktop Android" convergence build that loads extensions targets large-screen devices (dev channel, not phones). Kiwi Browser is dead (Jan 2025). Alternatives: Firefox Android + uBO, Brave/Vivaldi built-in, Edge Android (built-in ABP plus a small curated store including uBO Lite).

**Verdict**: extensions are still the strongest *in-browser* method — but only where full extensions still exist (Firefox). On Chrome the ceiling is uBO Lite; Brave's built-in engine now beats anything installable on Chrome. So no, extensions are not the only way, and on Chromium they are no longer even the best way.

---

## 6. System-wide blocking, per OS

### 6.1 Hosts file (Windows / macOS / Linux; Android needs root)

Map ad domains to `0.0.0.0` in `/etc/hosts` or `drivers\etc\hosts`. Free, no software. Limits: **no wildcards** (every subdomain must be listed literally → 50k–700k-line files), Windows DNS-cache performance issues at scale, domain granularity only, needs admin plus `ipconfig /flushdns`. Android requires root ([AdAway](https://github.com/AdAway/AdAway)).

### 6.2 Windows and macOS content-level filtering (the AdGuard-desktop model)

The only way to filter *inside* HTTPS system-wide is **MITM of your own traffic**: a local filtering proxy plus a locally generated root CA installed in the system trust store; TLS is terminated, filtered, re-encrypted. Traffic capture uses a **WFP callout driver** on Windows and **`NETransparentProxyProvider`** on macOS (the read-only `NEFilterDataProvider` can allow or drop but not modify). Costs: **certificate-pinned apps can't be filtered** (AdGuard maintains a public [exclusions list](https://github.com/AdguardTeam/HttpsExclusions)), the local CA enlarges attack surface, QUIC/HTTP-3 handling is poor across all interception stacks (practical filters block UDP/443 to force TCP fallback), and Encrypted Client Hello erodes even SNI-based filtering.

### 6.3 Linux

Hosts file, or run AdGuard Home / Pi-hole / dnsmasq locally, or nftables/eBPF redirection of port 53. Same MITM ceiling as above if content-level filtering is wanted.

### 6.4 Android (no root)

- **Local VPN (`VpnService`)** — the standard approach: the app claims the VPN slot with a *loopback* tunnel (nothing leaves via any remote server). Two designs: **DNS-only** (route just the DNS server IPs and filter queries — cheap on battery; DNS66, personalDNSfilter) versus **full userspace stack** (route 0.0.0.0/0, reassemble TCP/UDP, per-app firewall — [RethinkDNS](https://github.com/celzero/rethink-app), Apache-2.0, the best open-source reference; AdGuard for Android adds an HTTPS-filtering local proxy with its own CA).
- **Constraints**: Android allows **one active VPN** — an ad-block VPN and a privacy VPN can't coexist (RethinkDNS embeds WireGuard to solve this). **Google Play policy since Nov 2022 bans `VpnService` use for ad blocking** → AdGuard/Blokada full versions live off-Play (APK/F-Droid); DNS66 is archived and unmaintained (Aug 2026). Note this ban targets *VpnService abuse*, not browsers filtering their own WebView (§9.5).
- **Private DNS (Android 9+)**: point the OS at a filtering DoT resolver (`dns.adguard-dns.com`, `xyz.dns.nextdns.io`) — zero apps, zero battery cost, works on cellular, composes with a real VPN. Hostname-only, so a LAN Pi-hole needs a DoT certificate (e.g. via AdGuard Home). Domain-level only.

### 6.5 iOS

The most restricted platform: **no public API can inspect other apps' HTTPS content** (content-filter extensions require supervised/MDM devices). Available: Safari **Content Blockers** (§5), **encrypted-DNS configuration profiles** (DoH/DoT via the `DNSSettings` payload — [profile generator](https://github.com/paulmillr/encrypted-dns)), and local-VPN apps (`NEPacketTunnelProvider`) that exist mainly to own DNS resolution (AdGuard iOS). Any real VPN overrides these. Effectively: **iOS system-wide = DNS-only; in-Safari = declarative rules.**

### 6.6 Network-level (smart TVs, consoles, IoT — every device on the LAN)

- **Self-hosted DNS sinkhole**: [Pi-hole](https://github.com/pi-hole/pi-hole) (dnsmasq-derived; needs cloudflared/unbound for an encrypted upstream) or [AdGuard Home](https://github.com/AdguardTeam/AdGuardHome) (single Go binary; native DoH/DoT/DoQ in both directions; understands ABP *network* syntax). Hand it out via router DHCP.
- **Router firmware**: OpenWrt `adblock`/`adblock-fast`, pfBlockerNG (pfSense/OPNsense), Diversion (Asuswrt-Merlin).
- **Cloud DNS**: NextDNS / AdGuard DNS / Control D — per-profile blocklists plus CNAME uncloaking, and they follow devices off-LAN via Private DNS (Android) and profiles (iOS/macOS). This is how one filtering config spans all OSes (§7).
- **Limits**: everything in §4, plus devices leaving the LAN (fix: WireGuard/Tailscale home, or use cloud DNS), plus DoH bypass by apps and browsers.

### 6.7 Cross-platform comparison matrix

| Approach | Win | mac | Linux | Android | iOS | Granularity | Cosmetic? | All apps? | Effort to build |
|---|---|---|---|---|---|---|---|---|---|
| Browser extension (Firefox MV2) | ✅ | ✅ | ✅ | ✅ (Firefox) | ❌ | Full URL + DOM | ✅ | ❌ browser only | Medium |
| Browser extension (Chrome MV3 DNR) | ✅ | ✅ | ✅ | ❌ | ❌ | URL (declarative, capped) | limited | ❌ | Medium |
| Safari content blocker | ❌ | ✅ | ❌ | ❌ | ✅ | URL (declarative, 150k/blocker) | CSS-hide only | ❌ Safari only | Medium |
| Built-in engine in own browser fork | ✅ | ✅ | ✅ | ✅ | ⚠️ WebKit-only | Full URL + DOM | ✅ | ❌ that browser | Very high |
| Hosts file | ✅ | ✅ | ✅ | root only | ❌ | Domain | ❌ | ✅ | Trivial |
| Local MITM proxy + root CA | ✅ | ✅ | ✅ | ✅ (AdGuard model) | ❌ | Full URL (except pinned apps) | ✅ in browsers | ⚠️ mostly | Very high |
| Android local VPN (DNS filter) | — | — | — | ✅ | ❌ | Domain | ❌ | ✅ on device | High |
| Private DNS / DNS profile | — | ✅ | ✅ | ✅ | ✅ | Domain | ❌ | ✅ on device | Zero (config) |
| Pi-hole / AdGuard Home on LAN | ✅ | ✅ | ✅ | ✅ | ✅ | Domain (+CNAME chain) | ❌ | ✅ whole network | Low–Medium |
| Cloud filtering DNS (NextDNS-style) | ✅ | ✅ | ✅ | ✅ | ✅ | Domain (+CNAME) | ❌ | ✅ everywhere | High (to build one) |
| **Android WebView browser + `adblock-rust`** | — | — | — | ✅ | ❌ | Full URL + DOM (no POST/WS) | ✅ | ❌ that browser | **Medium** |

---

## 7. DNS providers compared, and AdGuard's business model

### 7.1 Ruled out first (common misconceptions)

- **Quad9** (9.9.9.9): excellent trust (Swiss non-profit) but **does not block ads** — security-only by charter.
- **Cloudflare for Families** (1.1.1.2/.3): malware and adult content only, **no ad blocking**.
- **dns0.eu: shut down** — resolution stopped Jan 15, 2026 (funding). Do not recommend it anywhere.
- **NextDNS free tier**: 300k queries/month, after which it **silently stops filtering** for the rest of the month — a bad failure mode for a set-and-forget router.
- Hobbyist resolvers (AhaDNS, Blahdns, DNSforge, LibreDNS, Alternate DNS): fine as toys, not as a household's only resolver — no anycast footprint, no funded operations; Alternate DNS reportedly logs.

### 7.2 The real alternatives (all free, no account, no cap)

| Provider | Router IPs (plain 53) | Encrypted | Who/where | Notes |
|---|---|---|---|---|
| **AdGuard DNS** | `94.140.14.14` / `94.140.15.15` | DoH/DoT/DoQ `dns.adguard-dns.com` | Cyprus (Russian-origin) | Incumbent; unlimited; on Privacy Guides' recommended list |
| **Control D "Ads & Trackers"** | `76.76.2.2` | DoH/DoT/DoQ `p2.freedns.controld.com` | Canada (Windscribe spin-off) | Best like-for-like swap; no logging on free resolvers; other free profiles: `.3` adult, `.4` family |
| **DNS4EU Protective+Ads** | `86.54.11.13` | DoH/DoT | EU-funded consortium (Whalebone, CZ) | Institutional backing; launched 2025; subject to EU court-ordered blocking; personal use only |
| **Mullvad adblock** | — (encrypted **only**, no plain 53) | `adblock.dns.mullvad.net` (also base/extended/family/all) | Sweden | Strongest trust: audited (Assured), VPN-funded, no accounts. Needs DoT-capable router firmware |
| **RethinkDNS** | — (DoH only) | `sky.rethinkdns.com` | FOSS project on Cloudflare Workers | 190+ configurable lists; great as a client resolver, thin as a sole household dependency |

**Router verdict**: AdGuard and Control D are the two practical picks for a normal router (plain-53 IPs, anycast, no caps). Mullvad wins on trust if the router speaks DoT (OpenWrt/Merlin/pfSense). DNS4EU if you prefer public-institution backing over a private company. Uptime and anycast footprint matter more than blocklist size for your only resolver.

### 7.3 AdGuard's business model (why the free DNS exists)

A bootstrapped company (~$5.4M revenue, ~50 people) with three revenue streams: **paid apps** (the desktop/mobile blockers — free tier limited, paid unlocks system-wide blocking), **AdGuard VPN**, and **B2B/SDK licensing** (white-label filtering tech). The *account-based* AdGuard DNS product has paid tiers (free "Starter" capped at 300k queries/month) — but the **public resolver `94.140.14.14` is uncapped and account-free**: it's a funnel and brand play, plus an anonymized data source. AdGuard Home (the self-hosted OSS server) serves the same funnel role.

**What they log on the public resolver** (their policy, corroborated by Privacy Guides): no personal data; a 24-hour **anonymized** database of requested domains, not linked to requesters, plus aggregate per-server metrics; ECS anonymized. Comparable to Cloudflare's 25-hour retention.

**Trust file**: HQ in Limassol, Cyprus; founded in Moscow 2009, relocated ~2017; the company states it has no servers in Russia (Frankfurt instead) and publicly rebutted the 2022 SetApp allegations; it has friction *with* Russia (its VPN was pushed out of Russian app stores in 2025). Honest weak points: part of engineering is reportedly still in Russia, and there is **no published independent audit** (postponed per their own 2024 statements). No known incidents of data misuse. Privacy Guides still recommends AdGuard public DNS, alongside Cloudflare, Control D, Mullvad and Quad9.

**Bottom line**: AdGuard public DNS is reasonable for a household router; the discomfort is structural (origin, jurisdiction, no audit), not evidentiary. If that's disqualifying: Mullvad with a DoT-capable router is the strongest-trust choice, Control D `76.76.2.2` the best plain-53 swap, DNS4EU the institutional option.

---

## 8. Where `adblock-rust` can actually be integrated

The engine is host-agnostic (§3): anything with a request hook and page injection can use it. The question is what each host can actually provide.

### 8.1 Stock Chrome via an MV3 extension — partial only, a structural ceiling

Chrome MV3 has **no blocking `webRequest`** (observational only, except enterprise force-installed extensions). So `check_network_request` — the heart of the engine — **can never gate a real request** in an extension on stock Chrome. The proof-of-shape is [gasanache/brave-shields-extension](https://github.com/gasanache/brave-shields-extension): it does *not* use the engine for network blocking at all — it ships pre-compiled static `declarativeNetRequest` rulesets (a capped, lossy subset of the lists) and uses the WASM engine **only for cosmetic filtering**, plus hand-written YouTube/Twitch hacks.

So on stock Chrome an extension gets **full cosmetics and scriptlets, plus DNR-subset network blocking** — roughly the uBO Lite ceiling with extra steps.

**Exceptions that get full power without a custom build:**

- **Enterprise-policy force-installed** extensions keep blocking `webRequest` (`ExtensionSettings` policy) — viable for a managed-devices product, not consumers. **This loophole is ruled out for personal machines**: MV2 is dead everywhere (the policy override was removed in Chrome 139), and MV3 + `webRequestBlocking` survives *only* for policy force-installed extensions; on unmanaged Windows, force-install is limited to Web-Store-hosted extensions (off-store requires an AD domain join), and Google is actively closing policy abuse on unmanaged devices. Unverified on macOS/Linux; assume it narrows further. Not a foundation to build on.
- **Firefox**: MV2 *and* MV3 keep blocking `webRequest` → a Firefox extension embedding the WASM engine gets the **entire** engine — network, cosmetics, scriptlets — on desktop and Android.
- **Stock Firefox 149+ literally ships `adblock-rust` already** — disabled, no UI, driven by two `about:config` prefs (`privacy.trackingprotection.content.protection.enabled` and `...test_list_urls`); [adblock-rust-manager](https://github.com/electricant/adblock-rust-manager) is an extension that flips them. Experimental and unofficial, but real.

### 8.2 Local MITM proxy — the only full-power path into unmodified Chrome

Point stock Chrome, or the whole OS, at a local filtering proxy that terminates TLS with a locally-trusted CA. Deployment: install the CA into the OS trust store and set the system proxy (Chrome follows it) or pass `--proxy-server=127.0.0.1:8100`.

- **[Zen](https://github.com/anfragment/zen)** (Go, MIT, Windows/macOS/Linux, active releases through July 2026) — **the maintained open-source option.** System-wide filtering proxy with EasyList syntax, cosmetic filtering and scriptlets via HTML rewriting, a generated local root CA, hosts lists, and a "security" list category; accepts arbitrary list URLs. It uses its own Go engine, not `adblock-rust`. No IP-layer or DNS enforcement.
- **[Privaxy](https://github.com/Barre/privaxy)** (Rust, AGPL-3.0, ~2.5k★) — architecturally the reference design: hyper + rustls MITM, per-host leaf certificates from a generated CA, the `adblock` crate for network decisions, and Cloudflare's `lol_html` streaming rewriter to inject cosmetic CSS and scriptlets into HTML responses; ~50 MB RAM with ~320k filters. **It is dead — last commit January 2023**, pinned to `adblock` 0.6.0, with no maintained fork found. Reviving it on the current crate is a genuine project opportunity (§13).
- **AdGuard for Windows/Mac** is the commercial equivalent (~$2.49/month or $79.99 lifetime).

**Class-wide proxy limits**: certificate-pinned apps break or need exclusions; **HTTP/3 and QUIC are not proxied** (block UDP/443 to force TCP fallback, or traffic silently bypasses the filter); the initiator is approximated from `Referer`/`Origin`, so third-party matching degrades; TLS re-encryption plus HTML rewriting costs latency.

### 8.3 Other hosts

| Host | How | Power |
|---|---|---|
| **Electron apps** | `session.webRequest.onBeforeRequest` (Electron's own API — MV3 doesn't apply, blocking works) + `adblock-rs` native module + a preload script for CSS/scriptlets. Turnkey comparison: `@ghostery/adblocker-electron` (`blocker.enableBlockingInSession(...)`) | **Full** |
| **Puppeteer / Playwright** (scrapers, test rigs, ad-measurement crawlers) | Request interception → engine check; `addStyleTag`/`addInitScript` for cosmetics. Ghostery ships `PuppeteerBlocker`/`PlaywrightBlocker`; real crates.io dependents: `spider_network_blocker`, `chromey`, Brave's `pagegraph` | **Full** |
| **Android WebView app** | `shouldInterceptRequest` + Rust via JNI/UniFFI; `addDocumentStartJavaScript` for cosmetics/scriptlets. Shipping precedents: [Edsuns/AdblockAndroid](https://github.com/Edsuns/AdblockAndroid), [webspace_app](https://github.com/theoden8/webspace_app). See §9 | **~78–83%** |
| **iOS / WKWebView** | No interception API at all. The engine's `content-blocking` feature converts lists → Apple content-blocker JSON (Brave's "Slim List" pipeline). Conversion **drops** scriptlets, `$redirect`, `$csp`, `$removeparam`, procedural cosmetics, most regex | Weak |
| **LAN/router-wide proxy** | A MITM proxy bound to 0.0.0.0 — works in principle, but every device needs the CA installed; Android 7+ apps ignore user CAs, and TVs/consoles can't install one | Full-but-impractical |
| **DNS server** | **Wrong tool.** No DNS API — you'd fake `http://domain/` and only `\|\|domain^` rules would match; path/type/party rules (most of EasyList) and all cosmetics are lost. Use AdGuard DnsLibs or a plain domain-set matcher; real "adblock DNS" projects compile lists to domain sets instead of embedding the engine | Not suitable |
| **Non-Rust hosts via IPC** | [adblock-rust-server](https://github.com/dudik/adblock-rust-server): a tiny daemon on a Unix socket (`n <url> <source> <type>` → block?, `c <url> ...` → CSS) — a clean pattern for embedding from any language | Full (network + cosmetic data) |

### 8.4 Verdict table

| Integration point | Network | Cosmetic | Scriptlets | Verdict |
|---|---|---|---|---|
| Custom browser build (Brave, Ladybird, Firefox 149) | ✅ | ✅ | ✅ | Full — but you're shipping a browser |
| **Stock Chrome, MV3 extension** | ⚠️ DNR subset | ✅ | ✅ | **Partial — hard ceiling, no per-request hook** |
| Stock Chrome, enterprise force-install | ✅ | ✅ | ✅ | Full, but managed devices only — ruled out for personal use |
| **Firefox extension (MV2/MV3, WASM engine)** | ✅ | ✅ | ✅ | **Full — desktop + Android** |
| **Local MITM proxy (Zen / Privaxy model)** | ✅ | ✅ | ✅ | **Full on stock browsers; CA/pinning/QUIC costs** |
| Electron app | ✅ | ✅ | ✅ | Full |
| Puppeteer/Playwright | ✅ | ✅ | ✅ | Full |
| **Android WebView** | ⚠️ URL-only, no POST/WS | ✅ | ✅ | **Partial (~78–83%) — the build-worthy gap** |
| iOS/WKWebView | ⚠️ converted JSON | ⚠️ hide-only | ❌ | Weak |
| DNS | ⚠️ domain rules only | ❌ | ❌ | Not suitable |

---

## 9. The Android lite WebView browser

**The question**: can we build a small-APK Android browser on the Chromium engine with Brave-level ad blocking, given that Chromium forks (Brave) and Firefox are both heavy and Firefox Lite is dead (discontinued 30 June 2021)?

**The answer**: yes, and the trick is **Android System WebView**. You cannot make a *Chromium fork* small — but you don't need to fork Chromium to use its engine. Android System WebView **is** Chromium: preinstalled on every device and security-updated by Google via Play. A WebView-based browser ships zero engine bytes — a 5–15 MB APK instead of ~190 MB — with the same Blink/V8 rendering performance, and Google maintains the engine for you.

### 9.1 Why every Chromium fork is 150 MB+

A fork bundles Blink, V8, the network stack, locales and codecs:

| Browser | Size | Notes |
|---|---|---|
| Cromite v148 (May 2026) | **198 MB** arm64 APK | verified from [GitHub releases](https://github.com/uazo/cromite/releases) |
| Brave Android | ~175–190 MB | secondary source, unverified |
| Chrome Android | "27 MB" listings are a split-APK stub; real Trichrome footprint >100 MB | Play doesn't publish install size |
| Firefox / Iceraven (GeckoView) | ~100–130 MB arm64 | Iceraven 123 MB via APKMirror (secondary) |
| Opera Mini | tiny — but only because "extreme mode" renders pages **server-side** on Opera's proxy (OBML); not a real local engine | [Wikipedia](https://en.wikipedia.org/wiki/Opera_Mini) |

Slimming a fork (dropping locales, ffmpeg codecs) saves tens of MB, not an order of magnitude — Blink+V8 is the floor.

### 9.2 The gap in existing lite browsers

Real lite WebView browsers — Via (claims <1 MB, realistically a few MB), SmartCookieWeb (<8 MB), Lightning, FOSS Browser, Soul, Hermit — all sit in the 3–15 MB band, roughly 20–40× smaller than any fork. **All of them ship only crude hosts/domain-list blocking**: no cosmetic engine, no scriptlets. That is the gap a proper engine integration fills.

### 9.3 WebView interception — verified facts

Verified against Chromium source (`android_webview/browser/network_service/`), AOSP javadoc, Chromium's own WebView test suite, and shipping apps. The claim "you cannot intercept or change Android WebView requests" is **false** — but it has one true kernel.

**What definitively works:**

- `shouldInterceptRequest` fires for **every http(s)/data/file subresource and navigation, including HTTPS** — it's a callback *above* the network stack, inside the app's own WebView, before the fetch is issued. No MITM, no certificate needed. It fires for the main frame, iframes, XHR/fetch, images and workers, and also for requests servable from the HTTP cache.
- **Blocking works**: return a `WebResourceResponse` with an **empty** stream. A `null` stream triggers `onReceivedError` — use empty, not null.
- **Replacing works**: serve arbitrary bytes/MIME/status, so uBO `$redirect=` stubs work exactly as designed.
- **`Referer` is guaranteed present** — Chromium explicitly sets it immediately before the intercept callback (its own test asserts this), so initiator and third-party classification are sound. Combined with `isForMainFrame()` this reconstructs the `source_url` that `Request::new()` needs.
- **Cosmetic/scriptlet injection has first-class support**: `WebViewCompat.addDocumentStartJavaScript` (androidx.webkit 1.5+, feature-gated — check `WebViewFeature.DOCUMENT_START_SCRIPT`, fall back to `onPageStarted` + `evaluateJavascript`) runs before page scripts and inside iframes.
- **Decisive shipping precedent**: [Edsuns/AdblockAndroid](https://github.com/Edsuns/AdblockAndroid) runs **Brave's ad-block engine via JNI**, blocking through `shouldInterceptRequest`, with cosmetic and scriptlet injection — literally "the Brave engine in a WebView", already working. [theoden8/webspace_app](https://github.com/theoden8/webspace_app) does the same: the `adblock` crate behind JNI (`AdblockEngineNative.checkUrl` in a native subresource interceptor), request type and source derived from headers, plus **full cosmetics** — document-start CSS hides, a page-side procedural shim (`:has-text()`, `:upward()`, `:remove()`), generic hides fed by a DOM class/id scanner (`hiddenClassIdSelectors`), and `$redirect` stub serving, with lists (EasyList, HaGeZi, ClearURLs) downloaded at runtime rather than bundled.

**What you cannot do (the kernel of truth):**

- **You cannot modify a request and let it continue.** The API is answer-or-pass-through: return a response, or return null and the request proceeds *unmodified*. No header mutation, no in-place URL rewrite. So `$removeparam`, `$csp` and `$removeheader` are out — except via a full re-fetch with your own HTTP client, which is not viable generally (cookies aren't shared; you lose Chromium's cache, H2 and H3).
- **POST/PUT bodies are invisible** (verified in the source: the request struct has no body field; [chromium 40436253](https://issues.chromium.org/issues/40436253)). Body-dependent rules are unavailable and POST beacons slip through.
- **`Sec-Fetch-Dest` is almost certainly NOT visible** in intercepted headers (~85% confidence from source reading): Chromium sets fetch-metadata headers *downstream* of the interception point, by design, as anti-spoofing. **Classify `request_type` from the `Accept` header plus the URL extension instead** — the heuristic DuckDuckGo's and AdblockAndroid's classifiers use: `image/*`→image, `text/css`→stylesheet, `*javascript*`→script, `text/html` on a subresource→subdocument, `font/`→font. First implementation task: log the real header map on-device once to confirm.
- **WebSockets and WebTransport are not interceptable**; service workers need `ServiceWorkerControllerCompat.setServiceWorkerClient`.
- **Redirects: only hop 1 is matched** — a tracker that 302s to another tracker is checked only on the first URL. A real but small gap.
- The callback is synchronous on a background thread, so the engine must answer in microseconds (`adblock-rust` does; load a preserialized DAT).
- Platform limits: no extensions ever, no WebUSB or Web Bluetooth, a single profile (the new Profile API aside). Rendering and JS performance are at parity — the same Blink and V8.
- Likely origins of the "it doesn't work" myth: confusing `shouldOverrideUrlLoading` (navigation-only, skipped for POST navigations) with `shouldInterceptRequest`, or the pre-API-21 URL-only overload, obsolete since 2014.

**Net power: ~78–83% of the full engine** — docked from the earlier 80–85% estimate for the redirect-hop gap and the `Sec-Fetch-Dest` correction, neither fatal. Realistically ~90% of uBO's user-visible result: full network rules for the GET subresources that make up nearly all ads, plus full cosmetics and scriptlets. Better than every hosts-based lite browser today, and better than Cromite on scriptlets.

### 9.4 The alternatives, for comparison

- **[Cromite](https://github.com/uazo/cromite)** (the Bromite successor): a maintained Chromium fork with integrated ABP-derived blocking *including* cosmetic rules and a subset of snippets, service-worker and WebSocket filtering, and CNAME uncloaking. Full-engine power, 198 MB, and someone else runs the fork. Weaker than uBO/Brave on scriptlets.
- **Brave Android**: full `adblock-rust`, mainstream support, ~175–190 MB.
- **Our own Chromium fork: unrealistic.** A ≥100 GB checkout, 7–20 hour builds, and a rebase treadmill that accelerates to a **2-week Chrome release cycle from Sept 2026**. Vanadium (GrapheneOS) shipped four releases in ten days in Aug 2026 just tracking Chromium — and doesn't even attempt an adblocker. Cromite and Vanadium are each roughly one maintainer's full-time job.

### 9.5 The recommended build

**WebView browser + `adblock-rust` over JNI (or UniFFI):**

1. Rust `cdylib` per ABI (~2–5 MB total) wrapping `Engine` (`check_network_request`, `url_cosmetic_resources`, `hidden_class_id_selectors`), with engine state as a serialized DAT cached on disk.
2. `shouldInterceptRequest` → build `Request` from URL + **`Accept` header/URL extension** (resource type) + `Referer` + main-frame flag → empty `WebResourceResponse` on block, stub response on `$redirect`.
3. `addDocumentStartJavaScript` → inject hide-CSS + scriptlets + a MutationObserver reporting new class/ids back for generic hides.
4. `ServiceWorkerControllerCompat.setServiceWorkerClient` so service-worker traffic is filtered too.
5. Runtime filter-list downloader (EasyList, EasyPrivacy, uBO filters, HaGeZi) with periodic refresh — keeps the APK tiny and lists fresh without app updates.
6. Distribution note: a *browser* with ad blocking is fine on Play (the Nov 2022 policy targets `VpnService` abuse, not browsers filtering their own WebView) — Via and SmartCookieWeb ship on Play with blockers. Verify current Play policy at submission time, and see §11 for developer verification.

### 9.6 AABrowser as a possible shell — assessed, and rejected for this purpose

[kododake/AABrowser](https://github.com/kododake/AABrowser) is a **plain Android WebView browser** (Kotlin, Views, `android.webkit.WebView` + androidx.webkit 1.16) marketed for **Android Auto head units** — not a projection app, just an Activity declaring the car-launcher categories so AA lists it (users enable AA "unknown sources"). Active and popular (464★, created Oct 2025, last push Jul 2026, v2.2), GPL-3.0, ~7,400 LOC across 31 well-organized files, no tests. Aggressive platform floor: **minSdk 35 (Android 15+), compileSdk 37** (preview toolchain). No native code or NDK at all. Realistic APK ~5–8 MB. Quirks you'd strip: a self-hosted Umami analytics pinger, `usesCleartextTraffic=true` and `MIXED_CONTENT_ALWAYS_ALLOW`.

**Architecture fit is surprisingly good**: it has **no ad blocking of any kind today** (the README says so, "contributions welcome!"; open issues [#120](https://github.com/kododake/AABrowser/issues/120) and [#87](https://github.com/kododake/AABrowser/issues/87) request it). All WebView setup funnels through **one function** — `web/ConfiguredWebView.kt:47` `configureWebView()`, whose anonymous `WebViewClient` (`:99`) has **no `shouldInterceptRequest` override**, so an interceptor drops straight in; there's a single call site per tab (`tabs/TabManager.kt:177`), so multi-tab is already centralized. The androidx.webkit feature-gating idiom is established (`WebViewFeature.*` checks, `addWebMessageListener` at `TabManager.kt:199-205`), though `addDocumentStartJavaScript` isn't used yet and existing JS injection happens too late (`onPageFinished`, `ConfiguredWebView.kt:140`). A working page↔native bridge exists to copy for the generic-cosmetics class/id feedback loop. No `ServiceWorkerControllerCompat`. Licensing is fine: a GPLv3 shell and an MPL-2.0 engine are compatible. Adding the engine would be entirely additive, roughly 1–2 weeks, no hard blockers — the same five steps as §9.5, with the JNI bridge greenfield (adding ~2–5 MB of `.so`, roughly doubling the APK).

**Verdict: legitimate project, wrong shell for a general-purpose lite browser.** Everything hard about the ad blocker is portable and AABrowser contributes none of it. What it contributes is 7k LOC of *car-head-unit* chrome (800×480 landscape UI, giant touch targets), an Android 15+ floor that excludes most phones, and defaults we'd strip. Ranked:

1. **Fresh minimal WebView browser**, using webspace_app and AdblockAndroid as references for the hard parts (header-based request reconstruction, procedural-cosmetics shim, redirect stubs) — the recommended path.
2. **AABrowser as a reading reference** — `ConfiguredWebView.kt` and `TabManager.kt` are a clean model of tab lifecycle and androidx.webkit gating. There's also a genuine opportunity in its niche: it owns the Android Auto browser market and ad blocking is its most-requested missing feature, so **contributing our blocker upstream to issue #120** would ship the engine work to real users without owning a browser.
3. Don't fork it into a phone browser.

---

## 10. The "personal SWG" concept

**Concept**: an on-device app (no cloud proxy) that pulls public blocklists and threat feeds (GitHub-hosted etc.) and enforces them locally with an `adblock-rust`-style engine — Netskope/Zscaler shape, personal scale.

**Verdict up front**: the concept is sound and **mostly already exists** in pieces. **Portmaster** is the DNS + per-app-firewall version, **Zen** is the MITM-proxy version, and the feeds are free and well-maintained. The honest description of a v1 project is *"Portmaster (or Zen) plus better curated lists"* — about 90% configuration, not code. Building a fresh agent is justified only for the union nobody ships: one app doing DNS + per-app IP blocking + URL-level MITM filtering with unified policy across OSes — which is real systems work (kernel drivers, macOS entitlements, certificate lifecycle) for marginal gain. Recommended path: prototype as a curated feed-bundle/config layer over an existing agent first.

### 10.1 Prior art

- **[Safing Portmaster](https://github.com/safing/portmaster)** (GPL-3.0, Go) — the closest match: an application firewall (WFP kernel driver on Windows, nfqueue/eBPF on Linux) that intercepts *all* DNS, even apps' stray queries, applies ad/tracker/malware lists, and enforces per-app rules. Acquired by IVPN in Dec 2024, actively maintained (v2.2.x through Aug 2026). This *is* the "personal Zscaler, no-MITM tier".
- **[Zen](https://github.com/anfragment/zen)** (MIT, Go) — the MITM tier; see §8.2.
- **AdGuard desktop** — the commercial precedent for exactly this pairing: MITM filtering plus "Browsing Security" (a local hash-prefix database of ~1.5M phishing/malware sites, Safe-Browsing-style, so full URLs never leave the machine).
- **AdGuard Home / Pi-hole on localhost** with threat lists = the DNS-only tier, available today.
- **RethinkDNS** = the Android analogue (`VpnService`: DNS + per-app firewall + 190+ lists including threat feeds).
- Per-app firewalls with list support: **OpenSnitch** (Linux; subscribes to domain/regex/IP/hash lists), **simplewall** (Windows, IP-oriented), **LuLu** (macOS — but macOS can only hostname-block Apple-framework connections; for Chrome and Firefox only IP blocking works, a real platform constraint).
- Nothing markets itself as an "open-source personal SWG" — the *name* is a gap, the functionality mostly isn't.
- **Overlap warning**: browsers already ship Safe Browsing/SmartScreen, so malicious-*website* blocking in the browser is duplicated work. The genuine additions of a local agent are non-browser processes (installers, droppers, Electron apps), per-app policy, system-wide ad blocking, and raw-IP C2 blocking.

### 10.2 The free threat feeds

- **abuse.ch** family — URLhaus (malware **URLs**, path-level, exactly what a URL engine can use and DNS can't), ThreatFox (IOCs; entries older than 6 months auto-expire since May 2025), Feodo Tracker (botnet **C2 IPs**), SSLBL. CC0, but **now requires a free auth key** (auth.abuse.ch).
- **[malware-filter/urlhaus-filter](https://gitlab.com/malware-filter/urlhaus-filter)** — URLhaus repackaged into uBO/AdGuard/hosts/RPZ formats, >260k filters, updated twice daily; siblings: phishing-filter, pup-filter. **The easiest way to feed URLhaus into `adblock-rust` or Zen.**
- **[HaGeZi TIF](https://github.com/hagezi/dns-blocklists)** — aggregated malicious domains (mini ~174k → full ~2.15M; also a plain-IP variant), daily.
- **[Phishing Army](https://phishing.army/)** — phishing domains from PhishTank/OpenPhish/CERT.pl and others, refreshed 6-hourly.
- **IP level**: **Spamhaus DROP** (criminal netblocks; eDROP merged into DROP in 2024), **FireHOL level1** (~3.9k subnets aggregate — some upstreams rotting, verify freshness), blocklist.de, CINS Army.
- **Google Safe Browsing Update API** — a local hash-prefix database (privacy-preserving), free for **non-commercial** use only.
- PhishTank: effectively closed to new API keys — don't depend on it.

**Honest gap versus real Zscaler**: these feeds are reactive and confirmed-bad-only — no sandboxing, no ML verdicts on fresh domains, no URL categorization, no DLP. Phishing infrastructure churns in hours; expect to catch commodity malware C2 and yesterday's phishing kits, not targeted or brand-new threats. Community aggregates also carry false positives.

### 10.3 Architecture tiers, if building

| Tier | Mechanism | Catches | Misses | Cost |
|---|---|---|---|---|
| (a) DNS-only | local resolver + ad/TIF domain lists | most ads/trackers/malware domains, all apps | URL paths, raw-IP C2, DoH bypass, cosmetics | trivial — already solved by AdGuard Home on localhost |
| (b) DNS + per-app firewall (Portmaster model) | + WFP/nftables/NEFilter + a CIDR trie over DROP/FireHOL/Feodo | + C2 by IP, per-app policy | URL paths, cosmetics | kernel-adjacent code; **no MITM or certificate burden** — the sweet spot |
| (c) Full MITM proxy (Zen/AdGuard model) | local CA + proxy + `adblock-rust` (ads + URLhaus URL rules) + CONNECT-IP checks | everything including paths, plus cosmetic rewriting | pinned apps, HTTP/3 pain | certificate trust, attack surface |

**Recommendation**: for personal use now, **Portmaster + HaGeZi TIF + Spamhaus DROP/FireHOL** (tier b, no certificates), or **Zen + urlhaus-filter + phishing-filter** (tier c, URL-level). Build only if the unified all-tier agent proves necessary — and if we do, it merges naturally with the desktop-proxy project in §13: a Privaxy revival plus threat feeds plus an IP matcher is the differentiated product.

---

## 11. Android developer verification

Status as of 2 Sept 2026. This matters for distributing anything from §9.

### 11.1 The "120 days" claim is a myth

Nothing in Google's documentation, blog posts, or press coverage contains a 120-day element. **Already-installed apps are never disabled** — enforcement blocks *new installs and updates*, not execution of what's installed. Likely confusions: Apple's 7-day free-provisioning expiry, Play's 180-day inactive-account closure, or Play's 12-testers/14-days rule.

### 11.2 The actual rules and timeline

- Apps must be registered (package plus signing key) to an identity-verified developer to install normally on **certified** devices (i.e. GMS phones). A full account requires government ID and a $25 one-time fee (organizations: D-U-N-S).
- **30 Sep 2026**: enforcement starts in Brazil, Indonesia, Singapore and Thailand — **but only for participating app stores** (Play, Galaxy Store, Xiaomi GetApps, etc.). Google's own FAQ states that direct sideloading (browser, GitHub, F-Droid) is **not covered in this phase**. Press claiming "sideloading blocked Sept 30" is wrong.
- **2027+**: global expansion to all install sources on certified devices; exact mechanics and dates unpublished. **India is in the 2027+ wave with no date set**, and regulator pressure there is live.

### 11.3 Escape hatches (all officially confirmed)

1. **Limited Distribution tier** (live Aug 2026): **free, no government ID**, up to **20 authorized devices** per account, installs via a QR/link handshake. Purpose-built for personal, family and friends apps — this answers the side-project question: yes, you can build and share without the fee or ID.
2. **ADB installs are permanently exempt.** `adb install` bypasses the gate entirely.
3. **Advanced flow** (live): a one-time power-user unlock (Developer options → reboot → ~24h wait → biometric confirm), after which unverified APKs install with an "install anyway" warning, indefinitely. It syncs across the user's devices.
4. **Non-certified ROMs** (LineageOS, GrapheneOS, /e/OS): no gate at all.

### 11.4 What it means for our project

Nothing changes today anywhere (GitHub isn't a participating store). From the 2027 wave, users on stock phones need the one-time advanced flow or ADB; existing installs keep working, and updates are what break for unverified apps. F-Droid is fighting the scheme (open letter Feb 2026, 60+ organizations) because its re-signing model makes *it* the "developer"; the outcome is unsettled. For our own distribution: the Limited Distribution tier covers a friends-and-family circle; a full account ($25 + ID) only if going public at scale; or public GitHub APKs relying on advanced-flow/ADB literacy — plausible for an ad-blocker-browser audience.

---

## 12. Hard limits nobody escapes

- **Server-side ad insertion** (YouTube, Twitch since 2016): ads share the content's domain, manifest and segments — only fragile client-side player patching helps (§4a).
- **Certificate pinning** blocks MITM filtering for many apps (banking, Firefox, most mobile SDKs).
- **QUIC/HTTP-3 plus Encrypted Client Hello** erode both interception and SNI-based filtering; current tools force TCP fallback by blocking UDP/443.
- **No request mutation in WebView** — answer-or-pass-through only, so `$removeparam`/`$csp`/`$removeheader` are structurally unavailable there, as are POST bodies and WebSockets (§9.3).
- **Policy walls**: Google Play bans ad-blocking `VpnService` apps; Apple confines iOS to DNS plus Safari declarative rules; Chrome Web Store enforces MV3; Android developer verification tightens sideloading from 2027 (§11).
- **The arms race**: anti-adblock detectors, randomized subdomains and paths, near-daily filter-list churn. A blocker is a *service* — continuous list updates — not a ship-once artifact.

---

## 13. Build options, ranked

### Worth building

1. **Android lite WebView browser + `adblock-rust`** (§9). Fills a real market gap: every existing lite WebView browser has only hosts-list blocking, and the only alternatives with real engines are 175–198 MB forks. Small scope, ~78–83% of full engine power, two shipping precedents to learn from, and Play distribution is permitted. **Recommended first project.**
2. **Desktop Rust local filtering proxy — a Privaxy revival on current `adblock-rust`** (§8.2). The only full-power path into stock Chrome and every other desktop app. Competes with Zen (Go, maintained) and AdGuard desktop (commercial). Costs: root CA lifecycle, pinned-app exclusions, QUIC handling. Merges naturally with the threat-feed agent in §10 — `adblock-rust` for ads plus URLhaus path rules, plus a separate CIDR matcher for IP feeds, is the differentiated version.
3. **Firefox (MV2/MV3) extension** for depth, if a browser-side component is wanted with the least work: full `webRequest` + cosmetics + scriptlets, desktop and Android. Reuse [`@ghostery/adblocker`](https://github.com/ghostery/adblocker) rather than writing matching logic.
4. **Contribute the blocker upstream to AABrowser issue #120** (§9.6) — a low-cost way to ship the engine work to real users in a niche that has no competition, without owning a browser.

### Not worth building

5. **Our own DNS resolver product.** AdGuard Home already exists and is excellent — *run* it, don't rewrite it. A DNS server is also the wrong host for the engine entirely (§8.3).
6. **A Chrome MV3 extension.** The ceiling is uBO Lite and it's already occupied; `check_network_request` can never gate a real request there (§8.1).
7. **The enterprise-policy force-install loophole.** Structurally narrowing, unmanaged-device force-install is Web-Store-only, Google is actively closing it. Not a foundation (§8.1).
8. **A Chromium fork.** ≥100 GB checkout, 7–20 hour builds, a 2-week rebase treadmill from Sept 2026; Cromite and Vanadium are each a full-time maintainer (§9.4).
9. **A desktop system-wide MITM app with kernel drivers** (a full AdGuard clone, as opposed to the proxy in option 2): per-OS driver and network-extension work, root-CA trust burden, cert-pinning exclusions, QUIC pain. Not a first project.
10. **A fresh unified "personal SWG" agent.** ~90% of the value is configuration over Portmaster or Zen; the systems work for the remaining union is disproportionate (§10).

---

## 14. Source notes, corrections, and uncertainty flags

### 14.1 Corrections applied during research (superseded claims)

| Earlier claim | Corrected version | Source of correction |
|---|---|---|
| `Sec-Fetch-Dest` is available in `shouldInterceptRequest` headers and can classify resource type | **`Sec-Fetch-Dest` is almost certainly NOT visible** (Chromium adds fetch-metadata headers downstream of the intercept point, by design). Classify from **`Accept` header + URL extension**. `Referer` *is* reliable and guaranteed set | Chromium source reading, 2026-09-02 |
| "No established Android WebView + `adblock-rust` project found" | **Precedent exists**: webspace_app, and decisively **Edsuns/AdblockAndroid**, which runs Brave's engine via JNI through `shouldInterceptRequest` with cosmetics and scriptlets | Repo review, 2026-08-28 / 2026-09-02 |
| Privaxy is "dormant since mid-2024" | **Privaxy is dead — last commit January 2023**, pinned to `adblock` 0.6.0, no maintained fork. **Zen** is the maintained open-source local proxy | Repo history, 2026-08-28 |
| WebView reaches ~80–85% of engine power | **~78–83%** — docked for the redirect-hop-1-only gap and the `Sec-Fetch-Dest` correction | 2026-09-02 verification |
| dns0.eu is a viable free ad-blocking resolver | **dns0.eu is shut down** — resolution stopped Jan 15, 2026 | Techzine/Cybernews, 2026-08-28 |
| NextDNS free tier is a drop-in router option | **Free tier silently stops filtering** after 300k queries/month — a bad failure mode for a set-and-forget router | NextDNS docs |

### 14.2 What was read directly

Brave internals (engine structures, tokenization, precedence, cosmetic pipeline, cargo features, content-blocking conversion losses) were read from `brave/adblock-rust` and `brave/brave-core` source on GitHub. WebView interception behavior was verified against Chromium's `android_webview/browser/network_service/`, AOSP javadoc, and Chromium's WebView test suite. AABrowser findings are from its repository at the file/line references given in §9.6. brave-shields-extension architecture, Privaxy architecture (`Cargo.toml`/README), Zen status (repo releases), and Cromite sizes (GitHub releases API) come from those repositories. Filter-list details are from the uBO wiki (static-filter syntax, MV3 incompatibilities), the uBOL FAQ, MDN `declarativeNetRequest`, the WebKit content-blockers blog, Apple `DNSSettings` docs, and the Pi-hole / AdGuard Home / RethinkDNS / StevenBlack / HaGeZi repositories. Circumvention research: CV-Inspector (NDSS 2021) and Nithyanand et al. (FOCI '16). DNS provider facts: mullvad/dns-blocklists on GitHub, Control D docs, adguard-dns.io privacy policy, Privacy Guides; AdGuard company history via Wikipedia/Crunchbase, and the SetApp rebuttal from AdGuard's blog. Chrome DoH auto-upgrade behavior from Chromium's own DoH provider documentation; policy restrictions from Google enterprise docs and the chromium-extensions list. Firefox 149 pref names via the adblock-rust-manager project, not Mozilla docs.

### 14.3 Unverified or secondary — treat with caution

- **Egress-blocked during research**: brave.com, adguard.com, easylist.to, developer.chrome.com, electronjs.org, F-Droid pages. Anything sourced from them here is secondary reporting.
- **Performance and version figures**: the 5.7 µs / 69× `adblock-rust` benchmark comes from secondary reporting. MV3 DNR rule budgets and Electron cancel semantics come from MDN and long-standing API contracts — re-verify exact numbers before relying on them.
- **Dates in flux**: the Chrome Web Store's final MV2 purge date (reported Aug 31, 2026) and Edge desktop's current uBO status are secondary-source.
- **APK sizes**: exact Brave, Firefox and Via/lite-browser sizes are from project READMEs, APKMirror or marketing; Play does not publish install sizes. Re-measure before quoting. Cromite's 198 MB is verified.
- **Block-rate percentages**: no rigorous measurement of "what fraction of ads does AdGuard DNS block" exists. Vendor numbers (e.g. Control D's) are marketing. Treat any single percentage skeptically — including the ~80% figure in §1, which is a request-count intuition, not a measurement.
- **`Sec-Fetch-Dest` absence is source-derived, not device-verified** (~85% confidence). Log the real header map on-device as the first implementation task.
- **Policy viability**: macOS/Linux self-managed force-install viability is unverified; assume the enterprise-policy route narrows further.
- **Android verification 2027 mechanics** and dates; whether limited-distribution device slots can be reassigned; possible regulator-forced carve-outs (EU/India).
- **DNS providers**: DNS4EU logging retention and post-grant funding; Control D anycast node count; whether AdGuard's 24-hour domain database feeds list curation (inferred, not stated).
- **Threat feeds**: Zen's default security-list contents and repo ownership chain; PhishTank key availability; per-source freshness inside FireHOL aggregates.
