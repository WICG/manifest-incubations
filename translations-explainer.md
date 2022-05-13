# Web App Translations

Authors: [Aaron Gustafson](https://github.com/aarongustafson), Louise Brett (loubrett@google.com), Glen Robertson (glenrob@chromium.org)

## Overview

This document proposes a new `translations` manifest member that enables web apps to provide alternate values for manifest members in specific languages. 

## Problem

Currently the Web App Manifest has no direct support for localization so fields can only be provided in a single language. There are two recommended ways to translate the manifest suggested in [non-normative text in the spec](https://www.w3.org/TR/appmanifest/#internationalization):

*   Using a different manifest URL for each language and having HTML point to the correct one.
*   Dynamically serving a localized manifest from the same URL.

There are several problems with these approaches:

*   For the first option having a different manifest URL can cause some user agents to consider it a different app.
*   The second option requires dynamic logic on the server side.
*   For both of these options the app will not update immediately when the language is changed.
*   For both options there is no way for a crawler to index all supported languages.

## Proposal

Add a new dictionary manifest entry `translations`, mapping locale strings to a [`ManifestOverride` object](#manifestoverride-object).


### `ManifestOverride` Object

A `ManifestOverride` is a generic object that contains a subset of redefined manifest properties appropriate to the context in which the `ManifestOverride` is being used. The context (e.g., `translations`) will define which properties may be redefined. Any properties not allowed within the context will be ignored.

If the condition governing a `ManifestOverride` is met (e.g., the locale string matches the user agent language), the override is active. When active,

* Redefined string properties will override the initial value set in the root of the Manifest.
* Redefined array items
  * **Will be overridden in the order they are authored.** When redefining objects (e.g., `ShortcutItem`, `ImageResource`), authors will only be able to redefine specific properties of that object. In order to ensure all overrides are applied correctly, the order must match the original array (i.e., each `ShortcutItem` must be redefined in order, as must their `icons`, if they also require re-definition).
  * **Should be equal in number to the array being overridden**. If there is a mismatch in the number of items in either array, any excess items will be ignored. This is only an issue if the original array has more items than the override array, because any excess items within the original array will not be re-defined.

<p id="translatable-members">In the context of the `translations` member, acceptable keys for the `ManifestOverride` include:</p>

*   `dir`
*   `name`
*   `short_name`
*   `description`
*   `icons`
    * `src`
    * `type`
*   `screenshots`
    * `src`
    * `type`
    * `label`
*   `shortcuts`
    * `name`
    * `short_name`
    * `description`
    * `icons`
      * `src`
      * `type`

### Simple `translations` Example

When the user agent language matches a language code key in the `translations` object, those fields can be used instead of the top-level fields. If none of the provided languages match, the top-level fields can be used.

```json
{
  "lang": "en",
  "name": "Good dog",
  "description": "An app for dogs",
  "translations": {
    "fr": {
      "name": "Bon chien",
      "description": "Une application pour chiens",
    }
  }
}
```

In the above example, people who have their language set to French would install a web app named "Bon chien," whereas people who have their language set to anything other than French would install a web app named "Good dog."

### Complex `translations` Example

For array members (e.g., `icons`, `screenshots`, `shortcuts`), only a subset of each item’s properties are open to redefinition. For example, a `ShortcutItem`’s `name` may be translated, but its `url` cannot be changed. [A complete list of translatable properties can be found above](#translatable-members). Any attempts to redefine properties other than those allowed will be ignored.


```json
"shortcuts": [
  {
    "name": "Pet Me",
    "url": "/pet-me"
  },
  {
    "name": "Feed Me",
    "url": "/feed-me"
  }
],
"translations": {
  "fr": {
    "shortcuts": [
      {
        "name": "Caressez-moi"
      },
      {
        "name": "Nourrissez-moi"
      }
    ]
  }
},
```

To ensure translations for these complex constructs do not get out of sync with their counterparts in the original arrays, it’s imperative that developers pay close attention to both the number and order of array members. Failure to do so, creates an opportunity for any/all of the following:

* The first and second `ShortcutItem`s are swapped, but the corresponding translations are not. When this happens, the wrong translations are applied and users will be incredibly confused.
* A new `ShortcutItem` is appended to the array, without corresponding translation(s). When this happens, the last shortcut would remain untranslated and appear in the default language of the Manifest.
* A new `ShortcutItem` is prepended to the array, without corresponding translation(s). When this happens, the translations get out of sync, with the original translation(s) of the first `ShortcutItem` being applied to the new item, and the final `ShortcutItem` would not be translated. Users will be incredibly confused.


## Possible extensions

In addition to the language string mapping to an object containing the translations as proposed, it could map to a string that defines the location of a separate JSON file containing the translations.


manifest.json:
```json
{
  "name": "Good dog",
  "description": "An app for dogs",
  "icons": [],
  "screenshots": [],
  "lang": "en",
  "translations": {
    "fr": "manifest.fr.json"
  }
}
```

manifest.fr.json:
```json
{
  "name": "Bon chien",
  "description": "Une application pour chiens",
  "icons": [],
  "screenshots": []
}
```


Since the translations field could become quite large and difficult to manage in the single file, this option reduces the size of the manifest and allows user agents to only download the languages they need. It is not proposed as the initial implementation due to the complexity and overhead of downloading a separate file for every language.

## Security and Privacy Considerations

Allowing apps to specify different names and icons introduces a potential spoofing concern as apps could pretend to be a different app once the language has changed. User agent developers should be aware of this and consider measures to prevent spoofing. For example, they might make it obvious to the user what has changed for each app when the language changes or show the alternatives at install time.

## Alternatives Considered

Some alternatives were considered on the [original issue thread](https://github.com/w3c/manifest/issues/676):

1. Have the translations field point to a single separate file which contains all of the translations. This has the advantage of keeping the manifest file small. However this adds the complexity of having another file to download without the benefit of only downloading the necessary languages (which is possible with the extension option above).
2. Have the translations field keyed by manifest member rather than by language:

```json
{
  "name": "Good dog",
  "description": "An app for dogs",
  "lang": "en",
  "translations": {
    "name": {
      "fr": "Bon chien"
    },
    "description": {
      "fr": "Une application pour chiens"
    },
  "icons": []
}
```

This approach is less readable for complex fields such as shortcuts, for little/no benefit over the proposed option.
