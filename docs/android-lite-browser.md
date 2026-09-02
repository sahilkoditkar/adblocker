# A Lite Android Browser with the Chromium Engine + adblock-rust

Research date: 2026-08-28. Question: can we build a small-APK Android browser on the Chromium engine with Brave-level ad blocking, given that Chromium forks (Brave) and Firefox are both heavy and Firefox Lite is dead (discontinued June 2021)?

## TL;DR — yes, and the trick is Android System WebView

You cannot make a *Chromium fork* small — but you don't need to fork Chromium to use its engine. **Android System WebView IS Chromium**, preinstalled on every device and security-updated by Google via Play. A WebView-based browser ships zero engine bytes: 5–15 MB APK instead of ~190 MB, same Blink/V8 rendering performance, and Google maintains the engine for you. adblock-rust integrates via JNI/FFI and reaches **~80–85% of Brave's blocking power** — including full cosmetic filtering and scriptlets, which no existing lite WebView browser ships today. A working open-source precedent exists ([webspace_app](https://github.com/theoden8/webspace_app)).

## 1. Why every Chromium fork is ~150 MB+ (verified numbers)

A fork bundles Blink + V8 + network stack + locales + codecs:

| Browser | Size | Notes |
|---|---|---|
| Cromite v148 (May 2026) | **198 MB** arm64 APK | verified from [GitHub releases](https://github.com/uazo/cromite/releases) |
| Brave Android | ~175–190 MB | secondary source, unverified |
| Chrome Android | "27 MB" listings are a split-APK stub; real Trichrome footprint >100 MB | Play doesn't publish install size |
| Firefox / Iceraven (GeckoView) | ~100–130 MB arm64 | Iceraven 123 MB via APKMirror (secondary) |
| Opera Mini | tiny — but only because "extreme mode" renders pages **server-side** on Opera's proxy (OBML); not a real local engine | [Wikipedia](https://en.wikipedia.org/wiki/Opera_Mini) |

Slimming a fork (drop locales, ffmpeg codecs) saves tens of MB, not an order of magnitude — Blink+V8 is the floor. **Firefox Lite** was discontinued 30 June 2021 ([Mozilla](https://support.mozilla.org/en-US/kb/end-support-firefox-lite)); Kiwi died Jan 2025.

## 2. The WebView route

Real lite WebView browsers: Via (claims <1 MB, realistically a few MB), SmartCookieWeb (<8 MB), Lightning, FOSS Browser, Soul, Hermit — all in the 3–15 MB band, ~20–40× smaller than any fork. **All of them ship only crude hosts/domain-list blocking** — no cosmetic engine, no scriptlets. That's the gap a proper engine integration would fill.

### Integrating adblock-rust — better than the naive reading suggests

The standard objection is "`shouldInterceptRequest` only gives you a URL." In practice `WebResourceRequest.getRequestHeaders()` exposes request headers including **`Referer`** (guaranteed set by Chromium right before the callback), which with `isForMainFrame()` reconstructs the initiator/`source_url` input `Request::new()` needs. **Correction (2026-09-02, see `webview-interception-and-verification.md`)**: `Sec-Fetch-Dest` is likely *not* visible (Chromium adds fetch-metadata headers downstream of the intercept point) — classify `request_type` from the **`Accept` header + URL extension** instead, the approach AdblockAndroid/DuckDuckGo use; verify the real header map on-device first.

**Proven precedent**: [theoden8/webspace_app](https://github.com/theoden8/webspace_app) embeds the Brave `adblock` crate behind JNI (`AdblockEngineNative.checkUrl` in a native subresource interceptor), derives request type/source exactly this way, and implements **full cosmetics**: document-start CSS hides, a page-side procedural shim (`:has-text()`, `:upward()`, `:remove()`), generic hides fed by a DOM class/id scanner (`hiddenClassIdSelectors`), and `$redirect` stub serving. Lists (EasyList, HaGeZi, ClearURLs) download at runtime — not bundled in the APK. *(This supersedes the "no established Android WebView + adblock-rust project found" note in `adblock-rust-integration.md`.)*

Cosmetic/scriptlet injection has first-class support: `WebViewCompat.addDocumentStartJavaScript` (androidx.webkit 1.5+, feature-gated — check `WebViewFeature.DOCUMENT_START_SCRIPT`, fall back to `onPageStarted` + `evaluateJavascript`) runs before page scripts and inside iframes.

### Honest limits of the WebView hook

- Callback is synchronous on a background thread → engine must answer in microseconds (adblock-rust does; load a preserialized DAT).
- **No POST bodies** ([chromium 40436253](https://issues.chromium.org/issues/40436253)) → `$csp` and body-dependent rules unavailable; POST beacons slip through.
- Blocking = returning an empty `WebResourceResponse` (no clean cancel; fake responses can confuse a few sites).
- `Referer` sometimes absent/trimmed → third-party classification degrades for those requests.
- Service workers need `ServiceWorkerControllerCompat.setServiceWorkerClient`; **WebSocket/WebTransport are not interceptable**.
- Platform limits: no extensions ever, no WebUSB/Web Bluetooth, single profile (new Profile API aside). Rendering/JS perf is parity — same Blink/V8.

**Net power: ~80–85% of the engine; realistically ~90% of uBO's user-visible result** — full network rules for the GET subresources that make up nearly all ads, plus full cosmetics and scriptlets. Better than every hosts-based lite browser today, and better than Cromite on scriptlets.

## 3. The alternatives, for comparison

- **[Cromite](https://github.com/uazo/cromite)** (Bromite successor): maintained Chromium fork with integrated ABP-derived blocking *with* cosmetic rules and a subset of snippets, service-worker and WebSocket filtering, CNAME uncloaking. Full-engine power, 198 MB, someone else runs the fork. Weaker than uBO/Brave on scriptlets.
- **Brave Android**: full adblock-rust, mainstream support, ~175–190 MB.
- **Own Chromium fork: unrealistic.** ≥100 GB checkout, 7–20 h builds, and a rebase treadmill that accelerates to a **2-week Chrome release cycle from Sept 2026**. Vanadium (GrapheneOS) shipped four releases in ten days in Aug 2026 just tracking Chromium — and doesn't even attempt an adblocker. Cromite/Vanadium are each ~one maintainer's full-time job.

## 4. Recommended build

**WebView browser + adblock-rust over JNI (or UniFFI):**
1. Rust `cdylib` per ABI (~2–5 MB total) wrapping `Engine` (`check_network_request`, `url_cosmetic_resources`, `hidden_class_id_selectors`), engine state as a serialized DAT cached on disk.
2. `shouldInterceptRequest` → build `Request` from URL + `Sec-Fetch-Dest` + `Referer` + main-frame flag → empty response on block, stub on `$redirect`.
3. `addDocumentStartJavaScript` → inject hide-CSS + scriptlets + a MutationObserver that reports new class/ids back for generic hides.
4. Runtime filter-list downloader (EasyList, EasyPrivacy, uBO filters, HaGeZi) with periodic refresh — keeps the APK tiny and the lists fresh without app updates.
5. Distribution note: a *browser* with ad blocking is fine on Play (Chrome policy applies to VpnService abuse, not browsers filtering their own WebView) — Via/SmartCookieWeb ship on Play with blockers. Still verify current Play policy at submission time.

## 5. Source notes

Cromite sizes read from its GitHub releases API; WebView API behavior from androidx.webkit release notes, MDN, and Chromium issue tracker; webspace_app details from its repo and spec files. Unverified (egress-blocked or secondary sources): exact Brave/Firefox/Via APK sizes, Play Store install sizes generally. F-Droid pages were unreachable; lite-browser sizes are from project READMEs/marketing and should be re-measured before quoting.
