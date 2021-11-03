# Web App Translations

Authors: [Aaron Gustafson](https://github.com/aarongustafson), Louise Brett (loubrett@google.com), Glen Robertson (glenrob@chromium.org)

## Overview

This document proposes a new `translations` manifest member that enables web apps to provide alternate values for manifest fields for different languages. 

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

Add a new dictionary manifest entry `translations`, mapping locale strings to an object containing the translations.

```
{
  "name": "Good dog",
  "description": "An app for dogs",
  "icons": [],
  "screenshots": [],
  "lang": "en",
  "translations": {
    "fr": {
      "name": "Bon chien",
      "description": "Une application pour chiens",
      "icons": [],
      "screenshots": []
    }
  },
}
```

When the user agent language matches a language present in the translations entry, those fields can be used instead of the top-level fields. If none of the provided languages match, the top-level fields can be used.

Not every field in the Web App Manifest needs to be localizable. The fields which could be localizable and able to be provided in the translations are:

*   `dir`
*   `name`
*   `short_name`
*   `description`
*   `icons`
*   `screenshots`
*   `shortcuts`

For complex fields such as `shortcuts` where only a subset of its fields can be translated (e.g. `name` but not `url`), only the fields to be translated should be redefined in `translations`. The ordering of the shortcuts will be used to match it’s translation. Therefore the number of shortcuts defined in the translations should match the number defined at the top level.


```
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

## Possible extensions

In addition to the language string mapping to an object containing the translations as proposed, it could map to a string that defines the location of a separate JSON file containing the translations.

```
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

```
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

```
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
  "icons": ...
}
```

This approach is less readable for complex fields such as shortcuts, for little/no benefit over the proposed option.
