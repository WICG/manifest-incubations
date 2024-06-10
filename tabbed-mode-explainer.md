# Tabbed Mode

Author: Louise Brett (loubrett@google.com)

## Overview

Currently PWAs in a standalone window can only have one page open at a time. Some apps expect users to have many pages open at once. Tabbed mode adds a tab strip to standalone web apps that allows multiple tabs to be open at once.

This document describes a new display mode `tabbed` and a new manifest member `tab_strip` which lets apps enable the tab strip and customize it.

## Proposal

Add a new display override option `tabbed` which behaves similarly to `standalone`, but with an added tab strip.

```json
"display_override": ["tabbed"]
```

Per the design of `display_override`, sites cannot request the `tabbed` display mode in the `display` member. This is because it would be backwards-incompatible with user agents that do not support the tabbed display mode. By forcing it into the `display_override` member, the site can specify an explicit fallback chain, ending with the declared `display` mode. (This means sites can decide whether unsupported browsers should fall back to `standalone` or `browser`, depending on what is more important to the site: tabs, or a standalone window.)

Add a new manifest field `tab_strip` which allows apps to customize the tab strip. The `tab_strip` field will only be used when the display mode is `tabbed`.

```
"tab_strip": {
    "home_tab": {
        "scope_patterns": [{"pathname": "..."}]
    },
    "new_tab_button": {
        "url": <url>,
    },
},
```

The home tab is a pinned tab that, if enabled for an app, should be present in all app windows. If the `home_tab` field is unset, then the app will not have a home tab.

The `home_tab.scope_patterns` field allows the app to set a list of [URLPatterns](https://wicg.github.io/urlpattern/#urlpattern) to define the scope of the home tab. This "home tab scope" carves the URL space of the application scope into two parts: "within home tab scope" and "outside of home tab scope". The home tab scope affects navigation in two important ways:

1. From within the home tab, a navigation to a URL outside of the home tab scope will open a new tab and perform the navigation there.
2. From outside the home tab, a navigation to a URL within the home tab scope will focus the home tab and perform the navigation there.

This "capturing" behaviour behaves a bit like `target=_blank` navigations, but applies to all navigations including regular `target=_self` links. Therefore, it explicitly breaks the normal HTML navigation flow, and is therefore something developers need to explicitly opt in to and be aware of. (As an example, a page may navigate itself to another URL, expecting its state to be destroyed, but due to the above home tab logic, the page may in fact stay open indefinitely.)

The `scope_patterns` member will be resolved against the manifest URL, and patterns must be within the app scope. If there is a home tab, the app's `start_url` will automatically be included in the home tab scope (this is necessary to ensure when a window is created there is a URL to navigate the home tab to). If `home_tab` is present but `scope_patterns` is absent, then there will be a home tab, but it will only include the `start_url`.

The new tab button should open a new tab at a URL that is within the scope of the app, but outside of the home tab scope. The app may choose to set a custom URL to be opened. If the new tab URL is within the scope of the home tab, the new tab button will be hidden. If this URL is not specified or is out of scope, it will default to the `start_url` (which means the button will, by definition, be hidden if there is a home tab).

If the `tab_strip` field is unset, it will default to the following object:
```
"tab_strip": {
    "new_tab_button": {
        "url": <start_url>,
    },
},
```

User agents can decide how to handle dragging these tabs around to create new windows or combine with browser tabs.

Apps can detect whether they have the tab strip enabled by checking the display mode with a media query.

```css
@media (display-mode: tabbed) {
  /* CSS to apply in tabbed application mode. */
}
```

```js
const tabbedApplicationModeEnabled = window.matchMedia('(display-mode: tabbed)').matches;
```

## Use cases

Tabbed mode is useful for any site where it is common to have more than one instance open at a time. Without tabbed mode, this would mean opening multiple windows for a single app which quickly becomes difficult to manage.

This can also be used instead of sites creating their own HTML-based tab strip.

The pinned home tab is suited for any site with a well defined home page that can be used to navigate the site. For example, productivity apps that have a home page with a gallery of current documents and a way to create a new documents. The home tab can be used as a menu to open existing files, all of which would then open in their own tab. These apps may also use a custom URL for the new tab button to quickly create a new document.

A use case for having the new tab button hidden is when the home tab is present and that page is sufficient to navigate the app. For example a news site where articles are opened from the home tab.

## Possible extensions

The proposed format makes it easy to add more fields to the `tab_strip` member in the future.

The `home_tab` field could have a `url` field, if an app wanted the home tab to open at a different URL to the `start_url` and an `icons` field to customize the icon displayed on the home tab.

The `new_tab_button` could have an `icons` field to customize the icon shown on the new tab button.

```
"tab_strip": {
    "home_tab": {
        "url": <url>,
        "icons": [...],
    },
    "new_tab_button": {
        "icons": [...],
    },
},
```

### Combining with other display modes

Some apps may want to use a combination of display modes together. An example of this is using [window controls overlay](https://wicg.github.io/window-controls-overlay/) with tabbed mode.

There is a temptation to design a maximally flexible system where sites can request any combination of display mode (e.g. `"window-controls-overlay, tabbed"`). However, this would require every browser to explicitly design, support and test all permutations of all display modes, making the addition of each additional display mode prohibitively expensive.

Instead, it was decided that *if* certain combinations of display modes were wanted in the future, we would explicitly design and implement those combinations; e.g., we could add a `window-controls-overlay-tabbed` display mode if there was enough support for it. This is less flexible, but scales in a controllable way. This is discussed in detail in the [Display Mode Override Proposal](https://github.com/WICG/display-override/blob/main/explainer.md#custom-display-mode-names-with-display-modifiers-style-specification).

## Interaction with Launch Handler API

The [Launch Handler API](https://wicg.github.io/web-app-launch/) lets sites redirect app launches into existing app windows to prevent duplicate windows being opened. When a tabbed app sets `"client_mode": "navigate-new"`, app launches will open a new tab in an existing app window.

## Security and Privacy

There are minimal security and privacy concerns with this proposal. Navigating out-of-scope will show the same out-of-scope UI that is used in standalone apps. This UI should only be shown on the out-of-scope tab.
