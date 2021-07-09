# Web App Launch Handling

Authors: alancutter@, mgiuca@, dominickn@
Last updated: 2021-07-09


## Overview

This document describes a new `launch_handler` manifest member that enables
web apps to customise their launch behaviour across all types of app launch
triggers.

We found that almost all "launch" use cases could be covered by a select few
fixed rules (for example, "choose an existing window in the same app, focus it,
and navigate it to the launched URL"). This `launch_handler` proposal enables
sites to specify a set of fixed rules without having to implement custom
[service worker `launch`][sw-launch-explainer] event logic, which should satisfy
most use cases, and simplify the implementation in browsers and sites.


## Use Cases

- Single-window web apps: a web app that prefers to only have a single instance
  of itself open at any time, with new navigations focusing the existing
  instance.\
  Sub-use cases include:
  - Apps that generally only make sense to have one instance running
    (e.g., a music player, a game).
  - Apps that include multi-document management within a single instance
    (e.g., an HTML-implemented tab strip, floating sub-windows like Gmail).


## Non-goals

- Forcing a web app to only ever appear in a single client (e.g. blocking being
  opening in a browser tab when open in an app window).
- Configuring whether link navigations into the scope of a web app launch the
  web app (this is out of scope and may be handled by a future version of the
  [Declarative Link Capturing][dlc-explainer] spec).
- Multi-document-instance web apps: a web app that opens documents in their own
  instances but wishes to refocus an already open document instead of opening
  duplicate instances for it. This would instead by handled by the [service
  worker `launch` event][sw-launch-explainer].



## Background

- There are several ways for a web app window to be opened:
  - [File handling](https://github.com/WICG/file-handling/blob/main/explainer.md)
  - [In scope link capturing](https://github.com/WICG/sw-launch/blob/master/declarative_link_capturing.md)
  - [Note taking](https://wicg.github.io/manifest-incubations/index.html#note_taking-member)
  - Platform specific app launch surface
  - [Protocol handling](https://github.com/MicrosoftEdge/MSEdgeExplainers/blob/main/URLProtocolHandler/explainer.md)
  - [Share target](https://w3c.github.io/web-share-target/)
  - [Shortcuts](https://www.w3.org/TR/appmanifest/#dfn-shortcuts)

  Web apps launched via these triggers will open in a new or existing app window
  depending on the user agent platform. There is currently no mechanism for the
  web app to configure this behaviour.

- This is a refactor of the [Declarative Link Capturing][dlc-explainer]
  explainer with a reduced behaviour scope. This does not prescribe link
  capturing as a means of launching a web app, instead it describes how the
  launch of a web app (via link capturing or any other means) may be configured
  by the manifest and service worker.


## Proposal

- Add a `launch_handler` member to the web app manifest specifying the default
  client selection and navigation behaviour for web app launches.
  The shape of this member is as follows:
  ```
  "launch_handler": {
    "route_to": "auto" | "new-client" | "existing-client",
    "navigate_client": "auto" | true | false
  }
  ```

  If unspecified then `launch_handler` defaults to
  `{"route_to": "auto", "navigate_client": "auto"}` whereby the behaviour
  is up to the user agent.

  Both `route_to` and `navigate_client` also accept a list of values, the
  first valid value will be used.

  Example manifest that choses to receive all app launches as events in existing
  web app windows:
  ```json
  {
    "name": "Example app",
    "start_url": "/index.html",
    "launch_handler": {
      "route_to": "existing",
      "navigate_client": false
    }
  }
  ```

- Enqueue a [`LaunchParams`](
  https://github.com/WICG/file-handling/blob/main/explainer.md#launch)
  object in the DOM Window's `launchQueue` of the chosen client **for every web
  app launch**.

- Add a `url` field to `LaunchParams` and set it to the URL being launched if
  the chosen launch client is not navigated as part of the launch.

  Other web app APIs that involve launching may extend the `LaunchParams` with
  more data specific to the method of launching e.g. a [share target](
  https://w3c.github.io/web-share-target/) payload.


## Future extensions to this proposal

- Add a service worker `"launch"` event that intercepts every web app launch.
  The service worker can choose to override the value of `route_to`,
  `navigate_client` and `LaunchParams`.

  **Use case:** Opening a productivity web app via a
  [file handler](https://github.com/WICG/file-handling/blob/main/explainer.md)
  causes an existing window that already had the file open to come into focus
  instead of launching a duplicate window.

- Add a `new_client_url` member to the web app manifest. All new clients that
  don't navigate to the launch URL will open `new_client_url` instead.

  If `new_client_url` is the default value `"auto"` then behave as if it is set to
  the value of `start_url`.

  **Use case:** Web apps that have heavy `start_url` pages and want to avoid
  loading unnecessary resources.

- Add the `launch_handler` field to other launch methods to allow sites to
  customise the launch behaviour for specfic launch methods. E.g.:
  ```json
  {
    "name": "Example app",
    "description": "This app will navigate existing clients unless it was launched via the share target API.",
    "launch_handler": {
      "route_to": "existing",
      "navigate_client": true
    },
    "share_target": {
      "action": "share.html",
      "params": {
        "title": "name",
        "text": "description",
        "url": "link"
      },
      "launch_handler": {
        "navigate_client": false
      }
    }
  }
  ```


## Related proposals


### [Service Worker launch event][sw-launch-explainer]

This proposal is declarative alternative to the [service worker `launch`](
sw-launch-explainer) proposal in WICG. It covers many of the same
use cases, but omits the more advanced use cases (specifically, the option to
choose which client to focus).

Use of `launch_handler` in the manifest would provide a "default" launch
behaviour that the service worker `launch` event handler can choose to override.


### [Declarative Link Capturing][dlc-explainer]

This `launch_handler` proposal is intended to be a successor to the "launch"
components of DLC and decouple launch configuration from "link capturing"
behaviour.

`launch_handler` generalises the concept of a launch into two core primitive
actions; launch client selection and navigation, and explicitly decouples them
from the "link capturing" launch trigger.


### [WICG: URL Handlers](https://github.com/WICG/pwa-url-handler/blob/master/explainer.md)

Similarly to Declarative Link Capturing this `launch_handler` proposal refactors
out the "launch" component from the URL Handler proposal. `launch_handler`
behaviour is intended for being "invoked" by URL Handlers at the point in which
a web app has been chosen to handle an out-of-browser link navigation.


### [File Handling](https://github.com/WICG/file-handling/blob/main/explainer.md)

This proposal takes the `LaunchQueue` and `LaunchParams` ideas from the File
Handling proposal and extends them slightly. Instead of enqueuing `LaunchParams`
for specific types of launches they will be enqueued for every type of web app
launch.


### [Tabbed Application Mode](https://github.com/w3c/manifest/issues/737)

This proposal is intended to work in tandem with tabbed mode web app windows.
The behaviour of `"route_to": "new-client"` with an already open tabbed window
is to open a new tab in that window.


[sw-launch-explainer]: https://github.com/WICG/sw-launch/blob/main/explainer.md
[dlc-explainer]: https://github.com/WICG/sw-launch/blob/main/declarative_link_capturing.md