# Client Hint for Display Mode

Author: [Aaron Gustafson](https://github.com/aarongustafson)

PR: https://github.com/w3c/manifest/pull/977

## Overview

Developers often need to know details about how their web app is being rendered in order to make content-negotiation decisions. Though this information is accessible via JavaScript, when sent as part of a Request, servers are empowered to do the content-negotiation earlier in the process of setting up the web app, before JavaScript execution takes place.

## Proposal

To provide `display_mode` information, this document suggests adding a `Sec-CH-Display-Mode` Client Hints Token. The value for the `Sec-CH-Display-Mode` header must be a valid `display_mode` value that represents the current display mode of the web application, reflective of both the authored `display_mode` and `display_override` values.

### Example

```http
Sec-CH-Display-Mode: standalone
```

## Security and Privacy Considerations

There are minimal security and privacy concerns with this proposal. This will allow sites to know the current display_mode being used for the PWA, however this information does not constitute PII and it is already exposed through CSS and JavaScript.

## Alternatives considered

Alternative header names were also considered.

## Open questions

1. 