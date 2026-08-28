# Where Can adblock-rust Actually Be Integrated?

Research date: 2026-08-28. Follow-up to `RESEARCH.md` §2. Question: does Brave's engine have to live in a browser build, can it ride on top of an already-installed Chrome, and where does it work outside browsers entirely?

---

## 1. What the engine is (and therefore where it can go)

`adblock` (crates.io, v0.13.3 as of Aug 2026, MPL-2.0) is **not a network stack and not tied to a browser** — it's a pure classifier + selector provider. You feed it, it answers; the *host* does the intercepting and injecting:

```rust
// network decision — host must supply full request context
Request::new(url, source_url /* initiator */, request_type, method)
engine.check_network_request(&request) -> BlockerResult { should_block, redirect, rewritten_url, .. }

// cosmetics — host must inject the returned CSS/JS into pages
engine.url_cosmetic_resources(url) -> { hide_selectors, injected_script (scriptlets), .. }
engine.hidden_class_id_selectors(classes, ids, ..)  // generic rules, fed by a page-side MutationObserver

engine.serialize() / deserialize()  // precompiled binary "DAT" for fast startup
```

So an embedder needs, in decreasing order of importance:
1. **A per-request hook** (before the request goes out) carrying url + initiator + resource type, with the power to **cancel/redirect**. Without this → cosmetics only.
2. **CSS/JS injection into pages** at document start. Without this → network blocking only, broken layouts remain.
3. A DOM feedback channel (page reports its class/ids) for cheap generic cosmetic filtering.

Notable: if the host can't supply the initiator (`source_url`), third-party classification degrades (unparseable source = treated third-party) — this is exactly what hurts proxy and WebView hosts.

**Bindings**: Rust crate (canonical, active); npm **`adblock-rs`** (version-locked to the crate, but it's a **Neon native module, not WASM** — installing it compiles Rust, so consumers need a toolchain); WASM works via `wasm-bindgen` but there is **no official published WASM package** (Brave's demo: [adblock-rust-dashboard](https://github.com/brave-experiments/adblock-rust-dashboard)); PyPI `adblock` (python-adblock) is **abandoned** (last release 2022). A `content-blocking` cargo feature converts lists to Apple Safari JSON — lossily (see §4c).

---

## 2. Can we bolt it onto an already-installed Chrome? (the honest answer)

**Not at full power — this is a hard, structural ceiling, not an engineering gap.**

### 2a. As an MV3 extension (WASM build) — partial only

Chrome MV3 has **no blocking `webRequest`** (observational only, except enterprise force-installed extensions). So `check_network_request` — the heart of the engine — **can never gate a real request** in an extension on stock Chrome. The proof-of-shape is [gasanache/brave-shields-extension](https://github.com/gasanache/brave-shields-extension): it does *not* use the engine for network blocking at all — it ships pre-compiled static `declarativeNetRequest` rulesets (a capped, lossy subset of the lists) and uses the WASM engine **only for cosmetic filtering** plus hand-written YouTube/Twitch hacks. What translates to DNR loses `$csp`, most regex, `$removeparam` combos, and the engine's exception/`$important` precedence semantics; budgets are ~30k static guaranteed (+ shared pool), only ~5k "unsafe" (block/redirect) dynamic rules, 1k regex.

So on stock Chrome an extension gets you: **full cosmetics + scriptlets, DNR-subset network blocking.** That's roughly the uBO-Lite ceiling with extra steps.

**Exceptions that do get full power without a custom build:**
- **Enterprise-policy force-installed** extensions keep blocking `webRequest` (`ExtensionSettings` policy) — viable for a managed-devices product, not consumers.
- **Firefox**: MV2 *and* MV3 keep blocking `webRequest` → a Firefox extension embedding the WASM engine gets the **entire** engine (network + cosmetics + scriptlets). Works on desktop and Android.
- **Stock Firefox 149+ literally ships adblock-rust already** — disabled, no UI, driven by two `about:config` prefs (`privacy.trackingprotection.content.protection.enabled` + `...test_list_urls`); [adblock-rust-manager](https://github.com/electricant/adblock-rust-manager) is an extension that flips them. Experimental/unofficial, but real.

### 2b. Local MITM proxy — the only way to get the full engine into unmodified Chrome

Point stock Chrome (or the whole OS) at a local filtering proxy that terminates TLS with a locally-trusted CA. The reference implementation exists: **[Privaxy](https://github.com/Barre/privaxy)** (Rust, AGPL-3.0, 2.5k★) — hyper + rustls MITM, per-host leaf certs from a generated CA, `adblock` crate for network decisions, and Cloudflare's `lol_html` streaming rewriter to inject cosmetic CSS/scriptlets into HTML responses. ~50 MB RAM with ~320k filters. Deployment: install its CA into the OS trust store, set system proxy (Chrome follows it) or `--proxy-server=127.0.0.1:8100`.

Caveats: Privaxy is **dormant** (no pushes since mid-2024, pinned to adblock 0.6.0, no maintained fork found — reviving/forking it is a genuine project opportunity). Class-wide limits: certificate-pinned apps break or need exclusions; **HTTP/3/QUIC isn't proxied** (must block UDP/443 to force TCP fallback or traffic silently bypasses the filter); initiator is approximated from `Referer`/`Origin` so third-party matching degrades; TLS re-encryption + HTML rewriting costs latency.

---

## 3. Outside the browser — where the engine genuinely fits

| Host | How | Power |
|---|---|---|
| **Electron apps** | `session.webRequest.onBeforeRequest` (Electron's own API — MV3 doesn't apply, blocking works) + `adblock-rs` native module + preload script for CSS/scriptlets. Turnkey comparison: `@ghostery/adblocker-electron` (`blocker.enableBlockingInSession(...)`) | **Full** |
| **Puppeteer / Playwright** (scrapers, test rigs, ad-measurement crawlers) | Request interception → engine check; `addStyleTag`/`addInitScript` for cosmetics. Ghostery ships `PuppeteerBlocker`/`PlaywrightBlocker`; real crates.io dependents: `spider_network_blocker`, `chromey`, Brave's `pagegraph` | **Full** |
| **Android WebView app** (own browser/reader app) | `shouldInterceptRequest` + Rust via JNI/UniFFI. Better than it first looks: `Sec-Fetch-Dest`/`Referer` headers reconstruct resource type + initiator, and `addDocumentStartJavaScript` covers cosmetics/scriptlets. Working precedent: [webspace_app](https://github.com/theoden8/webspace_app). See `android-lite-browser.md` for the full analysis | **~80–85%** |
| **iOS / WKWebView** | No interception API at all. The engine's `content-blocking` feature converts lists → Apple content-blocker JSON (Brave's "Slim List" pipeline). Conversion **drops** scriptlets, `$redirect`, `$csp`, `$removeparam`, procedural cosmetics, most regex | Weak |
| **LAN/router-wide proxy** | Privaxy bound to 0.0.0.0 — works in principle, but every device needs the CA installed; Android 7+ apps ignore user CAs, TVs/consoles can't install one | Full-but-impractical |
| **DNS server** | **Wrong tool.** No DNS API — you'd fake `http://domain/` and only `\|\|domain^` rules match; path/type/party rules (most of EasyList) and all cosmetics are lost. Use AdGuard DnsLibs or a plain domain-set matcher; real "adblock DNS" projects compile lists to domain sets instead of embedding the engine | Not suitable |
| **Non-Rust hosts via IPC** | [adblock-rust-server](https://github.com/dudik/adblock-rust-server): tiny daemon on a Unix socket (`n <url> <source> <type>` → block?, `c <url> ...` → CSS) — a clean pattern for embedding from any language | Full (network+cosmetic data) |

---

## 4. Verdict table

| Integration point | Network | Cosmetic | Scriptlets | Verdict |
|---|---|---|---|---|
| Custom browser build (Brave, Ladybird, Firefox 149) | ✅ | ✅ | ✅ | Full — but you're shipping a browser |
| **Stock Chrome, MV3 extension** | ⚠️ DNR subset | ✅ | ✅ | **Partial — hard ceiling, no per-request hook** |
| Stock Chrome, enterprise force-install | ✅ | ✅ | ✅ | Full (managed devices only) |
| **Firefox extension (MV2/MV3, WASM engine)** | ✅ | ✅ | ✅ | **Full — desktop + Android** |
| **Local MITM proxy (Privaxy model)** | ✅ | ✅ | ✅ | **Full on stock browsers; CA/pinning/QUIC costs; flagship impl dormant** |
| Electron app | ✅ | ✅ | ✅ | Full |
| Puppeteer/Playwright | ✅ | ✅ | ✅ | Full |
| Android WebView | ⚠️ URL-only | ✅ | ✅ | Partial |
| iOS/WKWebView | ⚠️ converted JSON | ⚠️ hide-only | ❌ | Weak |
| DNS | ⚠️ domain rules only | ❌ | ❌ | Not suitable |

**Bottom line**: the engine is host-agnostic — anything with a request hook and page injection can use it (Electron, automation, proxies, Firefox extensions, WebViews at reduced power). What it *cannot* do is reach full power inside an unmodified Chrome via an extension; for that, the only path is the local-MITM-proxy pattern, and the best existing implementation (Privaxy) is unmaintained — which is either a warning or an opportunity.

## 5. Source notes

Engine API, cargo features, and content-blocking conversion losses read directly from `brave/adblock-rust` source; brave-shields-extension architecture from its repo; Privaxy architecture from its `Cargo.toml`/README. developer.chrome.com and electronjs.org were unreachable from the research environment — DNR rule budgets and Electron cancel semantics come from MDN/secondary sources and long-standing API contracts; re-verify exact numbers before relying on them. Firefox 149 pref names via the adblock-rust-manager project, not Mozilla docs.
