# Self-Review Questionnaire: Security and Privacy

Security and Privacy questionnaire for [`scope_extensions`](https://github.com/WICG/manifest-incubations/blob/gh-pages/scope_extensions-explainer.md)

## What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?

The feature does not expose any information to web sites or other parties.

## Do features in your specification expose the minimum amount of information necessary to enable their intended uses?

Yes.

## How do the features in your specification deal with personal information, personally-identifiable information (PII), or information derived from them?

There is no information of this type being handled by the feature.

## How do the features in your specification deal with sensitive information?

There is no sensitive information handled by the feature. 

## Do the features in your specification introduce new state for an origin that persists across browsing sessions?

No, this feature does not introduce new states tied to an origin. The validation that an origin is within an extended scope is done at the same cadence as manifest updates. A validated status of an origin might persist across sessions.

## Do the features in your specification expose information about the underlying platform to origins?

No.

## Does this specification allow an origin to send data to the underlying platform?

No.

## Do features in this specification enable access to device sensors?

No.

## Do features in this specification enable new script execution/loading mechanisms?

No.

## Do features in this specification allow an origin to access other devices?

No.

## Do features in this specification allow an origin some measure of control over a user agent’s native UI?

Yes. The feature will affect the UA's out-of-scope banner on installed web applications. This UI will still appear if the PWA navigates to a page that is not in the (extended) scope.

## What temporary identifiers do the features in this specification create or expose to the web?

None.

## How does this specification distinguish between behavior in first-party and third-party contexts?

No, it does not make such distinction.

## How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?

It will not be available in Private Browsing/Incognito mode.

## Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

There is no specification written yet. (The [explainer](https://github.com/WICG/manifest-incubations/blob/gh-pages/scope_extensions-explainer.md) does have Security considerations.)

## Do features in your specification enable origins to downgrade default security protections?

This feature makes a distinction between origins in an app's extended scope versus unrelated origins. Origins in the extended scope will not trigger the out-of-scope banner while unrelated origins will. All cross-origin navigation will be higlighted by origin text appearing in the window's title bar. 

## How does your feature handle non-"fully active" documents?

This feature does not interact with non-"fully active" documents.

## What should this questionnaire have asked?

N/A.
