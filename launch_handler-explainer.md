# Web App Launch Handling

## Overview

This document describes a new `launch_handler` manifest member that enables
web apps to customise their launch behaviour across all types of app launch
triggers.


## Use Cases

- Web apps that are designed to be used in a single window e.g. a music app.

- Web app that capture and handle share target events and user navigations
  in existing windows without invoking a navigation and losing existing state.
  E.g. sharing an image to a chat web app could open a contact picker overlayed
  on top existing chat content.


## Background

There are several ways for a web app window to be opened:
- Platform specific app launch surface
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

- Add a `launch_handler` member to the web app manifest specifying the default
  client selection and navigation behaviour for web app launches.
  The shape of this member is as follows:
  ```
  "launch_handler": {
    "route_to_client": null | "new" | "existing",
    "navigate_client": null | true | false
  }
  ```

  If unspecified then `launch_handler` defaults to
  `{"route_to_client": null, "navigate_client": null}` whereby the behaviour
  is up to the user agent.

  Both `route_to_client` and `navigate_client` also accept a list of values, the
  first valid value will be used.

  Example manifest that choses to receive all app launches as LaunchQueue events
  in existing windows:
  ```json
  {
    "name": "Example app",
    "start_url": "/index.html",
    "launch_handler": {
      "route_to_client": "existing",
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
  The service worker can choose to override the value of `route_to_client`,
  `navigate_client` and `LaunchParams`.

  **Use case:** Opening a productivity web app via a
  [file handler](https://github.com/WICG/file-handling/blob/main/explainer.md)
  causes an existing window that already had the file open to come into focus
  instead of launching a duplicate window.

- Add a `new_client_url` member to the web app manifest. All new clients that
  don't navigate to the launch URL will open `new_client_url` instead.

  If `new_client_url` is the default value `null` then behave as if it is set to
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
      "route_to_client": "existing",
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


### [Declarative Link Capturing](https://github.com/WICG/sw-launch/blob/main/declarative_link_capturing.md)

`launch_handler` generalises the concept of `capture_links` into two core
primitive actions (launch client selection and navigation) and more explicitly
decouples them from the specific "link capturing" launch trigger.


### [File Handling](https://github.com/WICG/file-handling/blob/main/explainer.md)

This proposal takes the `LaunchQueue` and `LaunchParams` ideas from the File
Handling proposal and extends them slightly. Instead of enqueuing `LaunchParams`
for specific types of launches they will be enqueued for every type of web app
launch.