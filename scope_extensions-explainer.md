# Extended scopes for Web Apps

## Overview

This document describes a new `scope_extensions` manifest member that enables
web apps to extend their
[scope](https://www.w3.org/TR/appmanifest/#understanding-scope) to other
origins.

## Use Cases / Goals

- Allow sites that control multiple subdomains and top level domains to behave
  as one contiguous web app.\
  E.g. a site may span `example.com`, `example.co.uk` and `support.example.com`.

- Allow web apps to capture user navigations to sites they are affiliated with.\
  E.g. "News Aggregator App" capturing links navigations to examplenewssite.com.

## Background

Web app scope (defined by the `scope` field) is currently used for:
1. Constraining urls like start_url, file handlers, share target.
1. Defining the set of link navigations that can be captured by the app.
1. Determining whether an app window's root document has left the app's scope
   (possibly invoking window UI informing the user of this).

The `scope_extensions` mechanism can expand all these behaviours to include
other origins given agreement between the web app's primary origin and the
associated origins.

## Proposal

1. Add a `scope_extensions` member to the web app manifest specifying a list of
   origin patterns to associate with.

   Example manifest located at `https://example.com/manifest.webmanifest`:
   ```
   {
     "id": "",
     "name": "Example",
     "display": "standalone",
     "start_url": "/index.html",
     "scope_extensions": [
       {"origin": "*.example.com"},
       {"origin": "example.co.uk"},
       {"origin": "*.example.co.uk"}
     ]
   }
   ```
   In this example the Example app is extending its app scope to all its
   subdomains along with its .co.uk site and its subdomains.

1. Specify a `web-app-origin-association.json` file format that must be located
   at `https://<associated origin>/.well-known/web-app-origin-association.json`
   on the associated origin's domain. This specifies the set of web apps that
   may include it as a scope extension keyed on each web app's identifier.

   Example association file located at
   `https://example.co.uk/.well-known/web-app-origin-association.json`:
   ```
   {
     "web_apps": {
        "https://example.com/": {
          "include_paths": ["/*"],
          "permissions": ["intercept-links"]
        },
        "https://associated.site.com/": {
          "include_paths": ["/*"],
          "exclude_paths": ["/settings/*"]
        }
      }
    }
   ```

1. Let the extended scope be the set of URLs that:
    - Has an origin that matches one of the origin patterns in the manifest's
      `scope_extensions` list.
    - Has an origin with a valid
      `<origin>/.well-known/web-app-origin-association.json` association file
      with an association entry keyed by the web app's identifier.
    - Matches an `include_paths` entry in the association entry.
    - Does not match as `exclude_paths` entry in the association entry.

1. Replace the constraint on manifest URLs that are bound by scope to instead be
   bound by the extended scope with one exception; `start_url` remains bound by
   the `scope` member.

## Security Considerations

### Declarative link capturing events

Capturing user navigations via `"capture_links": "existing-client-event"` has
the potential for the web app to spoof its associated origins. Event link
capturing must not be supported for associated origins unless they specify
`"permissions": ["intercept-links"]` in their entry for the associated web app.
This permission opt-in is used as a signal of trust between the associated
origin and the web app.


## Related Proposals

- [URL Handlers](https://github.com/WICG/pwa-url-handler/blob/main/explainer.md)
- [Declarative Link Capturing](https://github.com/WICG/sw-launch/blob/main/declarative_link_capturing.md)
- [PWA Unique ID](https://github.com/philloooo/pwa-unique-id/blob/main/explainer.md)
