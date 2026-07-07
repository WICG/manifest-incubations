# Web App Manifest: Scope Inclusions and Exclusions

## Authors:

- Lu Huang (Microsoft Edge)

## Participate

- [Issue tracker TBD]
- [Discussion forum TBD]

## Introduction

This proposal adds two members — **`scope_exclusions`** and **`scope_inclusions`** — to the Web App Manifest, enabling fine-grained control over a Progressive Web App's navigation scope using [URLPattern](https://urlpattern.spec.whatwg.org/). They let developers remove specific paths from scope and add non-contiguous or parameterized paths into scope, as two **order-independent lists**, addressing the limitations of the current single-prefix `scope` member while maintaining backward compatibility with existing browsers.

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

**5. Excluding a path on an extended origin**

An app that extends its scope to another origin via [`scope_extensions`](https://wicg.github.io/manifest-incubations/#scope_extensions-member) (say, `help.example.com`) may still want to exclude a path on that origin — for instance, keeping `help.example.com/legal` in the browser rather than the app window. A single exclusion list can apply across both the app's own origin and its extension origins.

### Goals

- Enable PWAs to **include multiple non-contiguous paths** within their scope
- Allow PWAs to **exclude specific paths** from an otherwise broad scope, including from extended origins
- Support **URL patterns** (named parameters, wildcards) for flexible matching
- **Maintain backward compatibility**: apps work unchanged on old browsers that only understand `scope`

### Non-goals

- Cross-origin **inclusions** — adding another origin's URLs *into* scope (reserved for future work). This does **not** restrict `scope_exclusions`, which may exclude cross-origin paths.
- Regular expression groups (may be restricted for performance/security, per §4 of URL Pattern spec)
- Replacing or deprecating the `scope` member
- **Ordered, position-dependent matching and re-inclusion** (excluding a subtree then re-including a leaf) — deliberately out of scope; see [Alternatives considered](#alternatives-considered)
- Platform-specific scoping APIs (Android App Links, iOS Universal Links) — those remain independent mechanisms

## Proposed Approach

Introduce two new manifest members that refine the set of in-scope URLs, leaving the existing `scope` member unchanged:

- **`scope_exclusions`** — a list of URL patterns to *remove* from scope; the primary, urgently-requested capability. Exclusions apply **globally**: they may remove paths from the app's own origin and from any origins the app adds via `scope_extensions`. This needs no other origin's consent, because removing a URL from scope only ever shrinks the app's reach.
- **`scope_inclusions`** — a list of URL patterns to *add* to scope, for apps whose in-scope URLs are non-contiguous or parameterized and cannot be captured by a single `scope` prefix. These patterns may only match **`scope`'s origin** (the single origin the app occupies today): adding another origin's URLs to scope requires that origin's consent, which remains the responsibility of `scope_extensions` and its association-file handshake.

Both are JSON arrays of URL patterns, evaluated as **sets** so that matching is **order-independent** (entry order carries no meaning), drawn from a restricted, OS-mappable subset of the URL Pattern Standard (see [Accepted URLPattern Subset](#accepted-urlpattern-subset)). Bare strings and `URLPatternInit` objects are both accepted.

### Matching Algorithm

A URL is *in scope* when it is **included and not excluded**:

1. **Included** if it is within the `scope` prefix **or** matches any pattern in `scope_inclusions`.
2. If not included → **out of scope**.
3. **Excluded** if it matches any pattern in `scope_exclusions` → **out of scope** (exclude wins).
4. Otherwise → **in scope**.

The lists are **order-independent**: entries may appear in any order and the result is identical. This makes manifests easy to author, generate, merge, and reason about, and mirrors how OS deep-link filters already behave (iOS Universal Links uses include-paths plus `!`-exclude-paths with exclude-wins; Android/Windows are include-only sets). This set-based shape maps cleanly onto those platform APIs; an ordered list would not.

### Accepted URLPattern Subset

For **portability** (keeping scope mappable to OS deep-link mechanisms such as [Android App Links](https://developer.android.com/training/app-links), [iOS Universal Links](https://developer.apple.com/documentation/xcode/allowing-apps-and-websites-to-link-to-your-content), and [Windows AppUriHandler](https://learn.microsoft.com/en-us/windows/apps/develop/launch/web-to-app-linking)) and **performance** (matching runs on the navigation hot path), `scope_inclusions` and `scope_exclusions` accept a **restricted subset** of URLPattern syntax rather than the full grammar:

**Supported:**
- Literal path segments (`/app/about`)
- Named segments (`:name`, e.g. `/users/:id/dashboard`) — match a single path segment
- Wildcards (`*`, e.g. `/app/*`)
- Prefixes

This is an **allowlist**: any URLPattern feature not listed above — including custom regexp groups (`:id(\d+)`), modifiers (`?`, `+`, `{n,m}`), and query/fragment matching — is unsupported. An entry using an unsupported feature is **ignored** (processing continues with the next entry), and the user agent **SHOULD** emit a console warning. Because ignoring an exclusion fails *open* (the exclusion is lost), authors should validate their patterns; tooling and the console warning surface this at authoring time. The subset is intentionally conservative and can be **expanded additively** in future versions without breaking older user agents.

**Origin.** A `scope_inclusions` pattern may only reference `scope`'s origin. A `scope_exclusions` pattern may additionally name an origin the app has extended into via `scope_extensions`. A **bare path** applies to every in-scope origin; a **full URL** targets one origin:
- `"/logout"` excludes `/logout` on the app's origin **and** on every extension origin.
- `"https://help.example.com/legal"` excludes `/legal` only on `help.example.com`.

### Backward Compatibility

- **Old browser** (knows only `scope`): ignores both new members and uses `scope` alone.
  - Ignoring `scope_inclusions` → the extra sections are simply *not* captured (under-inclusion — safe; the app never captures more than intended).
  - Ignoring `scope_exclusions` → excluded paths *are* captured (the exclusion "fails open"). This is **no worse than the status quo**: without this feature the developer would have shipped the same broad `scope` anyway.
- **New browser**: evaluates the full set-based rule above.

> **Developer rule:** design `scope` to be acceptable on its own — the best single-prefix scope you would ship today — because any browser that doesn't parse the new members falls back to `scope` alone. Treat `scope_inclusions` and `scope_exclusions` as progressive enhancement: inclusions degrade to under-capture and exclusions to today's behavior, so the feature is safe to adopt incrementally.

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

> **Not supported: re-inclusion.** Excluding a broad subtree and then re-including a specific leaf under it (e.g. exclude `/docs/internal/*` but re-include `/docs/internal/preview`) is intentionally **not** expressible, because it requires position-dependent precedence. No collected use case needs it, it does not map to any OS deep-link filter, and it can be added later as a compatible extension if real demand appears. See [Alternatives considered](#alternatives-considered) for the full rationale.

### Solving Use Case 4: Excluding Parameterized URLs

**Problem**: Every user's settings page should be kept out of scope, across all user ids.

```json
{
  "scope": "/users/",
  "scope_exclusions": ["/users/:id/settings"]
}
```

- `/users/:id/settings` matches `/users/123/settings`, `/users/456/settings`, and so on — removing the whole family with one entry.

### Solving Use Case 5: Cross-Origin Exclusion

**Problem**: The app extends into `help.example.com` via `scope_extensions`, but `help.example.com/legal` should open in the browser, not the app window.

```json
{
  "scope": "/",
  "scope_extensions": [{ "origin": "https://help.example.com" }],
  "scope_exclusions": ["https://help.example.com/legal"]
}
```

- `scope_exclusions` applies **globally**: an entry may name an origin the app has extended into, removing a path from that origin.
- This needs no consent from `help.example.com` (unlike cross-origin *inclusion*, which requires the `scope_extensions` handshake).
- A bare-path entry (e.g. `"/legal"`) instead applies to *every* in-scope origin.

> **Refining an extended origin.** Today `scope_extensions` extends into another origin with a prefix-based scope only. `scope_exclusions` can refine that extended scope by excluding specific paths — e.g. extend into `help.example.com`, then exclude `/internal/*`.

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
- **Re-inclusion does not map to any OS deep-link filter.** iOS Universal Links is include + `!exclude` (exclude-wins, no re-inclusion); Android/Windows are include-only. Flattening an ordered list to (include set, exclude set) **silently drops re-inclusion**, so an app relying on it would behave **differently via OS deep-link than in-browser** — a correctness divergence, not merely reduced precision.
- **Does not compose across origins.** First-match requires a single ordered list. But the future cross-origin design spreads include patterns across multiple origins' association files, each authored independently — so there is no single order to evaluate. Set-based matching doesn't depend on order, so it composes cleanly.
- **No grounded use case needs re-inclusion.** Every collected developer report is plain include-minus-exclude.

A **last-match-wins** variant (mirroring `.gitignore` negation) was also considered; it has the same order-sensitivity and OS-mapping problems.

#### Reason for rejection

The single ordered list's only advantage over the proposed two-list model is re-inclusion — and re-inclusion is unused, un-mappable to OS filters, and non-composable across origins, while its order-sensitivity actively harms authorability. If real demand for re-inclusion emerges, the two-list model can be extended to an ordered form compatibly (the reverse would be a breaking change), so choosing two-list now is the low-regret path.

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
- **Worse cross-origin composition**: a conditional "present ? … : …" does not federate across independently-authored sources the way set union does.
- **Changes `scope`'s established meaning** for existing tooling and readers.

#### Reason for rejection

Union keeps a single uniform set-based rule, preserves `scope`'s existing meaning, composes across origins, and — most importantly — degrades safely: a browser that drops the new members under-captures via a still-additive `scope`, never over-captures. Replace's extra expressiveness only enables semantically-odd scopes (those excluding the `start_url` region, which developers rarely want) at the cost of the over-capture failure mode.

## Dependencies on non-stable features

This proposal depends on the **URL Pattern Standard** ([urlpattern.spec.whatwg.org](https://urlpattern.spec.whatwg.org/)), specifically:

- §4.2 "Integrating with JSON data formats" — the "build a URL pattern from an Infra value" algorithm
- The concept of `has regexp groups` for optionally restricting patterns

URL Pattern is a WHATWG standard and is being implemented across major browser engines; see [MDN's browser compatibility table](https://developer.mozilla.org/en-US/docs/Web/API/URLPattern#browser_compatibility) for current support.

## Privacy and Security Considerations

### Privacy

No new privacy surface: scope matching is a local, in-browser decision that exposes no new data to the app or to other origins.

### Security

**Same-origin enforcement**: a `scope_inclusions` pattern may only match `scope`'s origin (entries whose `protocol`, `hostname`, or `port` differ are dropped). A `scope_exclusions` entry may additionally target an origin the app has extended into via `scope_extensions`. An app cannot claim arbitrary origins.

**Exclusions are not a security boundary**: an exclusion that fails to parse, is unsupported, or is ignored by an old browser results in *over-capture* (the URL stays in scope). Apps MUST NOT rely on exclusions for security isolation (e.g., hiding a sign-out flow); server-side authorization is the real boundary.

## Stakeholder Feedback / Opposition

- Chromium/Edge: Proposing (Microsoft Edge)
- Mozilla: TBD
- WebKit: TBD
- Web developers: Demand documented in [WICG/manifest-incubations#105](https://github.com/WICG/manifest-incubations/issues/105) and [w3c/manifest#996](https://github.com/w3c/manifest/issues/996)

## Open Questions

### Ship `scope_exclusions` alone first, or both members together?

Exclusions are the urgent, dominant real-world request; inclusions serve the real-but-less-common non-contiguous / parameterized cases. Options: (a) ship `scope_exclusions` only in v1 and add `scope_inclusions` later; (b) ship both together for a symmetric, complete model. Either order is safe — shipping `scope_inclusions` in a later release is a purely additive change, not a breaking one.

## Future Work: Cross-Origin Inclusion

Cross-origin **inclusion** is out of scope for this proposal. A future project can extend `scope_extensions` so that each associated origin declares the paths it lends *into* the app inside its own **web-app-origin-association (WAOA)** file, while the same app-authored `scope_exclusions` continues to apply across all origins. The WAOA's bidirectional handshake remains the security boundary for cross-origin trust; the app cannot expand capture on another origin without that origin authoring the grant.

## Acknowledgements

Many thanks for valuable feedback and advice from:

- Daniel Murphy
- Jacob Mills
- Vatan Aggarwal
- Howard Wolosky
- Alan Cutter
