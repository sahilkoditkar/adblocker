# Two Concerns Verified: WebView Interception & Android Developer Verification

Research date: 2026-09-02. Concern 1: "you cannot intercept or change Android WebView requests, so porting adblock-rust wouldn't work." Concern 2: Google's developer-verification requirement ("sideloaded apps stop working after ~120 days?") and what it means for unregistered maintainers and personal apps.

---

## Concern 1 — WebView interception: claim refuted, with one true kernel and one correction

Verified against Chromium source (`android_webview/browser/network_service/`), AOSP javadoc, Chromium's own WebView test suite, and shipping apps.

### What is definitively TRUE (works)

- `shouldInterceptRequest` fires for **every http(s)/data/file subresource and navigation, including HTTPS** — it's a callback *above* the network stack inside the app's own WebView, before the fetch is issued. No MITM, no certificate needed. Fires for main frame, iframes, XHR/fetch, images, workers; also for HTTP-cache-servable requests.
- **Blocking works**: return a `WebResourceResponse` with an **empty** stream (a `null` stream triggers `onReceivedError` — use empty, not null).
- **Replacing works**: serve arbitrary bytes/MIME/status — uBO `$redirect=` stubs work exactly as designed.
- **`Referer` is guaranteed present** — Chromium explicitly sets it immediately before the intercept callback (own test asserts it), so initiator/third-party classification is sound.
- **Decisive shipping precedent**: [Edsuns/AdblockAndroid](https://github.com/Edsuns/AdblockAndroid) runs **Brave's ad-block engine via JNI** blocking through `shouldInterceptRequest`, with cosmetic + scriptlet injection — literally "the Brave engine in a WebView" already working. Plus webspace_app, Via, SmartCookieWeb etc. at the hosts-list tier.

### The kernel of truth (what you CANNOT do)

- **You cannot modify a request and let it continue.** The API is answer-or-pass-through: return a response or return null (request proceeds *unmodified*). No header mutation, no URL rewrite in place. So `$removeparam`/`$csp`/`$removeheader` are out (except via a full re-fetch with your own HTTP client — not viable generally: cookies aren't shared, you lose Chromium's cache/H2/H3).
- **POST/PUT bodies are invisible** (verified in the source: the request struct has no body field).
- **WebSockets aren't interceptable**; service workers need `ServiceWorkerControllerCompat`'s separate client.
- **Redirects: only hop 1 is matched** — a tracker that 302s to another tracker is checked only on the first URL. Real, small adblock gap.
- Likely myth-origins of the claim: confusing `shouldOverrideUrlLoading` (navigation-only, skipped for POST navigations) with `shouldInterceptRequest`, or the pre-API-21 URL-only overload (obsolete since 2014).

### ⚠️ Correction to `android-lite-browser.md`

**`Sec-Fetch-Dest` is almost certainly NOT visible** in intercepted headers (~85% confidence from source reading): Chromium sets fetch-metadata headers *downstream* of the interception point, by design (anti-spoofing). Resource-type classification should instead use the **`Accept` header + URL extension** heuristic (what DuckDuckGo's and AdblockAndroid's classifiers do): `image/*`→image, `text/css`→stylesheet, `*javascript*`→script, `text/html` on subresource→subdocument, `font/`→font. `Referer` + `isForMainFrame` remain solid. First implementation task: log the real header map on-device once to confirm.

### Verdict

"Wouldn't work" is **false**. Adjusted power estimate: **~78–83%** of the full engine (was 80–85%) — docked slightly for the redirect-hop gap and the Sec-Fetch correction, nothing fatal. Block + stub-redirect + cosmetics + scriptlets all stand.

---

## Concern 2 — Android Developer Verification (status: 2 Sept 2026)

### The "120 days" claim: no basis found

Nothing in Google's docs, blogs, or coverage contains a 120-day element. **Already-installed apps are never disabled** — enforcement blocks *new installs and updates*, not execution of what's installed. (Likely confusions: Apple's 7-day free-provisioning expiry, Play's 180-day inactive-account closure, or Play's 12-testers/14-days rule.)

### The actual rules & timeline

- Apps must be registered (package + signing key) to an identity-verified developer to install normally on **certified** devices (= GMS phones). Full account: government ID + $25 one-time (orgs: D-U-N-S).
- **30 Sep 2026**: enforcement starts in Brazil/Indonesia/Singapore/Thailand — **but only for participating app stores** (Play, Galaxy Store, Xiaomi GetApps, etc.). Google's own FAQ: direct sideloading (browser/GitHub/F-Droid) is **not covered in this phase**. Press saying "sideloading blocked Sept 30" is wrong.
- **2027+**: global expansion to all install sources on certified devices; exact mechanics/dates unpublished. **India is in the 2027+ wave, no date set** (and regulator pressure there is live).

### Escape hatches (all officially confirmed)

1. **Limited Distribution tier (live Aug 2026)**: **free, no government ID**, up to **20 authorized devices** per account, installs via QR/link handshake. Purpose-built for personal/family/friends apps — this answers the side-project question: yes, you can build and share without the fee or ID.
2. **ADB installs: permanently exempt.** `adb install` bypasses the gate entirely.
3. **Advanced flow** (live): one-time power-user unlock (Developer options → reboot → ~24h wait → biometric confirm), then unverified APKs install with an "install anyway" warning, indefinitely. Syncs across the user's devices.
4. **Non-certified ROMs** (LineageOS, GrapheneOS, /e/OS): no gate at all.

### What it means for projects like AABrowser

Nothing changes today anywhere (GitHub isn't a participating store). From the 2027 wave, users on stock phones need the one-time advanced flow or ADB; existing installs keep working; updates are what break for unverified apps. F-Droid is fighting the scheme (open letter Feb 2026, 60+ orgs) because its re-signing model makes *it* the "developer"; outcome unsettled. For our own distribution: Limited Distribution tier for the friends-and-family circle, Full ($25+ID) only if going public at scale — or public GitHub APKs relying on advanced-flow/ADB literacy of the audience, which for an ad-blocker-browser audience is plausible.

### Flagged uncertainties

2027 mechanics and dates; whether limited-distribution device slots can be reassigned; possible regulator-forced carve-outs (EU/India); the Sec-Fetch-Dest absence is source-derived, not device-verified.
