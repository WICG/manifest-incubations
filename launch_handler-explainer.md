# Web App Launch Handling

## Overview

This document describes a new `launch_handler` manifest member that enables
web apps to customise their launch behaviour across mulitple launch triggers.


## Use Cases

- Web apps that are designed to be used in a single window e.g. a music app.

- Web app that capture and handle share target events and user navigations
  in existing windows without invoking a navigation and losing existing state.\
  E.g. sharing an image to a chat web app could open a picker overlayed on top
  of an existing conversation to select which contact to send it to .

- Opening a productivity web app via a
  [file handler](https://github.com/WICG/file-handling/blob/main/explainer.md)
  causes an existing window that already had the file open to come into focus
  instead of launching a duplicate window.


## Background

There are several ways for a web app window to be opened:
- Operating system app list
- [File handling](https://github.com/WICG/file-handling/blob/main/explainer.md)
- [Share target](https://w3c.github.io/web-share-target/)
- [In scope link capturing](https://github.com/WICG/sw-launch/blob/master/declarative_link_capturing.md)
- [Shortcuts](https://www.w3.org/TR/appmanifest/#dfn-shortcuts)
- [Note taking](https://wicg.github.io/manifest-incubations/index.html#note_taking-member)
- [Protocol handling](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/URLProtocolHandler/explainer.md)

Web apps launched via these triggers will open in a new or existing app window
depending on the user agent platform. There is currently no mechanism for the
web app to configure this behaviour.


## Proposal

1. Add a `launch_handler` member to the web app manifest specifying .

   Example manifest located at `https://example.com/manifest.webmanifest`:
   ```json
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
   In this example the "Example" app is extending its app scope to all its
   subdomains along with its `.co.uk` site and its subdomains.



## Security Considerations

### Declarative link capturing events

Capturing user navigations via `"capture_links": "existing-client-event"` has
the potential for the web app to spoof its associated origins. Event link
capturing must not be supported for associated origins unless they specify
`"authorize": ["intercept-links"]` in their entry for the associated web app.
This opt-in is used as a signal of trust between the associated origin and the
web app.


## Related Proposals

### [URL Handlers](https://github.com/WICG/pwa-url-handler/blob/main/explainer.md)

The Scope Extensions proposal is intended to be a replacement for the
[URL Handlers](https://github.com/WICG/pwa-url-handler/blob/main/explainer.md)
proposal with the following changes:
 - Re-orient the goal to be focused just on expanding the set of origins/URLs in
   the web app's scope. Remove the goal of registering web apps as URL handlers
   in the user's operating system. That behaviour will be covered by the
   [Declarative Link Capturing](https://github.com/WICG/sw-launch/blob/main/declarative_link_capturing.md)
   proposal instead.
 - Rename the new manifest field from `url_handlers` to `scope_extensions` to
   reflect the change in goals.
 - Move the association file from "<origin>/web-app-origin-association.json" to
   "<origin>/.well-known/web-app-origin-association.json". This better conforms
   with [RFC 8615](https://datatracker.ietf.org/doc/html/rfc8615).
 - Change the association file entries to be keyed on the web app identifier
   rather than the web app's manifest URL. This aligns with the recent
   [PWA Unique ID](https://github.com/philloooo/pwa-unique-id/blob/main/explainer.md)
   proposal.
 - Rename `"paths"` to `"include_paths"` in the association file entries.
 - Add an "authorize" field to the association file entries for the associated
   origin to provide explicit opt-in signals for security sensitive
   capabilities.

### [Declarative Link Capturing](https://github.com/WICG/sw-launch/blob/main/declarative_link_capturing.md)

Scope extensions can be considered the first stage in the link capturing
pipeline. This proposal allows developers to control the set of user navigation
URLs that the web app is intended to capture. The
[Declarative Link Capturing](https://github.com/WICG/sw-launch/blob/main/declarative_link_capturing.md)
proposal allows developers to control the action that is taken once a user
navigation is captured e.g. open a new app context or navigate an existing one.