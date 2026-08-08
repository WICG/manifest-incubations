# Display Mode Override Proposal


# Authors:

dmurph@chromium.org


# Participate:

Currently discussion is in this github issue: 
[w3c/manifest#856](https://github.com/w3c/manifest/issues/856).

Please use this [issue tracker](https://github.com/WICG/manifest-incubations/issues) to record any issues or feedback. 

Intent-to-Prototype email: https://groups.google.com/a/chromium.org/g/blink-dev/c/WvIeZT8uSzw

TAG Review: https://github.com/w3ctag/design-reviews/issues/530

Pull request: https://github.com/w3c/manifest/pull/932

# Introduction

Web apps can choose how they are displayed by setting a 'display' mode in their manifest, like so:

```json
{
  "display": "standalone"
}
```

The manifest spec has a list of display modes [here](https://w3c.github.io/manifest/#display-modes). Browsers are NOT required to support all display modes, but they ARE required to support the spec-defined fallback chain ("fullscreen" -> "standalone" -> "minimal-ui" -> "browser"). If they don't support a given mode, the fall back to the next display mode in the chain.

**The current `display` property will work for 95% (most) of developers and web apps. Most web apps are perfectly happy with `standalone` or `minimal-ui`, and they have no reason to use the following new feature. Instead, this feature is for more advanced & fully featured web apps that require more advanced display modes (tabbed & customized) and/or fallback chains. It also paves the way for even more customization in the future.**

New display modes are being proposed, and the current way of specifying a display mode in the manifest has a [static fallback chain](https://w3c.github.io/manifest/#display-modes), which can:
 * Prevent developer from using display modes that may make them not PWAs on unsupporting UAs - A developer cannot, for example, request `minimal-ui` without being forced back into the `browser` display mode (essentially making it a non-PWA) on unsupporting UAs.
 * Forces display modes onto developers that they do not want. - A developer MUST handle all display modes the follow the requested mode. If they want `fullscreen`, and `tabbed` is introduced after `fullscreen` in the display mode list, they must support a `tabbed` display mode, even if they don't want it

To elaborate a bit more, the current `display` mode has the following issues:

1.   **The fallback chain is inflexible for developers.** A developer cannot specify they want `minimal-ui` and then fallback to `standalone` if that is not supported. Instead they must fail down to `browser`, which loses them a PWA window.
2.   **Developers have no way of handling cross-user-agent differences**, like if the user-agent includes or excludes a back button in the window for 'standalone' mode
3.   **New display modes don't have a clear 'place' in the current static fallback chain**. Example: [Tabbed Application Mode](https://github.com/w3c/manifest/issues/737) & [Window Control Overlay](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/master/TitleBarCustomization/explainer.md) ([issue](https://github.com/MicrosoftEdge/MSEdgeExplainers/issues/206)). Especially because this could force developers into a display mode they don't want to support at all.

This proposal is a result of the post [w3c/manifest#856](https://github.com/w3c/manifest/issues/856) by Matt Giuca on github outlining these problems in more detail.

As more display mode options arise (tabs, window-control-overlay, maybe back button, etc), I am predicting that developers will end up needing only a fixed number of display mode combinations. This proposal should be all that is necessary if that is the case.

However, if that is NOT the case and more intense customization is necessary, this proposal should work well with adding a `display modifiers`-style future API. See [Custom display mode names with display-modifiers-style specification](#custom-display-mode-names-with-display-modifiers-style-specification) for an idea here.

# How does this effect the user?

For most web apps, this will have no effect on the user or the developer. 95% of WebApp use-cases will be solved by one of the existing display modes in the specification.

For more advanced web apps, spec editors will struggle to create new display mode & web developers will struggle to use new display modes, resulting in the following cases:
 * PWAs will only work for certain UAs that support the exact display mode that is requested by the developer (example - if Spotify wants `minimal-ui`, they will only have a PWA window for browsers that support `minimal-ui`. Otherwise the user agent is forced to fall back to `browser`, which would just open Spotify in a browser tab.
 * PWAs will appear in unexpected display modes for users that have UAs that don't support the requested display mode. Example: if DropBox wanted to have a PWA with `tabbed` mode, but only wanted that mode, as they are a multiple-document-appplication, they would be forced to supported a `standalone` mode that they don't want, and wouldn't offer any functionality to the user.
 * PWAs not being created at all, as the developer cannot get the display mode or fallback configuration that they want & can support.

# Goals

**Acceptable fallback behavior** - Fallback behavior is acceptable to web developers. It doesn't force them into `browser` (kicking them out of the PWA experience), and it doesn't force them into a display mode they don't want at all.

**Acceptable user-agent supportability** - This shouldn't result in an overwhelmingly exponential number of display combinations that a user agent must support.

**Control visibility** - The developer must be able to detect exactly what user agent controls are available to them on their given window.

**Simplicity** - The API should be relatively simple to understand and use.

**Predictability** - The developer should be able to easily predict what the display mode configuration is for their website.


# Non Goals

**Enable every combination of display mode**

It seems too early to know if this is necessary, and at the very least this proposal shouldn't block this happening in the future.

**Remove or redesign the core fallback behavior of `display` field or modes**

This API seems like it isn't suiting our needs, and is hard to change while continuing to be backwards compatible. Better to create a new surface.


# Part 1: `display_override`

Add a new field, sequence of strings `display_override`, which the user agent considers before the `display` field. These are considered in-order, and the first supported display mode is applied. If none are supported, the user agent falls back to [evaluating](https://w3c.github.io/manifest/#display-modes) the `display` field (as already specified).

Example:


```json
{
  "display": "standalone",
  "display_override": ["window-control-overlay", "minimal-ui"],
}
```


In this example, the display mode fallback chain would be spec'd as:



1.  `window-control-overlay`
1.  `minimal-ui`
1.  `standalone` (`display_override` is exhausted, [evaluating](https://w3c.github.io/manifest/#display-modes) `display` now)
1.  `minimal-ui`
1.  `browser`

Requirements:

*   A user agent will not consider `display_override` unless `display` is also present.
*   The new display modes cannot be used with `"display"`.

New modes would only be allowed to be specified in the `"display_override"` field.


## Fallback Behavior

This proposal allows developers to request the specific display modes they want to support, and in the fallback order they want to display. The use case of `minimal-ui` falling back to `standalone` is now possible.


## User Agent Supportability

This lets the user agent support a fixed list (full list defined by spec) of chosen display options. A proposal like `display-modifiers` (below) creates a cross product of possible configuration which is very difficult for a UI team to support well, especially when every new feature can create a exponential amount of new display combinations.

In this API design, browsers can support (and document that they support, say on MDN) UI configurations that they have specifically designed & approved.

Note: this goes away with [Custom display mode names with display-modifiers-style specification](#custom-display-mode-names-with-display-modifiers-style-specification).

## Simple feature detection

When a user agent supports a mode, that is predictable and fully supported. If it does not, it appropriately falls back. This avoids complicated option detection within a display mode, and is obvious to a developer what is happening. There is no partial support.


## Backwards compatible

A user-agent that doesn't support `display_override` would fall back to the `display` field, which is fine and is predictable.


## Con - Possible large number of modes

If tabbed & title bar customization are approved, then this could be the list of modes:

1.  `minimal-ui`
1.  `standalone`
1.  `window-control-overlay`
1.  `tabbed`
1.  `fullscreen`
1.  `browser`
1.  `window-control-overlay-tabbed`
1.  `minimal-tabbed` [potentially remove this]

Not supported, as probably redundant

*   `window-control-overlay-minimal`
*   `window-control-overlay-tabbed-minimal`

Even though this seems long, this is probably how user agents would have to evaluate the combinations from a UX design perspective. These all need to be enumerated to be properly tested as well. So having them listed separately (and supported separately) seems acceptable.

If I put my product hat on, I would consider this a pro, as this would force the team to make sure that every display mode is appropriately designed & tested.

There might be a discussion here worth having around tradeoffs of UA differentiation and room for interpreting/updating modes, versus guarantees made to web app authors (eg. would tabbed always show back/forward/reload, is that up to the UA, or will web apps want to specify that themselves, and with how much granularity?)

Note: this goes away with [Custom display mode names with display-modifiers-style specification](#custom-display-mode-names-with-display-modifiers-style-specification).


## Con - Element Coupling may force features on developers

Since the list isn't meant to be a full cross product of all display mode options, the following two scenarios may occur:

*   Developers cannot find a configuration that has all of the options they desire, or
*   Developers are given display elements they don't want or need.

Example: button X is a new display option that is supported only in tabbed mode (so there is a new string, `tabbed-button-X`. Developers who want button X will be forced to also be in tabbed mode.

This is hopefully avoided by carefully considering the use cases of each display element, but it might happen.

Note: this goes away with [Custom display mode names with display-modifiers-style specification](#custom-display-mode-names-with-display-modifiers-style-specification).


# Part 2: Using media query for browser control elements like back button

Proposed by mgiuca@chromium.org in [w3c/manifest#693](https://github.com/w3c/manifest/issues/693) and explainer written by [@fallaciousreasoning](https://github.com/fallaciousreasoning) [here](https://github.com/fallaciousreasoning/backbutton-mediaquery).

This solution is necessary to detect differences in user agent's support for `standalone`, and gives the developer a robust way to modify their logic.

I'm expecting that this change will be a separate proposal / pull request. I wanted to mention it here for completeness, as its existence I think is necessary for this proposal to work.


# Key Scenarios

Note, again, that these scenarios represent **advanced** use cases. Most developers will not need to use `display_override` at all, and can just use `display`.

## 1) `minimal-ui` with `standalone` fallback

The developer prefers `minimal-ui` but can settle for `standalone`.


```json
{
  "display": "standalone",
  "display_override": ["minimal-ui"],
}
```

**Why is this not currently possible?**

Without `display_override`, Setting `minimal-ui` as the `display` property will
 * give the WebApp `minimal-ui` on browsers that support it, BUT
 * force the WebApp into the `browser` display mode (which makes it not a WebApp anymore, it just opens in a tab), as that is the next display mode in the fallback chain.

## 2) Multi-document application

A webapp that only wants a dedicated window if there can be tabs.


```json
{
  "display": "browser",
  "display_override": [ "window-control-overlay-tabbed", "tabbed"],
}
```

**Why does `display` not work for this use case?**

If `tabbed` (and `window-control-overlay-tabbed`) were added to the current `display` list (and fallback chain), it is not easy to determine where they should go.
 * If they are near the begining (right after `fullscreen`), then an app that wants these features MUST support all display modes below that (so `standalone` & `minimal-ui`), even if their webapp doesn't work well in that environment (in this example, they need tabs).
 * If they go near the beginning, say after `minimal-ui`, apps are forced to support these modes if the browser doesn't support `minimal-ui` (& that is what they set in their manifest).

## 3) in-app browser controls (& doesn't want `minimal-ui`)

A webapp wants to display their own browser controls if they are in a window, either through the 'customized' API or in their own in-app header. e.g. They do NOT want `minimal-ui`.


```json
{
  "display": "browser",
  "display_override": [ "window-control-overlay", "standalone"],
}
```

**Why does `display` not work for this use case?**

There is no way for the developer to tell the browser they do NOT want `minimal-ui`, however it is forced on them by the fallback list behavior.

## 4) No `minimal-ui`.

A webapp doesn't want `minimal-ui`.


```json
{
  "display": "browser",
  "display_override": ["standalone"],
}
```

## 4)  `fullscreen` or bust.

A webapp only wants to be a webapp if it can be fullscreen.


```json
{
  "display": "browser",
  "display_override": ["fullscreen"],
}
```

**Why does `display` not work for this use case?**
`display` requires that browsers who dont support fullscreen MUST fall back to `standalone` for this use case. The web app has no way of having that not happen.

This is more extreme in examples where new display modes need to be added - for example if `tabbed` mode is added after `fullscreen`, then `fullscreen` apps must expect to be in `tabbed` mode if the user agent doesn't support `fullscreen`.

# Future Considerations/Ideas


## Custom display mode names with `display-modifiers`-style specification

In one possible future, with each new display mode customization there is only one or two combinations with other features that developers want, and thus this API is scalable and mission accomplished.

**However**, there is a possible future where, as more display modes are added, developers need an increasingly large space of options that cannot be feasibly captured by individual feature strings, compounding too much with all of the requested possibilities.

At its core, `display_override` provides a developer-controlled display mode fallback functionality. This seems necessary no matter what. More customizable APIs could be added and interact symbiotically with this API.

One example is to allow developers to create their own display modes like so:


```json
{
  "display": "standalone",
  "display_override": ["back-tabbed-color", "tabbed-color", "tabbed"],
  "custom-display-modes": {
    "back-tabbed-color": {
      "required": {
        "back-button": true,
        "tabbed": true,
        "tabbed-color": "0x23F0A3",
      },
      "optional": {
        "reload-button": false,
        "forward-button": false,
      },
    },
    "tabbed-color": {
      "required": {
        "tabbed": true,
        "tabbed-color": "0x23F0A3",
      },
      "optional": {
        "reload-button": false,
        "forward-button": false
      },
    }
  }
}
```


In this example, the `custom-display-modes` specify custom display modes, similar to `display-modifiers`, except there are two parts:

*   `required` specifies the display modifiers that **need** to be supported by the browser for this display mode to become active. The configuration is used IFF all requirements are supported.
*   `optional` specifies the display modifiers that **may** be supported by the browser (and will be applied if they are supported).

Then, these display mode names can be used in the `display_override` list just like any of the built in display modes.


## No new entries in `display` mode fallback chain

Adding a new display mode to the `display` fallback chain could cause developer's previous expectations to become invalid. Ex: "I used to have a `fullscreen` app that fell back to a `standalone` window. Now I fall back to a `tabbed` window that I don't want.".


## Display Mode Options

If options were to be added for display modes, this proposal does not block that. These options can be specified as top level entries in the manifest (one option field per display mode possibility).


# Considered Alternatives

## Allow absence of `display` property

**NOTE:** Please comment on [Issue #10](https://github.com/dmurph/display-mode/issues/10) if you have opinions here.


As [Issue #10](https://github.com/dmurph/display-mode/issues/10) states, we could instead have this property be an alternative to the `display` property instead of an override. This means our end-state is less confusing to developers, as instead of needing to put:

```json
{
  "display": "standalone",
  "display_override": ["minimal-ui"],
}
```

they can write:

```json
{
  "display_override": ["minimal-ui", "standalone"],
}
```

Notes:
 * The fallback chain would have an inherent `"kBrowser"` value at the end (no need to specify).
 * The name could change, as it's no longer overriding. Maybe `display_fallback_list`?
 * The spec could say that this field overrides `display` if present.

The downside of this approach is that it would make new apps potentially not backwards compatible.

## Using a 'string' instead of a json field for options

For fallback purposes, each entry is effectively a key. This becomes more confusing to the developer if the key is a json entry. How does this behave? Does the browser only use that entry if it can support all of those options, and fall back if any don't work? Or, does it stay on the requested mode even if only some of the options apply?

This is too confusing. Instead, future display mode options (if they ever exist) can exist as a top level entry for the display mode they apply to.


## Specifying fallback list on `display` field

Instead of having a new field, the spec could just specify that `display` now accepts an array of strings, which could be used to implement the fallback behavior above.

This doesn't work because it breaks backwards compatibility. If a UA doesn't support this new array type, then the manifest would be invalid and the webapp wouldn't be installable at all.


## Display Modifiers

(copying from mgiuca@'s comment on [w3c/manifest#856](https://github.com/w3c/manifest/issues/856)):

> This was proposed by [@amandabaker](https://github.com/amandabaker) on [MicrosoftEdge/MSEdgeExplainers#206](https://github.com/MicrosoftEdge/MSEdgeExplainers/issues/206), and riffed on [in comments](https://github.com/MicrosoftEdge/MSEdgeExplainers/issues/206#issuecomment-592138391) by [@aarongustafson](https://github.com/aarongustafson). Keep the existing `display-mode`, but add a new list or dictionary member, `display-modifiers,` which lets developers add or remove things piecemeal.

> For example, Aaron's proposal presents the new title bar customization feature as simply a "removal" of the `"titlebar"` feature:


```
{
  "display": "standalone",
  "display_modifiers": {
    "titlebar": false
  }
}
```

> But this also allows you to explicitly add or remove individual elements like back button, refresh button, etc. I think these would still have to be hints to the UA (we can't mandate that the UA show a refresh button, for instance), but they could be SHOULD requirements.


### Easy customization for Developers

This API makes it very easy for developers to ask for exactly what they want & don't want.


### Con: Difficult/Impossible Fallback support

It is very difficult for this API design to handle the case where a developer wants `standalone` ONLY if there is a back button supported, and otherwise they want, say, `minimal-ui`. There isn't a simple way to do feature detection & fallback here


### Con: Difficult Feature detection

It is difficult for developers to detect what features the browser supports here & respond accordingly - they must ask for everything they want, and be forced to deal with the subset of features UA gives them.


### Con: Exponential UI Configurations

This style of API would force user agents to support an exponential amount of UI configuration, as every new modifier increases the complexity of combinations by another magnitude.

However, this might be necessary for supporting all use cases.


## Add new features as separate members

(copying from mgiuca@'s comment on [w3c/manifest#856](https://github.com/w3c/manifest/issues/856)):

> This is the most straightforward. We keep the existing display modes, but don't add any new ones. All new features are "add-ons" which have their own member. For example:

```json
{
  "display": "standalone",
  "tabbed_mode": true,
  "caption_controls_overlay": true
}
```


### Easy customization for Developers

This API makes it very easy for developers to ask for exactly what they want & don't want.


### Con: Difficult/Impossible Fallback support

It is very difficult for this API design to handle the case where a developer wants `standalone` ONLY if there is a back button supported, and otherwise they want, say, `minimal-ui`. There isn't a simple way to do feature detection & fallback here


### Con: Difficult Feature detection

It is difficult for developers to detect what features the browser supports here & respond accordingly - they must ask for everything they want, and be forced to deal with the subset of features UA gives them.


### Con: Exponential UI Configurations

This style of API would force user agents to support an exponential amount of UI configuration, as every new modifier increases the complexity of combinations by another magnitude


# References & acknowledgements

References:
* [w3c/manifest#856](https://github.com/w3c/manifest/issues/856) - Github issue discussion on this topic
* [w3c/manifest](https://github.com/w3c/manifest) - Manifest spec
* [MicrosoftEdge/MSEdgeExplainers#206](https://github.com/MicrosoftEdge/MSEdgeExplainers/issues/206) zdisplay-modifiers proposal by @amandabaker
  * Riffed on [in comments](https://github.com/MicrosoftEdge/MSEdgeExplainers/issues/206#issuecomment-592138391) by @aarongustafson
* Media queries proposed by mgiuca@chromium.org in [w3c/manifest#693](https://github.com/w3c/manifest/issues/693) and explainer written by @fallaciousreasoning [here](https://github.com/fallaciousreasoning/backbutton-mediaquery).

Thanks to the following people who helped me brainstorm & create this proposal:
*   Alan Cutter <alancutter@chromium.org>
*   Chase Phillips <cmp@chromium.org>
*   Joshua Bell <jsbell@chromium.org>
*   Matt Giuca <mgiuca@chromium.org>
*   Mike Wasserman <msw@chromium.org>
*   PJ Mclachlan <pjmclachlan@chromium.rog>
*   <your name here alphabetically :)>
