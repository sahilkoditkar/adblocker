# Free Ad-Blocking Public DNS — Alternatives to AdGuard DNS & Trust Analysis

Research date: 2026-08-28. Question: is AdGuard DNS the only good free ad-blocking resolver, and what's their business model if it's free?

## Ruled out first (common misconceptions)

- **Quad9** (9.9.9.9): superb trust (Swiss non-profit) but **does not block ads** — security-only by charter.
- **Cloudflare for Families** (1.1.1.2/.3): malware/adult only, **no ad blocking**.
- **dns0.eu: shut down** — resolution stopped Jan 15, 2026 (funding).
- **NextDNS free**: 300k queries/month, then **silently stops filtering** for the rest of the month — bad failure mode for a set-and-forget router.
- Hobbyist resolvers (AhaDNS, Blahdns, DNSforge, LibreDNS, Alternate DNS): fine as toys, not as a household's only resolver — no anycast footprint, no funded ops; Alternate DNS reportedly logs.

## The real alternatives (all free, no account, no cap)

| Provider | Router IPs (plain 53) | Encrypted | Who/where | Notes |
|---|---|---|---|---|
| **AdGuard DNS** | `94.140.14.14` / `94.140.15.15` | DoH/DoT/DoQ `dns.adguard-dns.com` | Cyprus (Russian-origin) | Incumbent; unlimited; on Privacy Guides' recommended list |
| **Control D "Ads & Trackers"** | `76.76.2.2` | DoH/DoT/DoQ `p2.freedns.controld.com` | Canada (Windscribe spin-off) | Best like-for-like swap; no logging on free resolvers; other free profiles: .3 adult, .4 family |
| **DNS4EU Protective+Ads** | `86.54.11.13` | DoH/DoT | EU-funded consortium (Whalebone, CZ) | Institutional backing; launched 2025; subject to EU court-ordered blocking; personal use only |
| **Mullvad adblock** | — (encrypted **only**, no plain 53) | `adblock.dns.mullvad.net` (also base/extended/family/all) | Sweden | Strongest trust: audited (Assured), VPN-funded, no accounts. Needs DoT-capable router firmware |
| **RethinkDNS** | — (DoH only) | `sky.rethinkdns.com` | FOSS project on Cloudflare Workers | 190+ configurable lists; great as a client resolver, thin as sole household dependency |

**Router verdict**: AdGuard and Control D are the two practical picks for a normal router (plain-53 IPs, anycast, no caps). Mullvad wins on trust if the router can speak DoT (OpenWrt/Merlin/pfSense). DNS4EU if you prefer public-institution backing over a private company. Uptime/anycast matters more than blocklist size for your only resolver.

## AdGuard's business model (why the free DNS exists)

Bootstrapped company (~$5.4M revenue, ~50 people), three revenue streams: **paid apps** (the desktop/mobile blockers — free tier limited, paid unlocks system-wide blocking), **AdGuard VPN**, and **B2B/SDK licensing** (white-label filtering tech). The *account-based* AdGuard DNS product has paid tiers (free "Starter" capped at 300k queries/month) — but the **public resolver `94.140.14.14` is uncapped and account-free**: it's a funnel and brand play, plus an anonymized data source. AdGuard Home (the self-hosted OSS server) serves the same funnel role.

**What they log on the public resolver** (their policy, corroborated by Privacy Guides): no personal data; a 24-hour **anonymized** database of requested domains (not linked to requesters) + aggregate per-server metrics; ECS anonymized. Comparable to Cloudflare's 25h retention.

**Trust file**: HQ Limassol, Cyprus; founded Moscow 2009, relocated ~2017; company states no servers in Russia (Frankfurt) and publicly rebutted the 2022 SetApp allegations; friction *with* Russia (their VPN pushed out of Russian app stores in 2025). Honest weak points: part of engineering reportedly still in Russia, and **no published independent audit** (postponed per their own 2024 statements). No known incidents of data misuse. Privacy Guides still recommends AdGuard public DNS (alongside Cloudflare, Control D, Mullvad, Quad9).

**Bottom line**: AdGuard public DNS is reasonable for a household router; the discomfort is structural (origin/jurisdiction, no audit), not evidentiary. If that's disqualifying: Mullvad (with a DoT-capable router) is the strongest-trust choice; Control D `76.76.2.2` is the best plain-53 swap; DNS4EU the institutional option.

## Source notes

dns0.eu shutdown via Techzine/Cybernews; Mullvad variants from mullvad/dns-blocklists GitHub; Control D free-tier logging status via Privacy Guides + Control D docs; AdGuard logging from adguard-dns.io privacy policy; company history via Wikipedia/Crunchbase; SetApp rebuttal from AdGuard's blog. Unverified: DNS4EU logging retention and post-grant funding; Control D anycast node count; whether AdGuard's 24h domain DB feeds list curation (inferred).
