# A "Personal Zscaler" — Local App Blocking Ads + Malicious Sites from Public Feeds

Research date: 2026-08-28. Concept: an on-device app (no cloud proxy) that pulls public blocklists/threat feeds (GitHub-hosted etc.) and enforces them locally with an adblock-rust-style engine — Netskope/Zscaler shape, personal scale.

## Verdict up front

The concept is sound and **mostly already exists** in pieces: **Portmaster** is the DNS+per-app-firewall version, **Zen** is the MITM-proxy version, and the feeds you'd want are free and well-maintained. The honest description of the v1 project is *"Portmaster (or Zen) + better curated lists"* — ~90% configuration, not code. Building a fresh agent is justified only for the union nobody ships: one app doing DNS + per-app IP blocking + URL-level MITM filtering with unified policy across OSes — real systems work (kernel drivers, macOS entitlements, cert lifecycle) for marginal gain. Recommended path: prototype as a curated feed-bundle/config layer over an existing agent first.

## Prior art

- **[Safing Portmaster](https://github.com/safing/portmaster)** (GPL-3.0, Go) — the closest match: application firewall (WFP kernel driver on Windows, nfqueue/eBPF on Linux) that intercepts *all* DNS (even apps' stray queries), applies ad/tracker/malware lists, per-app rules. Acquired by IVPN in Dec 2024, actively maintained (v2.2.x through Aug 2026). This *is* the "personal Zscaler, no-MITM tier."
- **[Zen](https://github.com/anfragment/zen)** (MIT, Go) — local MITM proxy with a generated root CA, EasyList-syntax + hosts lists including a "security" category; accepts arbitrary list URLs. Own Go engine (not adblock-rust). No IP-layer or DNS enforcement.
- **AdGuard desktop** — commercial precedent for exactly this pairing: MITM filtering + "Browsing Security" (local hash-prefix DB of ~1.5M phishing/malware sites, Safe-Browsing-style so full URLs never leave the machine).
- **AdGuard Home / Pi-hole on localhost** with threat lists = the DNS-only tier, available today.
- **RethinkDNS** = the Android analogue (VpnService: DNS + per-app firewall + 190+ lists incl. threat feeds).
- Per-app firewalls with list support: **OpenSnitch** (Linux; subscribes to domain/regex/IP/hash lists), **simplewall** (Windows, IP-oriented), **LuLu** (macOS — but macOS can only hostname-block Apple-framework connections; for Chrome/Firefox only IP blocking works, a real platform constraint).
- Nothing markets itself as an "open-source personal SWG" — the *name* is a gap, the functionality mostly isn't.
- **Overlap warning**: browsers already ship Safe Browsing/SmartScreen, so malicious-*website* blocking in the browser is duplicated work. The genuine additions of a local agent: non-browser processes (installers, droppers, Electron apps), per-app policy, system-wide ad blocking, raw-IP C2 blocking.

## The free threat feeds (the "open source malicious domains and IPs")

- **abuse.ch** family — URLhaus (malware **URLs**, path-level — exactly what a URL engine can use and DNS can't), ThreatFox (IOCs; entries >6 months auto-expire since May 2025), Feodo Tracker (botnet **C2 IPs**), SSLBL. CC0, but **now requires a free auth key** (auth.abuse.ch).
- **[malware-filter/urlhaus-filter](https://gitlab.com/malware-filter/urlhaus-filter)** — URLhaus repackaged into uBO/AdGuard/hosts/RPZ formats, >260k filters, updated 2×/day; siblings: phishing-filter, pup-filter. **The easiest way to feed URLhaus into adblock-rust or Zen.**
- **[HaGeZi TIF](https://github.com/hagezi/dns-blocklists)** — aggregated malicious domains (mini ~174k → full ~2.15M; also a plain-IP variant), daily.
- **[Phishing Army](https://phishing.army/)** — phishing domains from PhishTank/OpenPhish/CERT.pl etc., 6-hourly.
- **IP level**: **Spamhaus DROP** (criminal netblocks; eDROP merged into DROP in 2024), **FireHOL level1** (~3.9k subnets aggregate — some upstreams rotting, verify freshness), blocklist.de, CINS Army.
- **Google Safe Browsing Update API** — local hash-prefix DB (privacy-preserving), free for **non-commercial** use only.
- PhishTank: effectively closed to new API keys — don't depend on it.

**Honest gap vs. real Zscaler**: these feeds are reactive, confirmed-bad-only — no sandboxing, no ML verdicts on fresh domains, no URL categorization, no DLP. Phishing infrastructure churns in hours; expect to catch commodity malware C2 and yesterday's phishing kits, not targeted or brand-new threats. Community aggregates also carry false positives.

## Architecture tiers (if building)

| Tier | Mechanism | Catches | Misses | Cost |
|---|---|---|---|---|
| (a) DNS-only | local resolver + ad/TIF domain lists | most ads/trackers/malware domains, all apps | URL paths, raw-IP C2, DoH bypass, cosmetics | trivial — already solved by AdGuard Home localhost |
| (b) DNS + per-app firewall (Portmaster model) | + WFP/nftables/NEFilter + CIDR trie over DROP/FireHOL/Feodo | + C2 by IP, per-app policy | URL paths, cosmetics | kernel-adjacent code; **no MITM/cert burden** — sweet spot |
| (c) Full MITM proxy (Zen/AdGuard model) | local CA + proxy + adblock-rust (ads + URLhaus URL rules) + CONNECT-IP checks | everything incl. paths + cosmetic-ish rewriting | pinned apps, HTTP/3 pain | cert trust, attack surface |

**Engine facts**: adblock-rust natively ingests plain hosts lists (`FilterFormat::Hosts`) — so one engine covers ad rules + domain rules + URLhaus path rules. It has **no IP concept**: IP feeds need a separate longest-prefix-match/CIDR set matcher at the firewall or CONNECT layer.

## Recommendation

For personal use now: **Portmaster + HaGeZi TIF + Spamhaus DROP/FireHOL** (tier b, no certs), or **Zen + urlhaus-filter + phishing-filter** (tier c, URL-level). Build only if the unified all-tier agent proves necessary — and if we do build, it merges naturally with the desktop-proxy project from `RECOMMENDATION.md` (Privaxy-revival + threat feeds + an IP matcher = the differentiated product).

Unverified: Zen's default security-list contents and repo ownership chain; PhishTank key availability; per-source freshness inside FireHOL aggregates.
