# Web App Manifest Scope: Problems and Proposed Improvements

**Author:** Lu Huang (luhua@microsoft.com)

## Introduction

The web app manifest `scope` member defines which URLs belong to an installed web application. Today, scope is a single URL prefix — e.g., `"scope": "/app/"` — and every browser behavior tied to "is this URL part of the app?" inherits that single boundary. This was adequate when scope only governed display-mode switching (standalone vs. browser chrome), but as the platform adds more scope-dependent features — **navigation capturing**, **launch routing** — a single prefix is no longer expressive enough for real-world applications.

The [`scope_extensions`](https://wicg.github.io/manifest-incubations/#scope_extensions-member) feature — now shipping in Chromium — addressed the multi-origin dimension of this problem by letting an app extend its scope to additional origins. When `scope_extensions` was designed, fine-grained same-origin scope control (path exclusions, URL pattern filtering) was [explicitly listed as a non-goal](https://github.com/WICG/manifest-incubations/blob/gh-pages/scope_extensions-explainer.md#non-goals) to keep the proposal focused. With `scope_extensions` shipped, the time is right to tackle these harder problems. The `scope_extensions` explainer's ["Future work"](https://github.com/WICG/manifest-incubations/blob/gh-pages/scope_extensions-explainer.md#future-work-under-consideration) section anticipated this, suggesting include/exclude lists and URL patterns as next steps.

This document describes the concrete problems app developers are hitting and proposes areas of specification work to address them. Specific proposals for each area will be added in their own documents alongside this one.

---

## Problem 1: Scope Cannot Exclude Paths

**The pain.** Many web apps set `"scope": "/"` because their application routes span the root of their origin. However, the same origin may also host content that is *not* part of the app experience — marketing pages, documentation, blog posts, or API endpoints. Today there is no way to say "my app is everything on this origin *except* `/blog/` and `/docs/`."

**Why it matters now.** Before navigation capturing, the consequence was minor: out-of-place pages simply appeared in the standalone window with app chrome. With navigation capturing, clicking a link to `/blog/post-1` on any surface (email, chat, another browser tab) *launches the installed app*, pulling the user into an app window for content the developer never intended to be an app experience. Developers are forced to either shrink their scope (breaking deep links to real app pages) or accept that non-app content appears in the app window.

**What developers need.** A way to *exclude* specific paths or path patterns from scope, so that scope reflects the true boundary of the application.

---

## Problem 2: Navigation Capturing Scope Cannot Differ from App Scope

**The pain.** Even when a developer is comfortable with a broad app scope (all pages *should* render in standalone mode), they may not want every in-scope URL to *capture navigations* from outside the app. For example, a productivity suite at `app.example.com` may want its `/dashboard/` and `/editor/` paths to capture navigations but not `/settings/privacy-policy`, which is better opened in a regular browser tab.

**Why it matters.** Navigation capturing is an all-or-nothing proposition today: it applies to every URL inside scope. Developers cannot differentiate between "this URL is part of my app" and "this URL should pull the user into the app window when clicked externally." These are distinct intents, and conflating them forces developers into workarounds (e.g., moving content to a different origin).

**What developers need.** A mechanism to define a **capture scope** that is different from the application scope — controlling which URLs trigger navigation capturing independently of which URLs render in the app window.

---

## Problem 3: Launch Behavior Cannot Vary by URL

**The pain.** The `launch_handler` manifest member lets a developer choose a `client_mode` — `navigate-new`, `navigate-existing`, `focus-existing` — but it applies uniformly to all launches. A mapping app might want `/map/` navigations to reuse an existing window (`navigate-existing`) while `/trip/new` should open a fresh window (`navigate-new`). There is no way to express this today.

**Why it matters.** Different areas of an app have different interaction models. Forcing a single launch behavior leads to UX compromises: either users lose context when an existing window is reused for unrelated content, or they accumulate unnecessary windows when every launch opens a new one.

**What developers need.** The ability to specify **multiple launch handlers**, each associated with a set of URL patterns, so launch behavior can be tailored to the content being opened.

---

## Problem 4: `scope_extensions` Lacks Per-Origin Feature Control

**The pain.** The `scope_extensions` manifest member (and the corresponding `web-app-origin-association` file) allows an app to include additional origins in its scope. However, the association file cannot express which features the extended origin opts into. For example, `help.example.com` may want to appear inside the app window (in-scope for display purposes) but should *not* be subject to navigation capturing — external links to help articles should open normally in the browser.

**Why it matters.** Without annotations, extending scope is all-or-nothing: an associated origin inherits every scope-dependent behavior, including navigation capturing and link handling. This discourages adoption of `scope_extensions` because the side effects are too broad.

**What developers need.** The ability to annotate entries in the `web-app-origin-association` file (or in `scope_extensions` itself) to opt into or out of specific features such as navigation capturing on a per-origin basis.

**Prior consideration.** The `scope_extensions` explainer already [proposed](https://github.com/WICG/manifest-incubations/blob/gh-pages/scope_extensions-explainer.md#future-work-under-consideration) a future `"authorize"` field in the association file — e.g., `"authorize": ["intercept-links"]` — that would let an associated origin explicitly grant or withhold permission for security-sensitive capabilities like navigation capturing.

---

## Proposed Design Direction: Two-Layer Scope Architecture

The problems above point to a unifying design: **separate the concept of "app boundary" from the concept of "feature behavior within that boundary."**

### Layer 1 — OS-to-App Scope (coarse-grained)

This layer answers: *"Does this URL belong to the application?"* It determines install-time registration with the OS, deep-link routing, and display-mode application. Because it must map to OS-level URL filtering (Android intent filters, Windows protocol/URI handling, etc.), it should remain **simple** — path prefixes, origin lists, and at most basic include/exclude patterns.

Design constraints:
- Must be expressible using primitives that operating systems support for URL filtering. Prior work on the (now-obsolete) [`pwa-url-handler` explainer](https://github.com/WICG/pwa-url-handler/blob/main/explainer.md#os-specific-implementation-notes) documented these OS mechanisms: Android [intent filters](https://developers.google.com/web/fundamentals/integration/webapks?hl=ro#android_intent_filters) via WebAPK, Windows ["Apps for Websites"](https://docs.microsoft.com/en-us/windows/uwp/launch-resume/web-to-app-linking), and iOS/macOS [Universal Links](https://developer.apple.com/ios/universal-links/). All are origin- or prefix-based — none support regex or complex pattern matching — which bounds the expressiveness of this layer.
- Must perform well on low-end devices (no regex evaluation at navigation time).
- Keep the existing `scope` member as-is for backward compatibility; introduce new top-level manifest members (e.g., `scope_exclusions` or an enhanced `scope` object) to add or remove paths.

### Layer 2 — In-App Behavior (fine-grained)

This layer answers: *"Now that we are in the app, how should the browser handle this URL?"* This is where richer pattern matching (potentially `URLPattern`) becomes appropriate, because evaluation happens inside the browser after the OS has already routed the URL to the app.

**Prior art.** The [`tab_strip.home_tab.scope_patterns`](https://wicg.github.io/manifest-incubations/#home_tab-member) feature in the Manifest Incubations spec already uses `URLPattern` to define which URLs belong to a tabbed app's home tab vs. regular tabs. This demonstrates that URL-pattern-based behavior differentiation *within* a manifest is already a proven pattern, and the work proposed here can build on that precedent.

Possible features governed by this layer:
- **Capture scope**: which in-scope URLs trigger navigation capturing.
- **Client mode per URL pattern**: different `launch_handler` behaviors for different URL sets.
- **Display mode overrides**: e.g., certain paths rendered `minimal-ui` while the rest use `standalone`.

This layer could be expressed as extensions to existing manifest members (e.g., a `url_patterns` field inside `launch_handler`) or as a new top-level member that maps URL patterns to behavior overrides.

---

## Design Principles

1. **Backward compatibility.** The existing `scope` member must continue to work unchanged. New capabilities are additive.
2. **One syntax, three locations.** The `scope_extensions` explainer's [future work section](https://github.com/WICG/manifest-incubations/blob/gh-pages/scope_extensions-explainer.md#future-work-under-consideration) identifies that fine-grained scoping mechanisms "could be reused in 3 different places: in the association file, in `scope_extensions` in the manifest, at the top level in the manifest." Whatever pattern language is adopted should work consistently across all three. Developers should learn one new concept, not three.
3. **OS-mappable where it counts.** Layer 1 scope definitions must map to OS URL filtering primitives. Overly powerful syntax (full regex, complex URLPattern) at this layer risks poor performance and inconsistent OS support.
4. **Progressive enhancement.** Apps that don't need fine-grained control should not need to add any new manifest members. The defaults should match today's behavior.

---

## Summary

| # | Problem | Proposed Solution Area |
|---|---------|----------------------|
| 1 | Cannot exclude paths from scope | New manifest members for scope include/exclude lists |
| 2 | Navigation capture scope ≡ app scope | Separate "capture scope" concept (Layer 2) |
| 3 | Single launch behavior for all URLs | URL-pattern-keyed launch handlers (Layer 2) |
| 4 | `scope_extensions` lacks feature annotations | Per-origin opt-in/out in association file |

These improvements would bring the manifest's scope model in line with the growing set of features that depend on it, giving developers the control they need without sacrificing simplicity or cross-platform compatibility.

---

*Related: [Chromium issue 380489997](https://issues.chromium.org/issues/380489997) — AppManifest "scope" better fine-grained control by developer*
