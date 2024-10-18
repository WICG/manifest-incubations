# Scope Extensions for Web Apps

## Abstract

This document proposes a new `scope_extensions` [Web Application
Manifest](https://www.w3.org/TR/appmanifest/) member that extends the concept of
[app scope](https://www.w3.org/TR/appmanifest/#understanding-scope). User agents
apply the manifest to documents that are [within
scope](https://www.w3.org/TR/appmanifest/#dfn-within-scope) and often apply
different UX treatments when visiting documents outside of scope. According to
existing specification, the scope of an app can be made no broader than a single
[web origin](https://datatracker.ietf.org/doc/rfc6454/). Apps that wish to
define a scope spanning multiple origins could do so using the proposed
`scope_extensions` member.

## Introduction

In Chromium browsers, top-level app window navigation to an out-of-scope URL
creates a notification bar - this notification bar informs the user that they
have navigated outside of app scope or to a different origin altogether, and
provides a control to return to the `start_url`. Other browser implementations
may display similar behavior.

<figure>
    <img src="images/out-of-scope-ux.png" width="600" alt="">
    <figcaption>Example: installed web app window with out-of-scope bar UI</figcaption>
</figure>

The app window above has navigated to a url (`https://airhorner.com`) outside of
its scope (`https://diek.us/pwinter/`). The white bar above the web contents
informs the user of this difference. This feature aims to keep the user aware of
the content's security context.

A developer may intentionally want to include documents from different origins
in app scope. In this case, the out-of-scope bar is not necessary and could
confuse users. Furthermore, as app scope is used as the boundary for the
application of manifest features such as `theme_color`, documents intended to be
part of the app but are excluded from app scope are not presented in a
consistent manner. 

## Motivating use cases 

* Multiple TLDs

* Multiple subdomains (slack, zoom, etc)

* Link capturing
<!-- TODO -->

For example, an application might host content that is located
in one specific origin, and rely on a login page that is out of the scope of the
application itself to access that content. In other cases, the same application
might be associated to multiple Top Level Domains, that might respond to
different locales (`app.com`, `app.co.uk`, `app.co.cr`, etc).


## Goals

- Allow sites that control multiple subdomains and top level domains to behave
  as one contiguous web app.\
  E.g. a site may span `example.com`, `example.co.uk` and `support.example.com`.

- Allow web apps to capture user navigations to sites they are affiliated with.\
  E.g. "News Aggregator App" capturing links navigations to examplenewssite.com.

## Non-goals

<!-- TODO -->

## Proposal

`scope_extensions` works by ...
<!-- TODO -->

A new field `scope_extensions` in the manifest declares which origins or sites should
join the app scope. 

To extend the app scope, a developer will need to modify the web app manifest, host
one or more association files, and may also have to add a new response header with
returned documents.

A two-way handshake between the app and the extension origin/site is established
when the origin/site hosts a `.well-known/web-app-origin-association` file. This file
declares which apps are allowed to incorporate itself and which resources can
participate.

<!-- TODO: Add graphic illustrating the 3 components. -->

### Manifest
Use `scope_extensions` member in the web app manifest to declare origins in the
the extended app scope. 

<!-- TODO each origin has `scope` parameter as well. -->

Example manifest located at `https://example.com/manifest.webmanifest`:
   ```json
   {
     "id": "/",
     "name": "Example App",
     "display": "standalone",
     "start_url": "/app/index.html",
     "scope": "/app",
     "scope_extensions": [
       { "type": "site", "value": "https://example.com" },
       { "type": "site", "value": "https://example.co.uk" },
       { "type": "origin", "value": "https://helpcenter.example-help-center.com" }
     ]
   }
   ```
The "Example" app has a regular scope of `http://example.com/app` and is
extending its app scope to the sites `https://example.com`,
`https://example.co.uk`, and the single origin 
`https://helpcenter.example-help-center.com`. 

Each object in `scope_extensions` must contain both `type` and `value` string
fields. The value of `type` must be one of `["origin" | "site"]`. The `value`
field must contain a valid URL. 

An `"origin"` extension adds that specific web origin to the app scope, while a
`"site"` extension adds all origins that passes the
[same-site](https://html.spec.whatwg.org/multipage/browsers.html#same-site) test with the
specified host. 

| Type   | Behavior |
|--------|----------|
| `origin` | The URL in `value` is converted to an [origin](https://html.spec.whatwg.org/multipage/browsers.html#concept-origin-tuple). All allowed paths of that origin is added to the extended scope.|
| `site` | The URL in `value` is converted to a [host](https://url.spec.whatwg.org/#hosts-(domains-and-ip-addresses)). All allowed paths of all origins that pass the [same-site test](https://html.spec.whatwg.org/multipage/browsers.html#obtain-a-site) as the host are added to the extended scope.|

This format is both backward and forward compatibility. Entries with types not recognized
by a user agent should be ignored.

### Association file

Create a `web-app-origin-association` file that must be downloadable at
`https://<associated origin/site>/.well-known/web-app-origin-association`. 
This specifies a list of web apps that may include it as a scope extension.

Origins/sites have the option of filtering what resources are allowed to join 
the app scope by configuring filters in their 
`.well-known/web-app-origin-association` file. A `scope` filter will allow paths
to be included, working the same way as the manifest `scope` field. Other filtering
formats can be added in the future.

Example association file located at
   `https://example.co.uk/.well-known/web-app-origin-association`:
   ```json
   {
     "web_apps": [{
       "web_app_identity": "https://example.com/", 
       "scope": "/app"
     }, {
       "web_app_identity": "https://associated.site.com/",
     }]
   }
   ```

The `web_app_identity` field must contain a valid [web application
id](https://w3c.github.io/manifest/#id-member).

### Resulting extended scope
A URL is in the extended scope of a web app if both:
  - It matches an entry in the `scope_extensions` list.
  - That entry has been validated by fetching the
    `<origin>/.well-known/web-app-origin-association` association file with an
    entry matching the app's
    [identity](https://w3c.github.io/manifest/#id-member).

### Response header
For `origin` extensions, a `web-app-origin-association` file from the extended
origin is sufficient to complete the two-way handshake. 

For `site` extensions, a `web-app-origin-association` is necessary but insufficient
as it cannot represent every origin that passes the same-site test with the site.
For this reason, documents from the site will need to be accompanied by a 
`App-Scope-Extension-Allow-Id` response header. 

Example header: 
`App-Scope-Extension-Allow-Id: https://myapp.com/index.html, https://otherapp.com/index.html`

This response header represents the specific origin and validates that it
agrees to be incorporated into apps that match a list of manifest IDs.
 

## Security Considerations

### Link capturing from another origin

If an origin A adds a web app B to its `web-app-origin-association` file, A is
implicitly authorizing app B to intercept navigations to URLs in A. This implies
that app B can potentially spoof origin A and therefore it is advised that
origin A and web app B should be owned by the same entity.

User agents may perform link capturing for user navigations within a web app's
extended scope and launch the web app instead of performing the navigation.

The [launch handler][launch-handler] proposal enables sites to reroute app
launches into existing web app contexts.

The combination of link capturing, launch handler and scope extensions leads to
the following attack vector:
1. User installs the TestApp web app from app.com.
1. TestApp's scope includes site.com with valid origin associations.
1. TestApp sets its `launch_handler` to
   ```
   {
     "client_mode": "focus-existing"
   }
   ```
1. User clicks on a link to site.com.
1. Navigation is captured by an existing TestApp window that is brought into
   focus and has a LaunchParam is enqueued.
1. *TestApp is now aware that the user is navigating to site.com and could
   perform a fake navigation with the intention of duping the user into thinking
   they're on site.com.*

## Future extensions

- More specific scoping e.g. scope suffix or include/exclude lists or [URL
  patterns](https://wicg.github.io/urlpattern/).
  - To be able to apply these more specific scoping rules to the primary scope
    (including exclusion). One possible approach is to have the primary origin
    specified in the `scope_extensions` list and have it override the behaviour
    of `scope`.
- Replace the constraint on manifest URLs that are bound by scope (except for
  `start_url`) to instead be bound by the extended scope. Validation of the
  associated origins is not required for these URLs to be part of a valid
  manifest. Prior to validation the URLs must be treated as if they were not
  specified.
- Add an `"authorize"` field to `web-app-origin-association` e.g.:
  ```json
  {
    "web_apps": [{
      "web_app_identity": "https://example.org",
      "authorize": ["intercept-links"]
    }]
  }
  ```
  This opt-in serves as a signal of trust from the associated origin to allow
  the web app to [capture navigations][link-capturing-from-another-origin] into
  the associated origin.

### Extended Scope Permissions

When an application uses `scope_extensions` to expand its scope, **each
additional scope's permissions remain the same**. Expanding scopes does not
imply any change in permissions. The only thing that changes after being
included in a scope is that the security UX will not appear when an app
navigates to content served from those scopes.

### Additional security UX

For added security when in the installed web application, the app might display
UX that always displays the current scope that is being served, along with
privacy and permission settings of that specific scope.

![Current scope being displayed on the privacy menu of installed web
app](images/scope-privacy-info.png)

In the previous image, a user can always see which domain is serving content
under the privacy menu. 

## Alternatives considered

<!-- TODO -->

## Related Proposals

### [URL Handlers][url-handlers]

The Scope Extensions proposal is intended to be a replacement for the [URL
Handlers][url-handlers] proposal with the following changes:
 - Re-orient the goal to be focused just on expanding the set of origins/URLs in
   the web app's scope. Remove the goal of registering web apps as URL handlers
   in the user's operating system. That behaviour will be covered by individual
   browsers optionally offering users the choice to capture link navigations as
   web app launches.
 - Rename the new manifest field from `url_handlers` to `scope_extensions` to
   reflect the change in goals.
 - Move the association file from "<origin>/web-app-origin-association.json" to
   "<origin>/.well-known/web-app-origin-association". This better conforms with
   [RFC 8615](https://datatracker.ietf.org/doc/html/rfc8615).
 - Change the association file entries to be keyed on the [web app
   identifier](manifest-identity) rather than the web app's manifest URL (the
   former having been added to the Manifest spec in the interim).
 - Rename `"paths"` to `"include_paths"` in the association file entries.
 - Add an "authorize" field to the association file entries for the associated
   origin to provide explicit opt-in signals for security sensitive
   capabilities.


[launch-handler]: https://github.com/WICG/sw-launch/blob/main/launch_handler.md
[url-handlers]: https://github.com/WICG/pwa-url-handler/blob/main/explainer.md
[manifest-identity]: https://w3c.github.io/manifest/#dfn-identity
