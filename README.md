
# Specification 'manifest-incubations'

Specification link: https://wicg.github.io/manifest-incubations/index.html

# Explainers

 - [Scope Extensions for Web Apps](scope_extensions-explainer.md)
 - [Web App Translations](translations-explainer.md)

# Background

There are a number of [Web App Manifest](https://www.w3.org/TR/appmanifest/) features that are either not mature enough to go into the main W3C spec, or that do not belong there because they do not have multiple independent implementations.

- [`BeforeInstallPrompt`](https://github.com/w3c/manifest/pull/836)
- [`display_override`](https://github.com/w3c/manifest/pull/932) (unless that PR can land in W3C Manifest)
- [Tabbed mode](https://github.com/w3c/manifest/issues/737)
- [Protocol handlers](https://github.com/w3c/manifest/issues/846)
- [Declarative link capturing](https://github.com/WICG/sw-launch/blob/master/declarative_link_capturing.md)

Rather than creating a separate WICG repo for each of these (given that there are often interdependencies between them), we are creating a "manifest-incubations" spec with all of them in a single document. That means readers only need to keep track of two documents, instead of ~10.

Other related incubations that _do_ have their own WICG repos:

- [URL handlers](https://github.com/WICG/pwa-url-handler/)
- [File handlers](https://github.com/WICG/file-handling/)
- [Window controls overlay](https://github.com/WICG/window-controls-overlay/)

Despite this, it may make sense for their spec text to live inside `manifest-incubations` since they may have dependencies on other things here (e.g., `display_override`).

## Contact

* Daniel Murphy \<dmurph@chromium.org\>
* Matt Giuca \<mgiuca@chromium.org\>
