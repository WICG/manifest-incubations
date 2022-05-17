# Tabbed Mode

Author: Louise Brett (loubrett@google.com)

## Overview

Currently PWAs in a standalone window can only have one page open at a time. Some apps expect users to have many pages open at once. Tabbed mode adds a tab strip to standalone web apps that allows multiple tabs (within the app's scope) to be open at once.

This document describes a new display mode `tabbed` and a new manifest member `tab_strip` which lets apps enable the tab strip and customize it.

## Proposal

Add a new display override option `tabbed` which behaves similarly to `standalone`, but with an added tab strip.

```json
"display_override": ["tabbed"]
```

Add a new manifest field `tab_strip` which allows apps to customize the tab strip. The `tab_strip` field will only be used when the display mode is `tabbed`.

```
"tab_strip": {
    "home_tab": "auto" | "absent" | {
        "url": "auto" | <url>,
        "icons": "auto" | [...],
    },
    "new_tab_button": "auto" | "absent" | {
        "url": "auto" | <url>,
        "icons": "auto" | [...],
    },
},
```

The home tab is a pinned tab that, if enabled for an app, should always be present when the app is open. This tab should never navigate - links clicked from this tab should open in a new app tab. Apps can choose to customize the URL this tab is locked to and the icon displayed on the tab.

The new tab button, if present, should open a new tab at a URL that is within the scope of the app. The app may choose to set a custom URL and icon for this button.

If the `tab_strip` field is not present, the particular subfields' `auto` values should be used. The user agent can decide what values to use for `auto`. For example it might set `url` to be the `start_url`.

User agents can decide how to handle dragging these tabs around to create new windows or combine with browser tabs.

## Use cases

A use case for the pinned home tab is productivity apps that allow editing multiple documents at once. The home tab can be used as a menu to open existing files, all of which would then open in their own tab. These apps may also use a custom URL for the new tab button to quickly create a new document.

A use case for an `absent` new tab button would be if the home tab is present and that page is sufficient to navigate the app. For example a news site where articles are opened from the home tab.

## Possible extensions

The proposed format makes it easy to add more fields to the `tab_strip` member in the future.

A possible extension to the home tab is a `scope` field. This could allow for the home tab to navigate within that scope, and only open links in new tabs that are outside the scope.

### Combining with other display modes

Some apps may want to use a combination of display modes together. An example of this is using [window controls overlay](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/TitleBarCustomization/explainer.md) with tabbed mode.

These could be supported by adding more display override values, e.g. `window-controls-overlay-tabbed`, but this doesn’t scale well if many more display modes are added in the future.

Another solution is to allow apps to create custom display modes. See the [display override explainer](https://github.com/WICG/display-override/blob/main/explainer.md#custom-display-mode-names-with-display-modifiers-style-specification) for more detail.


## Security and Privacy

There are minimal security and privacy concerns with this proposal. Navigating out-of-scope will show the same out-of-scope UI that is used in standalone apps. This UI should only be shown on the out-of-scope tab.
