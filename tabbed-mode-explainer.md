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
    "home_tab": {
        "scope_patterns": [{"pathname": "..."}]
    },
    "new_tab_button": {
        "url": <url>,
    },
},
```

The home tab is a pinned tab that, if enabled for an app, should be present in all app windows. Navigations outside of the home tab scope (as specified by `scope_patterns`) should open in a new tab instead of navigating the home tab. If the `home_tab` field is unset, then the app will not have a home tab.

The `home_tab.scope_patterns` field allows the app to set a list of [URLPatterns](https://wicg.github.io/urlpattern/#urlpattern) to define the scope of the home tab. Navigation within the home tab going outside of this scope will be opened in a new tab, and navigation to a URL within the home tab scope will happen in the home tab. The [`URLPattern.baseURL`](https://wicg.github.io/urlpattern/#dom-urlpatterninit-baseurl) will be initialized to the parsed app scope, and apps will only be able to specify the pathname component of URLPattern. If the `scope_patterns` field is unset, then the home tab scope will only include the `start_url`.

The new tab button, should open a new tab at a URL that is within the scope of the app. The app may choose to set a custom URL to be opened. If this URL is not specified or is out of scope, it will default to the `start_url`. If the `new_tab_button` field is unset, it will default to an object with its subfields default values set. If the new tab URL is within the scope of the home tab, the new tab button will be hidden.

If the `tab_strip` field is unset, it will default to the following object:
```
"tab_strip": {
    "new_tab_button": {
        "url": <start_url>,
    },
},
```

User agents can decide how to handle dragging these tabs around to create new windows or combine with browser tabs.

Apps can detect whether they have the tab strip enabled by checking the display mode with a media query. Eg. `media="(display-mode: tabbed)"`.

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

These could be supported by adding more display override values, e.g., `window-controls-overlay-tabbed`, but this doesn’t scale well if many more display modes are added in the future.

Another solution is to allow apps to create custom display modes. See the [Display Mode Override Proposal](https://github.com/WICG/display-override/blob/main/explainer.md#custom-display-mode-names-with-display-modifiers-style-specification) for more detail.

## Interaction with Launch Handler API

The [Launch Handler API](https://wicg.github.io/web-app-launch/) lets sites sites redirect app launches into existing app windows to prevent duplicate windows being opened. When a tabbed app sets `"client_mode": "navigate-new"`, app launches will open a new tab in an existing app window.

## Security and Privacy

There are minimal security and privacy concerns with this proposal. Navigating out-of-scope will show the same out-of-scope UI that is used in standalone apps. This UI should only be shown on the out-of-scope tab.
