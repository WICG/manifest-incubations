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

Add a new manifest field `tab_strip` which allows apps to customize the tab strip. The `tab_strip` field will only be used when the display mode is `tabbed`.

```
"tab_strip": {
    "home_tab": "absent" | {
        "icons": [...],
        "scope_patterns": [{"pathname": "..."}]
    },
    "new_tab_button": "absent" | {
        "url": <url>,
    },
},
```

The home tab is a pinned tab that, if enabled for an app, should be present in all app windows. This tab should not navigate outside of the home tab scope (as specified by `scope_patterns`), navigations outside of this scope should open in a new tab. If the `home_tab` field is unset, it will default to `absent`. Apps can choose to customize the icon displayed on the tab.

The `home_tab.scope_patterns` field allows the app to set a list of [URLPatterns](https://wicg.github.io/urlpattern/#urlpattern) to define the scope of the home tab. Navigation within the home tab going outside of this scope will be opened in a new tab, and navigation to a URL within the home tab scope will happen in the home tab. The [`URLPattern.baseURL`](https://wicg.github.io/urlpattern/#dom-urlpatterninit-baseurl) will be initialized to the parsed app scope, and apps will only be able to specify the pathname component of URLPattern. If the `scope_patterns` field is unset, then the home tab scope will only include the `start_url`.

The new tab button, if present, should open a new tab at a URL that is within the scope of the app. The app may choose to set a custom URL to be opened. This URL will default to the `start_url`. If the `new_tab_button` field is unset, it will default to an object with its subfields default values set:

```
"new_tab_button": {
    "url": <start_url>,
},
```

User agents can decide how to handle dragging these tabs around to create new windows or combine with browser tabs.

## Use cases

A use case for the pinned home tab is productivity apps that allow editing multiple documents at once. The home tab can be used as a menu to open existing files, all of which would then open in their own tab. These apps may also use a custom URL for the new tab button to quickly create a new document.

A use case for an `absent` new tab button would be if the home tab is present and that page is sufficient to navigate the app. For example a news site where articles are opened from the home tab.

## Possible extensions

The proposed format makes it easy to add more fields to the `tab_strip` member in the future.

The `home_tab` field could have a `url` field, if an app wanted the home tab to open at a different URL to the `start_url`.

The `new_tab_button` could have an `icons` field to customize the icon shown on the new tab button.

```
"tab_strip": {
    "home_tab": {
        "url": <url>,
    },
    "new_tab_button": {
        "icons": [...],
    },
},
```

### Combining with other display modes

Some apps may want to use a combination of display modes together. An example of this is using [window controls overlay](https://wicg.github.io/window-controls-overlay/) with tabbed mode.

These could be supported by adding more display override values, e.g., `window-controls-overlay-tabbed`, but this doesn’t scale well if many more display modes are added in the future.

Another solution is to allow apps to create custom display modes. See the [Display Mode Override Proposal](https://github.com/WICG/display-override/blob/main/explainer.md#custom-display-mode-names-with-display-modifiers-style-specification) for more detail.

## Security and Privacy

There are minimal security and privacy concerns with this proposal. Navigating out-of-scope will show the same out-of-scope UI that is used in standalone apps. This UI should only be shown on the out-of-scope tab.
