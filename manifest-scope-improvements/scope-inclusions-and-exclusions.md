# Web App Manifest: Scope Inclusions and Exclusions

## Authors:

- Lu Huang (Microsoft Edge)

## Participate

- [Issue tracker TBD]
- [Discussion forum TBD]

## Introduction

This proposal adds two scope-refinement members — **`scope_exclusions`** and **`scope_inclusions`** — to the Web App Manifest and to app entries in [web-app-origin-association](https://wicg.github.io/manifest-incubations/#the-web-app-origin-association-file) (WAOA) files. They let developers remove specific paths from scope and add non-contiguous or parameterized paths into scope using [URLPattern](https://urlpattern.spec.whatwg.org/). The two lists are **order-independent**, and each manifest or WAOA entry controls only its own origin. The application's effective scope is the union of those independently evaluated, origin-owned scopes.

## Developer Problem

Progressive Web Apps (PWAs) use the `scope` manifest member to define which URLs should open within the app window versus a browser tab. Currently, `scope` only accepts a **single URL prefix**, which is too restrictive for real-world applications with complex URL structures. Developers have repeatedly reported that this blocks them from adopting scope-dependent features such as [navigation capturing](https://developer.chrome.com/docs/capabilities/pwa-navigation-management). Their needs recur in a few shapes, illustrated below.

> These use cases are grounded in real developer feedback, including public discussion on [WICG/manifest-incubations#105](https://github.com/WICG/manifest-incubations/issues/105) and [w3c/manifest#996](https://github.com/w3c/manifest/issues/996), as well as partner feedback received through other channels.

**1. Exclusions within a broad scope**

Many apps set a broad `scope` (often `scope: "/"`) because the app spans the origin root — but the same origin also hosts URLs that should *not* open in the app window: marketing and help pages (`/about`), authentication and sign-out flows (OAuth callbacks, `/logout`), download or export endpoints (`/download`), and API routes (`/api/*`). A conferencing app, for example, serves meeting rooms from codes on the root path (e.g. `/aaa-bbbb-ccc`) yet must keep pages like `/about` and `/download` out. With only a prefix-based `scope`, it must either:
- Set `scope: "/"` (captures `/about`, `/download`, etc. incorrectly) and handle exclusions client-side
- Not enable navigation capturing at all

```json
{
  "scope": "/",  // Too broad - captures /about, /download, /api/*, etc.
  "start_url": "/"
}
```

**2. Apps with multiple non-contiguous sections**

An app with `/app/`, `/help/`, and `/products/` cannot include all three without over-including by setting `scope: "/"` (which captures sign-out flows, third-party auth callbacks, etc.).

**3. Parameterized URLs**

An app wants every user's dashboard — `/users/123/dashboard`, `/users/456/dashboard` — in scope, but *not* the sibling `/users/123/settings` or `/users/123/billing`. Because `scope` is a **prefix** match, there is no single prefix that captures this: `scope: "/users/"` over-captures settings and billing, and no prefix can "skip" the variable id segment to anchor on `/dashboard`. A pattern with a named segment — `/users/:id/dashboard` — expresses exactly this.

**4. Excluding parameterized URLs**

Exclusions are often themselves parameterized, not literal paths. An app in scope at `/users/` may want *every* user's settings page kept out of the app window — `/users/123/settings`, `/users/456/settings`, and so on for every id. A literal exclusion list cannot enumerate every id; a single pattern (`/users/:id/settings`) can. This is the exclusion-side counterpart to Use Case 3.

**5. Refining scope on an extended origin**

An app that extends its scope to another origin via [`scope_extensions`](https://wicg.github.io/manifest-incubations/#scope_extensions-member) (say, `help.example.com`) may still need a non-contiguous scope on that origin — for instance, keeping `help.example.com/legal` in the browser rather than the app window. The extended origin should own that refinement in its WAOA entry, alongside the `scope` it already grants to the app.

### Goals

- Enable PWAs to **include multiple non-contiguous paths** within their primary and extended scopes
- Allow PWAs to **exclude specific paths** from an otherwise broad primary or extended scope
- Keep every origin's scope configuration **owned by that origin**
- Support **URL patterns** (named parameters, wildcards) for flexible matching
- **Maintain backward compatibility**: apps work unchanged on old browsers that only understand `scope`

### Non-goals

- Manifest-authored **cross-origin patterns** — top-level `scope_inclusions` and `scope_exclusions` cannot affect another origin. An extended origin can refine only its own scope in its WAOA entry after the existing `scope_extensions` consent handshake.
- Regular expression groups (may be restricted for performance/security, per §4 of URL Pattern spec)
- Replacing or deprecating the `scope` member
- **Ordered, position-dependent matching and re-inclusion** (excluding a subtree then re-including a leaf) — deliberately out of scope; see [Alternatives considered](#alternatives-considered)
- Platform-specific scoping APIs (Android App Links, iOS Universal Links) — those remain independent mechanisms

## Proposed Approach

Introduce two new members that refine the set of in-scope URLs described by a `scope` member, leaving the existing `scope` semantics unchanged:

- **`scope_exclusions`** — a list of URL patterns to *remove* from the declaring configuration's scope; the primary, urgently-requested capability.
- **`scope_inclusions`** — a list of URL patterns to *add* to the declaring configuration's scope, for apps whose in-scope URLs are non-contiguous or parameterized and cannot be captured by a single `scope` prefix.

Both are JSON arrays of URL patterns, evaluated as **sets** so that matching is **order-independent** (entry order carries no meaning), drawn from a restricted, OS-mappable subset of the URL Pattern Standard (see [Accepted URLPattern Subset](#accepted-urlpattern-subset)). Bare strings and `URLPatternInit` objects are both accepted. They can appear alongside `scope` at the top level of a manifest or inside that app's entry in an extended origin's WAOA file.

### Placement and Origin Ownership

Scope is configured independently for each origin:

- The Web App Manifest's `scope`, `scope_inclusions`, and `scope_exclusions` configure the app's primary origin.
- A validated WAOA app entry's `scope`, `scope_inclusions`, and `scope_exclusions` configure only the origin hosting that WAOA file.
- The manifest's `scope_extensions` member requests additional origins, but it does not configure their paths. The user agent uses an origin's refinements only after validating that origin's WAOA entry through the existing `scope_extensions` handshake.
- A configuration cannot include or exclude URLs on any other origin.

The same refinement algorithm is therefore reused for the primary origin and every validated extension origin:

```text
primary scope = refine(manifest scope configuration)
extension scope N = refine(validated WAOA scope configuration N)

effective app scope = primary scope ∪ extension scope 1 ∪ extension scope 2 ∪ ...
```

### Matching Algorithm

Within one origin's scope configuration, a URL is *in scope* when it is **included and not excluded**:

1. If the URL's origin differs from the configuration's origin → **out of scope**.
2. **Included** if it is within the `scope` prefix **or** matches any pattern in `scope_inclusions`.
3. If not included → **out of scope**.
4. **Excluded** if it matches any pattern in `scope_exclusions` → **out of scope** (exclude wins).
5. Otherwise → **in scope**.

The lists are **order-independent**: entries may appear in any order and the result is identical. This makes scope configurations easy to author, generate, merge, and reason about.

### Accepted URLPattern Subset

For **portability** (keeping scope mappable to OS deep-link mechanisms such as [Android App Links](https://developer.android.com/training/app-links), [iOS Universal Links](https://developer.apple.com/documentation/xcode/allowing-apps-and-websites-to-link-to-your-content), and [Windows AppUriHandler](https://learn.microsoft.com/en-us/windows/apps/develop/launch/web-to-app-linking)) and **performance** (matching runs on the navigation hot path), `scope_inclusions` and `scope_exclusions` accept a **restricted subset** of URLPattern syntax rather than the full grammar:

**Supported:**
- Literal path segments (`/app/about`)
- Named segments (`:name`, e.g. `/users/:id/dashboard`) — match a single path segment
- Wildcards (`*`, e.g. `/app/*`)
- Prefixes

This is an **allowlist**: any URLPattern feature not listed above — including custom regexp groups (`:id(\d+)`), modifiers (`?`, `+`, `{n,m}`), and query/fragment matching — is unsupported. An entry using an unsupported feature is **ignored** (processing continues with the next entry), and the user agent **SHOULD** emit a console warning. Because ignoring an exclusion fails *open* (the exclusion is lost), authors should validate their patterns; tooling and the console warning surface this at authoring time. The subset is intentionally conservative and can be **expanded additively** in future versions without breaking older user agents.

**Resolution and origin.** Pattern inputs follow the existing base-URL rules of the format that contains them. A top-level Web App Manifest entry is built using the manifest URL as its base, consistent with existing manifest URL-valued members and `display_override.url_patterns`. An entry in a validated WAOA app record is built using that scope extension's origin URL as its base, consistent with existing WAOA `scope` processing. After resolution, a pattern whose origin differs from the scope configuration's origin is ignored:
- `"/logout"` in the Web App Manifest's top-level `scope_exclusions` excludes `/logout` only on the primary origin.
- `"/logout"` in `help.example.com`'s WAOA `scope_exclusions` excludes `/logout` only on `help.example.com`.
- `"https://other.example/logout"` is ignored when it appears in either of those configurations.

### Backward Compatibility

- **Old browser** (knows `scope` but not the new members): ignores both new members and uses only the `scope` value from the Web App Manifest and each validated WAOA entry.
  - Ignoring `scope_inclusions` → the extra sections are simply *not* captured (under-inclusion — safe; the app never captures more than intended).
  - Ignoring `scope_exclusions` → excluded paths *are* captured (the exclusion "fails open"). This is **no worse than the status quo**: without this feature the developer would have shipped the same broad `scope` anyway.
- **New browser**: refines each origin's scope independently, then unions the resulting scopes.

> **Developer rule:** design every `scope` to be acceptable on its own — both the manifest's primary scope and each WAOA entry's extended scope — because any browser that doesn't parse the new members falls back to that `scope` alone. Treat `scope_inclusions` and `scope_exclusions` as progressive enhancement: inclusions degrade to under-capture and exclusions to today's behavior, so the feature is safe to adopt incrementally.

### Solving Use Case 1: Exclusions at Root Level

**Problem**: Meeting codes are at root (`/aaa-bbbb-ccc`) but marketing pages like `/about` must not be captured.

```json
{
  "name": "Conferencing App",
  "scope": "/",
  "scope_exclusions": ["/about", "/download"]
}
```

- Root meeting codes stay in scope; `/about` and `/download` are excluded.

### Solving Use Case 2: Multiple Non-Contiguous Sections

**Problem**: An app's real pages live under `/app/`, `/help/`, and `/products/`, but the origin also hosts sign-out flows and auth callbacks that must not be captured.

```json
{
  "scope": "/app/",
  "scope_inclusions": ["/help/*", "/products/*"]
}
```

- `scope` already covers `/app/*`; `scope_inclusions` adds `/help/` and `/products/`. All three sections are in scope, and nothing else on the origin is captured.

### Solving Use Case 3: Parameterized URLs

**Problem**: Every user's dashboard should be in scope, but their other per-user pages (`/settings`, `/billing`) should not. No single `scope` prefix can express "any user id, but only the `/dashboard` leaf."

```json
{
  "scope": "/app/",
  "scope_inclusions": ["/users/:id/dashboard"]
}
```

- `/users/:id/dashboard` matches `/users/123/dashboard` and `/users/456/dashboard` (`:id` matches one path segment), but not `/users/123/settings` or `/users/123/billing`.
- This is something a prefix-based `scope` fundamentally cannot express — the value of URL-pattern-based inclusion.

> **Not supported: re-inclusion.** Excluding a broad subtree and then re-including a specific leaf under it (e.g. exclude `/docs/internal/*` but re-include `/docs/internal/preview`) is intentionally **not** expressible, because it requires position-dependent precedence. No collected use case needs it, and OS support is inconsistent: Apple Universal Links and Android 15+ Dynamic App Links can represent ordered re-inclusion, while Windows and older Android cannot. See [Alternatives considered](#alternatives-considered) for the full rationale.

### Solving Use Case 4: Excluding Parameterized URLs

**Problem**: Every user's settings page should be kept out of scope, across all user ids.

```json
{
  "scope": "/users/",
  "scope_exclusions": ["/users/:id/settings"]
}
```

- `/users/:id/settings` matches `/users/123/settings`, `/users/456/settings`, and so on — removing the whole family with one entry.

### Solving Use Case 5: Origin-Owned Extended Scope

**Problem**: The app extends into `help.example.com` via `scope_extensions`, but `help.example.com/legal` should open in the browser, not the app window.

Primary manifest at `https://app.example.com/manifest.webmanifest`:

```json
{
  "id": "/app",
  "scope": "/",
  "scope_extensions": [
    { "type": "origin", "origin": "https://help.example.com" }
  ]
}
```

WAOA file at `https://help.example.com/.well-known/web-app-origin-association`:

```json
{
  "https://app.example.com/app": {
    "scope": "/",
    "scope_exclusions": ["/legal"]
  }
}
```

- `help.example.com` grants the app a broad `scope` but removes `/legal` from that grant.
- The primary manifest cannot include or exclude paths on `help.example.com`; the extended origin owns those rules.
- The user agent applies the WAOA refinements only after validating the existing `scope_extensions` handshake.
- `help.example.com` can update its own scope without requiring a change to the primary manifest.

## Alternatives considered

### Alternative 1: Single ordered list with include/exclude actions

A single `scope_patterns` member holding an **ordered list** of entries, each `{ "pattern": ..., "action": "include" | "exclude" }`, evaluated **first-match-wins** (an unknown action falls through to the next entry). Bare patterns default to `include`.

```json
{
  "scope": "/docs/",
  "scope_patterns": [
    { "pattern": "/docs/internal/preview", "action": "include" },
    { "pattern": "/docs/internal/*", "action": "exclude" },
    "/docs/*"
  ]
}
```

#### Pros
- **Re-inclusion**: position encodes precedence, so a specific include can override a broad exclude (the `/docs/internal/preview` case above).
- **Single extensible list**: future action types slot into one place.

#### Cons
- **Order-sensitivity is an authoring footgun.** Rearranging entries silently changes meaning; a broad include placed before a specific exclude makes the exclude a **dead, shadowed entry** with no error. The two-list model is order-independent and has no such trap.
- **Re-inclusion is not portable across OS deep-link filters.** Apple Universal Links and Android 15+ Dynamic App Links use ordered rules that can represent it, but Windows and older Android cannot. Translating an ordered list for those platforms can **silently drop re-inclusion**, so an app relying on it would behave **differently via OS deep-link than in-browser** — a correctness divergence, not merely reduced precision.
- **No grounded use case needs re-inclusion.** Every collected developer report is plain include-minus-exclude.

A **last-match-wins** variant (mirroring `.gitignore` negation) was also considered; it has the same order-sensitivity and OS-mapping problems.

#### Reason for rejection

The single ordered list's only advantage over the proposed two-list model is re-inclusion. No collected use case needs re-inclusion, it lacks uniform OS support, and the ordered model makes authoring more error-prone. The proposed two-list model therefore deliberately does not support it.

### Alternative 2: Prefix-only patterns (no URL Pattern dependency)

Restrict patterns to simple path/URL prefixes, avoiding the URL Pattern Standard entirely:

```json
{
  "scope_inclusions": ["/app/"],
  "scope_exclusions": ["/admin/"]
}
```

#### Pros
- **Best platform compatibility**: pure prefixes map directly to OS deep-link filters (Android `pathPrefix`, iOS Universal Links) with no widening.
- **Simplest matching**: prefix lookup is cheaper than matching named segments or wildcards.

#### Cons
- **Limited expressiveness**: Cannot match a variable segment with a fixed suffix, like `/users/:id/dashboard`
- **Reinvents matching machinery**: URLPattern is already standardized and implemented in browsers; a bespoke prefix syntax would need its own definition and implementation

#### Reason for rejection

Prefix-only can't match the parameterized URLs developers need (Use Case 3), and URLPattern is already adopted by the manifest and shipping in browsers — so the added expressiveness is worth the modest cost.

**Note**: Developers should still prefer simple prefixes (e.g. `/app/*`) when they suffice, for better performance and platform compatibility.

### Alternative 3: Inclusions replace `scope` (rather than union with it)

This is a *semantics* choice, orthogonal to the list-shape alternatives above: how should `scope_inclusions` combine with the existing `scope` prefix? The proposal **unions** them — a URL is included if it is within `scope` **or** matches an inclusion. The alternative is to let inclusions **replace** `scope`: when `scope_inclusions` is present and parseable, a URL is included only if it matches an inclusion, and `scope` is consulted solely as a fallback for browsers that don't understand the new member.

```
// Proposed (union):
included = within(scope) OR matchesAny(scope_inclusions)

// Alternative (replace):
included = scope_inclusions present ? matchesAny(scope_inclusions) : within(scope)
```

#### Pros
- **Strictly more expressive**: inclusions can define scope sets that no prefix-superset can express (e.g. exactly `/products/:id` with nothing broader), because the `scope` prefix no longer forces its region into the result.
- **Clean separation**: `scope` reads as a pure legacy fallback and `scope_inclusions` as the authoritative definition.

#### Cons
- **Unsafe degradation** (decisive): a browser that ignores `scope_inclusions` applies `scope` alone, so it captures the `scope`-minus-`inclusions` region that the developer intended to leave *out*. This is **over-capture** — the dangerous direction — whereas union only ever **under-captures** on old browsers (safe). Staying safe would require the developer to keep `scope ⊆ inclusions`, and `scope` becomes dormant in new browsers but active in old, so its fallback behavior diverges from its role — directly at odds with the "design `scope` to stand alone" guidance.
- **Mode-switch semantics**: `scope`'s effect depends on whether `scope_inclusions` is present, which is harder to specify and teach than one uniform rule.
- **Changes `scope`'s established meaning** for existing tooling and readers.

#### Reason for rejection

Union keeps a single uniform per-origin rule, preserves `scope`'s existing meaning, and — most importantly — degrades safely: a browser that drops the new members under-captures via a still-additive `scope`, never over-captures. Replace's extra expressiveness only enables semantically-odd scopes (those excluding the `start_url` region, which developers rarely want) at the cost of the over-capture failure mode.

## Dependencies on non-stable features

This proposal depends on the **URL Pattern Standard** ([urlpattern.spec.whatwg.org](https://urlpattern.spec.whatwg.org/)), specifically:

- §4.2 "Integrating with JSON data formats" — the "build a URL pattern from an Infra value" algorithm
- The concept of `has regexp groups` for optionally restricting patterns

URL Pattern is a WHATWG standard and is being implemented across major browser engines; see [MDN's browser compatibility table](https://developer.mozilla.org/en-US/docs/Web/API/URLPattern#browser_compatibility) for current support.

## Privacy and Security Considerations

### Privacy

No new privacy surface: scope matching is a local, in-browser decision that exposes no new data to the app or to other origins.

### Security

**Origin-local enforcement**: a scope configuration can match only its own origin; entries whose `protocol`, `hostname`, or `port` differ are dropped. Top-level manifest rules apply only to the primary origin, and WAOA rules apply only to the origin hosting that file. Expanding the app into another origin still requires both the manifest's `scope_extensions` request and that origin's validated WAOA grant, so an app cannot claim arbitrary origins.

**Exclusions are not a security boundary**: an exclusion that fails to parse, is unsupported, or is ignored by an old browser results in *over-capture* (the URL stays in scope). Apps MUST NOT rely on exclusions for security isolation (e.g., hiding a sign-out flow); server-side authorization is the real boundary.

## Stakeholder Feedback / Opposition

- Chromium/Edge: Proposing (Microsoft Edge)
- Mozilla: TBD
- WebKit: TBD
- Web developers: Demand documented in [WICG/manifest-incubations#105](https://github.com/WICG/manifest-incubations/issues/105) and [w3c/manifest#996](https://github.com/w3c/manifest/issues/996)

## Open Questions

### How are WAOA scope changes activated and revoked?

An extension origin can update its WAOA entry independently of the primary manifest. The browser's precise scope and the OS deep-link registration may therefore update at different times. To avoid losing valid deep links, an expansion should not become active in browser scope until the OS registration covers it. A contraction or revocation should take effect in browser scope first; a temporarily stale OS registration then only over-routes URLs that the browser can reject.

The processing model still needs to define WAOA refresh and cache lifetime, replacement of one validated generation with another, behavior on fetch failure versus explicit revocation, and rollback when native registration cannot be updated.

### What if an exclusion matches `start_url`?

The Web App Manifest algorithm for [processing the `scope` member](https://www.w3.org/TR/appmanifest/#scope-member) sets the default processed scope to the containing path of `start_url`, then adopts the declared `scope` only if it contains `start_url`. Refinement must preserve or deliberately revise that invariant. The specification needs to choose whether a conflicting exclusion is ignored, causes the refinement members to be rejected, or triggers another explicit fallback. Authors should not rely on excluding `start_url`.

### Ship `scope_exclusions` alone first, or both members together?

Exclusions are the urgent, dominant real-world request; inclusions serve the real-but-less-common non-contiguous / parameterized cases. Options: (a) ship `scope_exclusions` only in v1 and add `scope_inclusions` later; (b) ship both together for a symmetric, complete model. Either order is safe — shipping `scope_inclusions` in a later release is a purely additive change, not a breaking one.

## Acknowledgements

Many thanks for valuable feedback and advice from:

- Daniel Murphy
- Jacob Mills
- Vatan Aggarwal
- Howard Wolosky
- Alan Cutter
