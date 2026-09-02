# AABrowser as a Base for Ad Blocking — Assessment

Research date: 2026-09-02. Question: could [kododake/AABrowser](https://github.com/kododake/AABrowser) be the shell we add adblock-rust to, per the plan in `android-lite-browser.md`?

## What it is

A **plain Android WebView browser** (Kotlin, Views, `android.webkit.WebView` + androidx.webkit 1.16) that markets itself for **Android Auto head units** — not a projection app, just an Activity declaring the car-launcher categories so AA lists it (users enable AA "unknown sources"). Active and popular (464★, created Oct 2025, last push Jul 2026, v2.2), GPL-3.0, ~7,400 LOC across 31 well-organized files, no tests. Aggressive platform floor: **minSdk 35 (Android 15+), compileSdk 37** (preview toolchain). No native code/NDK at all. Realistic APK ~5–8 MB. Quirks you'd strip: a self-hosted Umami analytics pinger, `usesCleartextTraffic=true` + `MIXED_CONTENT_ALWAYS_ALLOW`.

## Architecture fit (surprisingly good shape)

- **No ad blocking of any kind today** — README says so ("contributions welcome!"); open issues [#120](https://github.com/kododake/AABrowser/issues/120) and [#87](https://github.com/kododake/AABrowser/issues/87) request it.
- All WebView setup funnels through **one function**: `web/ConfiguredWebView.kt:47` `configureWebView()`, whose anonymous `WebViewClient` (`:99`) has **no `shouldInterceptRequest` override** — our interceptor drops in right there. Single call site per tab (`tabs/TabManager.kt:177`), so multi-tab is already centralized.
- androidx.webkit feature-gating idiom is established (`WebViewFeature.*` checks, `addWebMessageListener` at `TabManager.kt:199-205`), but `addDocumentStartJavaScript` isn't used yet; existing JS injection happens too late (`onPageFinished`, `ConfiguredWebView.kt:140`) — would move to document-start for cosmetics. A working page↔native bridge exists to copy for the generic-cosmetics class/id feedback loop. No `ServiceWorkerControllerCompat` (SW traffic currently unfilterable).
- Licensing fine: GPLv3 shell + MPL-2.0 engine are compatible.

## What adding adblock-rust would take

Entirely additive, ~1–2 weeks, no hard blockers: (a) Rust `cdylib` + JNI bridge (greenfield here — no NDK setup exists; adds ~2–5 MB of `.so`, roughly doubling the APK); (b) `shouldInterceptRequest` building `Request` from URL + `Sec-Fetch-Dest` + `Referer` + `isForMainFrame`; (c) document-start CSS/scriptlet injection; (d) service-worker client; (e) filter-list downloader + DAT cache (okhttp/serialization already in deps).

## Verdict

**Legitimate project, wrong shell for our general-purpose lite browser.** Everything hard about the ad blocker is portable and AABrowser contributes none of it — what it contributes is 7k LOC of *car-head-unit* chrome (800×480 landscape UI, giant touch targets), an Android 15+ floor that excludes most phones, and defaults we'd strip. Ranked:

1. **Fresh minimal WebView browser**, using webspace_app as the reference for the hard parts (header-based request reconstruction, procedural-cosmetics shim, redirect stubs) — still the recommended path from `android-lite-browser.md`.
2. **AABrowser as a reading reference** — `ConfiguredWebView.kt` and `TabManager.kt` are a clean model of tab lifecycle and androidx.webkit gating. And there's a genuine opportunity in its niche: it owns the Android Auto browser market and ad blocking is its most-requested missing feature — **contributing our blocker upstream to issue #120** would ship the engine work to real users without owning a browser.
3. Don't fork it into a phone browser.
