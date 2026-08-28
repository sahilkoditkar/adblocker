# Ad Blocking Across Platforms — Research Report

Research date: 2026-08-28. Goal: map every practical way to block ads on PC (Windows/macOS/Linux), Android, and iOS — browser-level, system-level, and network-level — before choosing what to build.

---

## TL;DR

1. **There is no single mechanism that blocks all ads on all platforms.** Every real-world solution is a *stack* of layers, each with a different power/coverage trade-off:
   - **DNS/domain blocking** = widest coverage (every app, every device), lowest precision.
   - **Browser-level filtering** (extension or built-in engine) = highest precision (URL paths, cosmetic cleanup, scriptlets), narrowest coverage (that one browser).
   - **On-device MITM proxy** (AdGuard desktop style) = in-between: content-level filtering for all apps on one machine, but defeated by certificate pinning and increasingly by QUIC/ECH.
2. **Your suspicion about domain lists is correct**: domain-only blocking structurally cannot block (a) first-party ads served from the content's own domain (YouTube, Facebook, Twitch — including server-side ad insertion where ads are stitched into the same video stream), (b) anything requiring path-level granularity (`cdn.site.com/ads/x.js` vs `cdn.site.com/app.js` are indistinguishable at DNS), and (c) it leaves broken layouts/empty boxes because it can't do cosmetic filtering. Details in §4.
3. **Brave doesn't use an extension** — it compiles EasyList/uBO-syntax filters in a Rust engine (`adblock-rust`, MPL-2.0, reusable) wired directly into Chromium's network stack. That engine (or Ghostery's TypeScript equivalent) is the best starting point if we build anything browser-side. Details in §2.
4. **Extensions are no longer "the" reliable way on Chrome.** Chrome's Manifest V3 killed full uBlock Origin (MV2 is fully removed as of mid-2026); declarative rules can't do dynamic filtering or scriptlets. Full-power extensions now only exist on **Firefox** (desktop *and* Android). Details in §5.
5. **Cross-OS "block everywhere" is possible but only at DNS granularity**: a DNS sinkhole (Pi-hole/AdGuard Home self-hosted, or NextDNS/AdGuard DNS cloud) reached via router DHCP on the LAN, Android Private DNS (DoT) and iOS encrypted-DNS profiles off-LAN. Anything deeper than DNS has to be built per-OS (§6).
6. **Android without root**: the standard trick is a *local* `VpnService` "loopback VPN" that intercepts DNS (or all traffic) on-device — how Blokada, RethinkDNS, DNS66, AdGuard for Android work. Google Play policy since Nov 2022 bans ad-blocking VpnService apps, so distribution is F-Droid/APK sideload. Details in §6.4.
7. **Recommended architecture if we build**: start with a DNS-level blocker (works everywhere, smallest scope), and reuse an existing filter engine (`adblock-rust` or `@ghostery/adblocker`) for any browser-side component rather than writing a matcher from scratch. Options ranked in §8.

---

## 1. The three layers of ad blocking

Every blocker does some subset of:

| Layer | What it does | Requires | Examples |
|---|---|---|---|
| **Network filtering** | Cancel/redirect ad requests before they load | Seeing the request URL + context (initiator, resource type) | uBO, Brave Shields, AdGuard proxy |
| **DNS filtering** | Make ad domains not resolve (sinkhole → `0.0.0.0`/NXDOMAIN) | Controlling the resolver | Pi-hole, AdGuard Home, NextDNS, hosts file |
| **Cosmetic + scriptlet filtering** | Hide leftover DOM elements (`##.ad-slot`), inject JS to defuse ad SDKs and anti-adblock detectors | Running code inside the page | uBO content scripts, Brave's isolated-world injection |

DNS sees only hostnames. Network filtering sees full URLs + request context. Cosmetic/scriptlet filtering sees the DOM. The further down this list, the cleaner the result — and the harder it is to deploy outside a browser.

---

## 2. How Brave implements it (and what's reusable)

Brave Shields is **not** an extension — it's native browser code, which is exactly why it escapes all Manifest V3 restrictions.

**Engine — [`adblock-rust`](https://github.com/brave/adblock-rust)** (crate `adblock`, MPL-2.0):
- Parses ABP/uBO filter syntax (EasyList etc.) into `NetworkFilter` structs with a bitmask for options (`$script`, `$third-party`, `$important`, anchors, …); `$domain=` values stored as hashes.
- Matching uses **tokenization + a reverse index**, not bloom filters: each filter is bucketed under its *least frequent* token (frequency histogram computed across all filters); at request time the URL is tokenized the same way, so only the few filters sharing a token get evaluated. Brave reported ~5.7 µs per request on EasyList+EasyPrivacy, ~69× faster than their old C++ bloom-filter engine (figure from secondary reporting; brave.com blog was unreachable during research).
- Filter buckets are evaluated in precedence order: `$important` → normal filters → exceptions (`@@`) → `$redirect`/`$removeparam`/`$csp`. Engines serialize to a binary DAT for fast startup.

**Chromium integration**: the hook is `OnBeforeURLRequest_AdBlockTPPreWork` in [`brave_ad_block_tp_network_delegate_helper.cc`](https://github.com/brave/brave-core/blob/master/browser/net/brave_ad_block_tp_network_delegate_helper.cc), inside Brave's proxying URL loader — it gets real initiator origin, frame tree, and Chromium resource type, computes third-partyness, calls `AdBlockEngine::ShouldStartRequest()` on a task runner, and on match blocks or substitutes a stub response. Being in the browser also enables CNAME uncloaking and seeing requests extensions can't.

**Cosmetic/scriptlet side**: rules are split hostname-specific vs generic; generic rules are keyed by class/id so the renderer reports which classes/ids actually exist on the page (via Mojo `HiddenClassIdSelectors()`) and only matching selectors come back. Injection runs in an isolated world before page scripts; procedural operators (`:has-text()`, `:upward()`, …) are evaluated by a bundled in-page observer script. Scriptlets are uBO-compatible ([brave/adblock-resources](https://github.com/brave/adblock-resources)).

**Lists**: catalog of ~91 lists in [`adblock-lists`](https://github.com/brave/adblock-lists)' `list_catalog.json` — defaults include uBO filters, EasyList, EasyPrivacy, URLhaus, cookie-notice lists, plus Brave's own. Delivered as signed CRX components via Chromium's component updater (no store review needed for list updates).

**Reusability**: MPL-2.0 with Rust, Node (`adblock-rs`), Python and WASM bindings. Reported embedders: Mozilla (experimental, disabled-by-default prototype landed ~March 2026), Ladybird, Waterfox, Perplexity Comet. Comparable embeddable engines:
- **[`@ghostery/adblocker`](https://github.com/ghostery/adblocker)** (TypeScript, MPL-2.0) — same reverse-index idea, ~99% uBO filter compat, ready adapters for WebExtensions, **Electron, Puppeteer, Playwright**, Node. Easiest to embed in a JS app.
- **AdGuard CoreLibs** — *not* actually open (repo is README-only). **[AdGuard DnsLibs](https://github.com/AdguardTeam/DnsLibs)** *is* open (Apache-2.0, C++ with Java/C# bindings) but DNS-level only.

**Even Brave can't block everything**: server-side ad insertion (§4a), first-party ads (which Brave *by policy* doesn't target except in Aggressive mode — [blocking policy](https://github.com/brave/brave-browser/wiki/Blocking-goals-and-policy)), ~a-day lag in the anti-adblock arms race, and a much weaker iOS version (App Store confines it to WebKit content-blocker JSON, a converted "Slim List").

---

## 3. The open-source filter lists

Two fundamentally different kinds:

**URL/DOM-aware lists (EasyList family)** — usable only by engines that see full requests and the DOM:
- EasyList, EasyPrivacy (dual GPLv3 / CC BY-SA 3.0), uBlock filters (GPLv3), AdGuard filters (GPLv3), Fanboy lists (CC BY 3.0), Peter Lowe's (non-commercial).
- Syntax power: full-URL patterns (`||cdn.example.com/assets/ads/*`), options like `$third-party`, `$script`, `$domain=`, exceptions `@@`, and DOM rules: `##selector` hiding, `#?#` procedural filters, `##+js(...)` scriptlets.

**Domain-only lists (hosts-style)** — the only kind DNS blockers can use:
- [StevenBlack/hosts](https://github.com/StevenBlack/hosts) (aggregator, ~2k–168k domains depending on variant), [OISD](https://oisd.nl/) ("zero breakage" philosophy; ABP-syntax `||domain^` only since 2024), [HaGeZi](https://github.com/hagezi/dns-blocklists) (tiered Light→Ultimate, ~42k–269k entries), AdAway, 1Hosts. Mostly GPL-3.0.
- One bit of information per hostname. Notably, **HaGeZi's own README says DNS blockers "can't catch everything"** and recommends pairing with a browser content blocker.

Licensing caveat: the FSF notes it's unclear filter lists are copyrightable at all, so the licenses are partly nominal — but attribution is cheap, keep it.

---

## 4. Validated: why domain/DNS blocking can't block everything

This confirms the concern in the original question, with specifics:

**(a) First-party ads & server-side ad insertion (SSAI).** YouTube serves ad video segments and content segments from the same `*.googlevideo.com` hosts and the same API origin — blocking the domain kills playback entirely. Twitch's SureStream stitches ad segments *into the same HLS manifest* as the stream (same CDN hostname, same URL shape). Facebook serves ads first-party with randomized/obfuscated markup. Against SSAI, **no network-layer blocker of any kind helps** — only client-side player patching (scriptlets rewriting `ytInitialPlayerResponse` etc.), which is exactly the layer DNS and MV3-declarative blockers don't have.

**(b) No path granularity.** DNS resolves names; the path/query never reaches the resolver and is encrypted under HTTPS anyway. A shared CDN is all-or-nothing.

**(c) No cosmetic filtering.** A sinkholed request still leaves the `<div>`/`<iframe>` in the DOM: empty boxes, broken carousels, "content failed to load" placeholders.

**(d) CNAME cloaking** — trackers hide behind first-party aliases (`analytics.publisher.com` CNAME → `tracker.evil.net`). Naive domain lists miss it; uBO uncloaks only on Firefox (Chromium has no DNS API for extensions); recursive DNS blockers actually *see* the CNAME chain, so this is one place DNS is the *stronger* layer.

**(e) DNS bypass.** Apps/devices with hardcoded resolvers (Chromecast → 8.8.8.8) or built-in DoH clients ignore your DNS entirely. Countermeasures: NAT-redirect all port 53, block port 853 (DoT falls back to plain DNS), blocklist known DoH endpoints — inherently incomplete since DoH is indistinguishable HTTPS on 443.

**Arms race**: measured research (CV-Inspector, NDSS 2021; Nithyanand et al., FOCI '16) shows circumvention uses randomized high-entropy subdomains and paths — exactly the case static domain lists lose and URL/scriptlet rules counter. Hence the universal recommendation: **DNS blocking for breadth + browser blocker for depth.**

---

## 5. Browser extensions — how they work and where they still work

**Manifest V2 (the uBlock Origin model)**: blocking `chrome.webRequest.onBeforeRequest` runs the extension's *code* synchronously per request → arbitrary logic, dynamic per-site filtering, `$redirect` stubs, header-based rules; content scripts at `document_start` do cosmetic + procedural filtering; **scriptlet injection** patches page globals before site code runs (the #1 anti-adblock weapon); filter lists update as data, no store review.

**Manifest V3 (Chrome today)**: `declarativeNetRequest` — a static rule table matched by the browser; the extension never sees requests. Limits: 30k static rules guaranteed (+ shared pool to ~330k), 30k dynamic, capped RE2-only regex. Lost: dynamic filtering, response-header rules, per-site switches, most scriptlet power. **MV2 is gone from Chrome**: disabling began Oct 2024, enterprise policy ended with Chrome 138 (July 2025), last flag workarounds stripped in Chrome 150–151 (mid-2026). uBlock Origin **Lite** is the MV3 fallback — no custom filters, no element picker, cosmetic/scriptlet filtering only with per-site opt-in.

**Firefox is the holdout**: ships both APIs, has said MV2/`webRequest` is not deprecated → **full uBO works on Firefox desktop and Firefox for Android** (open extension ecosystem since Firefox 120, Dec 2023). This is the only mobile configuration with desktop-grade blocking.

**Safari**: declarative-only Content Blocker JSON (`WKContentRuleList`), 150k rules per blocker (was 50k); AdGuard for Safari ships **six** content-blocker extensions to reach ~900k rules. The blocker never sees traffic (privacy-by-design), so no request-time logic.

**Chrome on Android**: no extensions, and none planned for phones (the "Desktop Android" convergence build that loads extensions targets large-screen devices — dev-channel, not phones). Kiwi Browser is dead (Jan 2025). Alternatives: Firefox Android + uBO, Brave/Vivaldi built-in, Edge Android (built-in ABP + a small curated store incl. uBO Lite).

**Verdict**: extensions are still the strongest *in-browser* method — but only where full extensions still exist (Firefox). On Chrome the ceiling is uBO Lite; Brave's built-in engine now beats anything installable on Chrome. So no, extensions are not the only way, and on Chromium they're no longer even the best way.

---

## 6. System-wide blocking, per OS

### 6.1 Hosts file (Windows / macOS / Linux; Android needs root)
Map ad domains to `0.0.0.0` in `/etc/hosts` / `drivers\etc\hosts`. Free, no software. Limits: **no wildcards** (every subdomain listed literally → 50k–700k-line files), Windows DNS-cache perf issues at scale, domain granularity only, needs admin + `ipconfig /flushdns`. Android: root required ([AdAway](https://github.com/AdAway/AdAway)).

### 6.2 Windows & macOS content-level (the AdGuard-desktop model)
The only way to filter *inside* HTTPS system-wide is **MITM of your own traffic**: a local filtering proxy + a locally generated root CA installed in the system trust store; TLS is terminated, filtered, re-encrypted. Traffic capture: **WFP callout driver** on Windows; on macOS, **`NETransparentProxyProvider`** (Network Extension — the read-only `NEFilterDataProvider` can allow/drop but not modify). Costs: **certificate-pinned apps can't be filtered** (AdGuard maintains a public [exclusions list](https://github.com/AdguardTeam/HttpsExclusions)), the local CA enlarges attack surface, QUIC/HTTP-3 handling is poor across all interception stacks (practical filters block UDP/443 to force TCP fallback), and Encrypted Client Hello erodes even SNI-based filtering.

### 6.3 Linux
Hosts file, or run AdGuard Home/Pi-hole/dnsmasq locally, or nftables/eBPF redirection of port 53. Same MITM ceiling as above if content-level filtering is wanted.

### 6.4 Android (no root)
- **Local VPN (`VpnService`)** — the standard approach: app claims the VPN slot with a *loopback* tunnel (nothing leaves via any remote server). Two designs: **DNS-only** (route just the DNS server IPs; filter queries — cheap on battery; DNS66, personalDNSfilter) vs **full userspace stack** (route 0.0.0.0/0, reassemble TCP/UDP, per-app firewall — [RethinkDNS](https://github.com/celzero/rethink-app), Apache-2.0, the best open-source reference; AdGuard for Android adds an HTTPS-filtering local proxy with its own CA).
- **Constraints**: Android allows **one active VPN** — an ad-block VPN and a privacy VPN can't coexist (RethinkDNS embeds WireGuard to solve this). **Google Play policy (Nov 2022) bans VpnService use for ad blocking** → AdGuard/Blokada full versions live off-Play (APK/F-Droid); DNS66 is archived/unmaintained (Aug 2026).
- **Private DNS (Android 9+)**: point the OS at a filtering DoT resolver (`dns.adguard-dns.com`, `xyz.dns.nextdns.io`) — zero apps, zero battery cost, works on cellular, composes with a real VPN. Hostname-only (a LAN Pi-hole needs a DoT cert, e.g. via AdGuard Home). Domain-level only.

### 6.5 iOS
The most restricted: **no public API can inspect other apps' HTTPS content** (content-filter extensions require supervised/MDM devices). Available: Safari **Content Blockers** (§5), **encrypted-DNS configuration profiles** (DoH/DoT via `DNSSettings` payload — [profile generator](https://github.com/paulmillr/encrypted-dns)), and local-VPN apps (`NEPacketTunnelProvider`) that exist mainly to own DNS resolution (AdGuard iOS). Any real VPN overrides these. Effectively: **iOS system-wide = DNS-only; in-Safari = declarative rules.**

### 6.6 Network-level (covers smart TVs, consoles, IoT — every device on the LAN)
- **Self-hosted DNS sinkhole**: [Pi-hole](https://github.com/pi-hole/pi-hole) (dnsmasq-derived; needs cloudflared/unbound for encrypted upstream) or [AdGuard Home](https://github.com/AdguardTeam/AdGuardHome) (single Go binary; native DoH/DoT/DoQ both directions; understands ABP *network* syntax). Hand out via router DHCP.
- **Router firmware**: OpenWrt `adblock`/`adblock-fast`, pfBlockerNG (pfSense/OPNsense), Diversion (Asuswrt-Merlin).
- **Cloud DNS**: NextDNS / AdGuard DNS / ControlD — per-profile blocklists + CNAME uncloaking, and they follow devices off-LAN via Private DNS (Android) / profiles (iOS/macOS) — this is how one filtering config spans all OSes.
- **Limits**: everything in §4, plus devices leaving the LAN (fix: WireGuard/Tailscale home, or use cloud DNS), plus DoH bypass by apps/browsers.

---

## 7. Comparison matrix

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

---

## 8. If we build something — options ranked

1. **DNS-level blocker (best first project, truly cross-OS).** A filtering DNS server (self-hostable like AdGuard Home, and/or a hosted DoH/DoT endpoint) consuming OISD/HaGeZi/StevenBlack lists. Reaches Windows/macOS/Linux/Android/iOS/TVs with zero client software (Private DNS on Android, profiles on iOS, DHCP on LAN). Reuse [AdGuard DnsLibs](https://github.com/AdguardTeam/DnsLibs) or write a resolver in Go/Rust. Accept §4's limits and document them.
2. **Firefox (MV2) extension** for depth: full webRequest + cosmetic + scriptlets, works on desktop and Android. Reuse [`@ghostery/adblocker`](https://github.com/ghostery/adblocker) as the engine instead of writing matching logic.
3. **Chrome MV3 extension** only for reach, knowing the ceiling is uBO-Lite-level.
4. **Android local-VPN app** (`VpnService`, DNS-filtering mode first): study [RethinkDNS](https://github.com/celzero/rethink-app) and personalDNSfilter; plan for F-Droid/APK distribution, not Play.
5. **Desktop system-wide MITM app** (AdGuard clone): highest power outside browsers, but driver/network-extension work per OS, root-CA trust burden, cert-pinning exclusions, QUIC pain. Not a first project.
6. **Browser fork with `adblock-rust` built in** (Brave's path): maximum quality, enormous maintenance (tracking Chromium). Embedding `@ghostery/adblocker` in an Electron/WebView shell is the lightweight cousin.

**The pragmatic stack** (what power users assemble today, and the gap analysis for any product): AdGuard Home/NextDNS for everything + Brave or Firefox+uBO for browsing + Android Private DNS / iOS profile when off-LAN.

---

## 9. Hard limits nobody escapes

- **Server-side ad insertion** (YouTube testing it broadly, Twitch since 2016): ads share the content's domain, manifest, and segments — only fragile client-side player patching helps.
- **Certificate pinning** blocks MITM filtering for many apps (banking, Firefox, most mobile SDKs).
- **QUIC/HTTP-3 + Encrypted Client Hello** erode both interception and SNI-based filtering; current tools force TCP fallback by blocking UDP/443.
- **Policy walls**: Google Play bans ad-block VpnService apps; Apple confines iOS to DNS + Safari declarative rules; Chrome Web Store enforces MV3.
- **The arms race**: anti-adblock detectors, randomized subdomains/paths, ~daily filter-list churn. A blocker is a *service* (continuous list updates), not a ship-once artifact.

## 10. Source notes & uncertainty flags

Claims above about Brave internals were read directly from `brave/adblock-rust` and `brave/brave-core` source on GitHub; performance figures (5.7 µs / 69×) and some dates come from secondary reporting because brave.com, adguard.com, easylist.to and developer.chrome.com were unreachable from the research environment. Chrome Web Store's final MV2 purge date (reported Aug 31, 2026) and Edge desktop's current uBO status are secondary-source and in flux. Vendor block-rate percentages (e.g. ControlD's) are marketing numbers. Key primary sources: uBO wiki (static-filter syntax, MV3 incompatibilities), uBOL FAQ, MDN `declarativeNetRequest`, WebKit content-blockers blog, Apple `DNSSettings` docs, Pi-hole/AdGuard Home/RethinkDNS/StevenBlack/HaGeZi repos, CV-Inspector (NDSS '21) and Nithyanand et al. (FOCI '16) papers.
