# Self-Review Questionnaire: Security and Privacy

Security and Privacy questionnaire for [`update-token`](https://github.com/Dp-Goog/manifest-incubations/blob/gh-pages/predictable-app-updating.md)

## What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?

User agents will be able to know if a manifest update has been applied on an installed PWA or not.

## Do features in your specification expose the minimum amount of information necessary to enable their intended uses?

Yes.

## How do the features in your specification deal with personal information, personally-identifiable information (PII), or information derived from them?

There is no information of this type being handled by the feature.

## How do the features in your specification deal with sensitive information?

User agents will have the option of showing a UX dialog in case security sensitive fields of the app (`name`, `short_name` and `icons`) change.

## Do the features in your specification introduce new state for an origin that persists across browsing sessions?

No.

## Do the features in your specification expose information about the underlying platform to origins?

No.

## Does this specification allow an origin to send data to the underlying platform?

Yes, this will send the information that a site has changed its manifest, and the user agent can choose to apply that update if they want.

## Do features in this specification enable access to device sensors?

No.

## Do features in this specification enable new script execution/loading mechanisms?

No.

## Do features in this specification allow an origin to access other devices?

No.

## Do features in this specification allow an origin some measure of control over a user agent’s native UI?

Yes. User agents can create their own native UI based on whether the field in the manifest has a different value from what's stored or not.

## What temporary identifiers do the features in this specification create or expose to the web?

`update_token` could be used to distinguish between manifests being served to a single site.

## How does this specification distinguish between behavior in first-party and third-party contexts?

No, it does not make such distinction.

## How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?

It will not be available in Private Browsing/Incognito mode.

## Does this specification have both "Security Considerations" and "Privacy Considerations" sections?

There is no specification written yet.

## Do features in your specification enable origins to downgrade default security protections?

No.

## How does your feature handle non-"fully active" documents?

This feature does not interact with non-"fully active" documents.

## What should this questionnaire have asked?

N/A.
