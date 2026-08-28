# Final Recommendation — Blocking Ads Everywhere

Research date: 2026-08-28. Synthesis of `RESEARCH.md`, `adblock-rust-integration.md`, `android-lite-browser.md`, plus final validation research on router-level DNS and stock-Chrome options.

## The answer in one line

**Router-level AdGuard DNS is the right base layer and genuinely solves most of the *web* ad problem network-wide — but it's ~80% of requests, not 80% of perceived annoyance (YouTube: 0%). Layer a browser-side blocker on top for the rest; no custom browser build is required on desktop, and on Android the WebView-browser route is the build-worthy project.**

## Layer 1 — Router DNS (the "generic, cross-platform" base)

**Setup**: home router WAN/DHCP DNS fields take **IP addresses, not hostnames**. Use AdGuard DNS public servers: `94.140.14.14` / `94.140.15.15` (free, no signup, no query cap). `dns.adguard-dns.com` is the DoT/DoH hostname — that one goes in **Android Private DNS** and iOS/macOS profiles for when devices leave home, not in the router. NextDNS at a router needs its linked-IP dance (dynamic-IP households need a DDNS updater) — AdGuard's fixed IPs are simpler. Routers running OpenWrt/Asuswrt-Merlin can use encrypted upstream (DoT/DoH) or run **AdGuard Home on the router itself** (GL.iNet ships it in the GUI) for custom lists and per-client rules with no query caps.

**What it solves**: third-party display/programmatic ads and trackers in *every* app on *every* device — browsers, mobile apps, smart TVs. Nice surprise: Chrome's "automatic" Secure DNS **auto-upgrades AdGuard's IPs to AdGuard's own DoH** (they're in Chromium's provider list), so filtering survives Chrome's encrypted DNS by default. The bypass case is a user manually selecting Google/Cloudflare in `chrome://settings/security`.

**What it does not solve** (validated):
- **YouTube ads: essentially 0%** — same `googlevideo.com` hosts for ads and video; NextDNS's own docs say DNS blocking can't touch them. Same for Twitch, Facebook/Instagram feed ads, Spotify.
- Cosmetic leftovers — empty boxes, "disable your ad blocker" walls.
- Hardcoded-DNS devices (Roku ships hardcoded 8.8.8.8) — needs router NAT-redirect of port 53 + blocking 853, which consumer firmware can't do (OpenWrt/Merlin/pfSense can).
- Devices off your home network — cover with Android Private DNS / iOS profile pointing at the same AdGuard hostname.

**Honest "80%" assessment**: fair for general web browsing measured in requests/bytes (Pi-hole installs typically sinkhole 15–25% of all DNS queries; academic work confirms DNS filtering works but under-performs full extension blockers). Optimistic measured in *ads you actually see*, because the most annoying category (first-party video ads) is in the 0% bucket.

## Layer 2 — In-browser, per platform (the "addon, not a browser" tier)

| Where | Best option without building anything | Notes |
|---|---|---|
| Desktop, if browser choice is free | **Brave** (or Firefox + uBO) | Full engine built in; nothing to add |
| **Desktop, staying on Chrome** | **uBO Lite (Complete mode) + the DNS layer**, or a **local MITM proxy** for full power | See below |
| Android | Brave (heavy) or **our WebView browser** (see Layer 3) | Chrome Android has no extensions |
| iOS | AdGuard/Brave content blockers + AdGuard DoT profile | Platform ceiling; nothing better is possible |

### Staying on stock Chrome — validated conclusions

1. **Extension route (uBO Lite in Complete mode)**: covers plain network ads + cosmetics + scriptlets per-site; combined with the DNS layer this is the community-consensus Chrome stack. Ceiling: no dynamic filtering, capped rules, slow anti-adblock response.
2. **The enterprise-policy loophole is not practical for personal machines**: MV2 is dead everywhere (policy override removed in Chrome 139; Web Store purge Aug 31, 2026). MV3 + `webRequestBlocking` survives *only* for policy force-installed extensions, and on unmanaged Windows force-install is limited to Web-Store-hosted extensions (off-store needs AD domain join); Google is actively closing policy abuse on unmanaged devices. Unverified on macOS/Linux; assume it narrows further. Not a foundation to build on.
3. **Local MITM proxy — the only full-power path into stock Chrome, and there IS a maintained open-source one now**: **[Zen](https://github.com/anfragment/zen)** (Go, Windows/macOS/Linux, active releases through July 2026) — system-wide filtering proxy with EasyList syntax, cosmetic filtering and scriptlets via HTML rewriting, local root CA. The commercial equivalent is AdGuard for Windows/Mac (~$2.49/mo or $79.99 lifetime). Privaxy (the Rust/adblock-rust one) is confirmed dead (last commit Jan 2023). **If we build for desktop, the highest-leverage project is a Privaxy-revival: adblock-rust + modern TLS + lol_html — i.e., an open Rust competitor to Zen/AdGuard.**

## Layer 3 — Android (the build-worthy gap)

Nothing installable on stock Chrome Android gives real blocking, Brave is ~190 MB, Firefox is heavy, Firefox Lite is dead. Per `android-lite-browser.md`: a **WebView-based browser (5–15 MB APK, Chromium engine from the OS) + adblock-rust via JNI** reaches ~80–85% of Brave's engine (full cosmetics/scriptlets; network blocking minus POST/WebSocket/`$csp`) — better than every existing lite browser and better than Cromite on scriptlets. This is the recommended build target. Fallbacks that require no building: Cromite (198 MB, maintained fork) or Brave.

## The recommended stack, assembled

1. **Now, zero build**: AdGuard DNS IPs on the router + Android Private DNS/iOS profile off-network + Brave on desktop (or Chrome + uBOL Complete + Zen if staying on Chrome) + Brave/Cromite on Android.
2. **Build option A (Android, recommended first)**: the lite WebView browser with adblock-rust — fills a real market gap, small scope, per `android-lite-browser.md` §4.
3. **Build option B (desktop)**: Rust local-proxy blocker (Privaxy revival on current adblock-rust) — full-power blocking for stock Chrome and every desktop app, competing with Zen/AdGuard desktop.
4. **Not worth building**: our own DNS resolver product (AdGuard Home already exists and is excellent — *run* it, don't rewrite it), a Chrome MV3 extension (ceiling already occupied by uBOL), or a Chromium fork (2-week rebase treadmill).

## Verification notes

Chrome DoH auto-upgrade behavior from Chromium's own DoH provider docs; policy-route restrictions from Google enterprise docs + chromium-extensions list; YouTube-over-DNS impossibility from NextDNS/Pi-hole sources; Zen status from its repo releases. Unverified: exact AdGuard DNS ad-block percentage claims (no rigorous measurement exists — treat any single number skeptically), macOS/Linux self-managed force-install viability.
