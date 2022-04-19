# User Preferences Explainer

Authors: [Aaron Gustafson](https://github.com/aarongustafson), Louise Brett (loubrett@google.com)

## Overview

Currently in the web app manifest a `theme_color` and `background_color` can be defined. However there is no way to change these based on dark mode or [other user preferences](#possible-extensions).

This document proposes a new `user_preferences` manifest member that enables web apps to provide alternate values for manifest members given specific user preferences. The structure of this `user_preferences` field matches the structure of the proposed <code>[translations](https://github.com/WICG/manifest-incubations/blob/gh-pages/translations-explainer.md)</code> member.

This new member is inspired by the [CSS media queries user preferences](https://drafts.csswg.org/mediaqueries-5/#mf-user-preferences). The keys are simple strings but they are derived from the CSS media query syntax.

## Proposal

Add a new dictionary manifest entry `user_preferences`, mapping preference strings to a [`ManifestOverride` object](#manifestoverride-object).

Initially, the valid keys of the `user_preferences` member are `color_scheme_dark` and `color_scheme_light`. This could be expanded in the future to include other [user preference media features](#possible-extensions).

### `ManifestOverride` Object

The `ManifestOverride` is a generic object that contains a subset of redefined manifest properties appropriate to the context (e.g. `user_preferences`) in which the `ManifestOverride` is being used. The properties that may be redefined in the `ManifestOverride` object depend on the context (e.g. `user_preferences`). Any properties not allowed within the context will be ignored. The redefined fields override the values set in the root of the manifest.

For the `user_preferences` member, the acceptable keys for the `ManifestOverride` include:

*   `theme_color`
*   `background_color`
*   `icons`
    *   `src`
    *   `type`
*   `shortcuts`
    *   `icons`
        *   `src`
        *   `type`

### Example

```json
{
  "user_preferences": {
    "color_scheme_dark": {
      "theme_color": "#000",
      "background_color": "#000"
    },
    "color_scheme_light": {
      "theme_color": "#fff",
      "background_color": "#fff"
    }
  }
}
```

When a user has dark mode enabled, the fields redefined for `color_scheme_dark` will be used instead of the top level fields.

## Possible extensions

In addition to the preferences `color_scheme_dark` and `color_scheme_light`, other CSS [user preference media features](https://drafts.csswg.org/mediaqueries-5/#mf-user-preferences) could be added as valid keys. These include:

*   Prefers reduced motion
*   Prefers reduced transparency
*   Prefers contrast
*   Forced colors
*   Prefers reduced data

## Security and Privacy Considerations

There are minimal security and privacy concerns with this proposal. This will allow sites to know what user preferences are set and use this for fingerprinting, however this information is already exposed through CSS.

## Alternatives considered

It would also be possible to use the CSS media query syntax for the keys (e.g. `(prefers-color-scheme: dark)`) and parse this as CSS instead of using fixed keys as proposed. However, using the CSS parser adds a lot of complexity which we would like to avoid. Before a web app can launch, the CSS parser would need to be run to analyze the media query. This would also allow media queries which don’t make sense for this proposal, such as window size.

## Open questions

How does this interact with `translations`  for fields that can be overridden by either `user_preferences` or `translations`, such as icons?

For sites that have an in app dark mode setting, how can they communicate that the selected theme is different from the system theme?
