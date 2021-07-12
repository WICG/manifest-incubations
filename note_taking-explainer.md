# Note Taking Web Apps

## Overview

Note-taking apps have special integrations in some user agents and OSs to take a
quick note. Examples:

- [keyboard shortcut on Windows](https://support.microsoft.com/en-us/office/create-quick-notes-0f126c7d-1e62-483a-b027-9c31c78dad99)
- [action center button on Windows](https://www.windowscentral.com/how-change-note-button-action-open-other-note-taking-apps-windows-10)
- [stylus tools button on CrOS](https://support.google.com/chromebook/answer/7073299)
- [Google Assistant on Android](https://support.google.com/assistant/answer/9053424)
- [touch bar or Siri on OSX](https://support.apple.com/en-au/guide/notes/not9474646a9/mac)

Such integrations could begin to be supported for web apps too, if they had a
way to identify themselves as note-taking apps and a URL to launch for taking
new note. It wouldn't cover all the use-cases linked above (eg. voice assistant
actually adding items to the note) but it covers the core flow of adding a new
note.

### Background

Other web app APIs set a precedent of using a URL declared in the web app
manifest to advertise capabilities and facilitate integrations with the OS:

* [Web Share Target](https://w3c.github.io/web-share-target/)
* [File Handling](https://github.com/WICG/file-handling/blob/main/explainer.md)
* [Protocol Handlers](https://web.dev/url-protocol-handler/)

## Proposal

This explainer proposes a new dictionary manifest entry `note_taking` with an
entry `new_note_url` to specify a URL, within app scope, to launch for taking a
new note.

```
{
  "name": "My note taking app",
  "start_url": "/index.html",
  ... other required manifest fields ...
  "note_taking": { "new_note_url": "/new_note.html" },
}
```

The presence of the `note_taking` member signals intent to act as a note-taking
app, and `new_note_url` is a simple note-taking app capability. The url is
launched in the same way as the `start_url`, so is affected by the app's other
properties, such as
[display mode](https://www.w3.org/TR/appmanifest/#display-member) and
[link capturing](
https://github.com/WICG/sw-launch/blob/main/declarative_link_capturing.md).

By using a dictionary we can easily add other note-taking app functionality or
extend the current functionality, and it keeps the manifest organised with
related functionality together. It also means note-taking can be specified
largely orthogonally to other manifest specification changes, which helps to
keep the manifest specification modular and manageable.

### Future Considerations

If the need arises in the future to add more note-taking functionality, this can
easily be added to the `note_taking` member. For example (**not
currently proposed**), if we want to be able to populate a note with contents
from another app or add items to a list from voice input, we could add:

```
{
  ... other manifest fields ...
  "note_taking": {
    "new_note_with_contents": {
      "url": "/POST_url/add_note.html",
      "params": {
        "title": "name",
        "text":  "description",
        "url":   "link"
      }
    },
    "add_item_to_note": {
      "action": "/POST_url/add_item_to_note.html",
      "params": {
        "existing_note_id": "id",
        "item_text":  "description",
      }
    }
  }
}
```

## Alternatives Considered

A common theme of alternatives is to make a generalised interface for web apps
to advertise their capabilities. Ideally this would let us solve a whole class
of problems instead of just one case. For example, a similar mechanism to "
create a new note" could be used to "create new calendar entry" or "add a
contact".

### Extend the `Shortcuts` Member

If we used the existing [Shortcuts manifest member](
https://www.w3.org/TR/appmanifest/#shortcuts-member), we would need to add a way
for the user agent to identify shortcuts as intended to be used for specific
purposes/tasks (eg. a new optional "purpose" field on each shortcut entry, with
a defined set of values like "new-note", "new-contact", etc). Some of these
future tasks might need additional parameters/context, which would further
extend and complicate the `Shortcuts` member. Additionally, Shortcuts are
intended to be displayed to the user, while many tasks/capabilities should not
be displayed in a shortcuts menu, eg. any parameterised or contextual tasks,
non-task capabilities like [lock screen](https://github.com/WICG/lock-screen)
support. It seems like extending `Shortcuts` this far would be significantly
changing the semantics of the member.

### A General Capability Member

We also considered using a more general-purpose integrations field, where each
individual integration has a particular schema of fields. For example:

```
"integrations":  [
  { "task": "new-note", "url": <URL> }
  { "task": "add-to-existing-note", "url": <URL>, <additional params> }
  { "task": "lock-screen", "url": <URL> }
  { "task": "new-contact", "url": <URL>, <probably additional params> }
]
```

Or:

```
"integrations": {
  "note-taking": {
    "create": {
      "action": "/new_note.html",
    },
    "add-item": {
      "action": "/add_item_to_note.html",
      "params": {
        "title": "name",
        "text":  "description",
        "url":   "link"
      }
    }
  }
}
```

However, these individual integrations likely need to be specified individually
anyway, so we get little benefit from grouping like this. In fact there might be
a drawback to grouping disparate features and tying their specification together
rather than having separate tiny specifications. This "integrations" member is
really serving the same purpose as the root manifest object.

## Security and Privacy Considerations

This proposal would simply allow a web app to declare an additional URL (within
scope), which the user and/or user agent may choose to navigate to. So there are
minimal security & privacy effects: if the `new_note_url` is launched, then the
site can know that the URL was loaded, so it could be used for fingerprinting in
exactly the same way as the manifest `start_url` already allows. Launching
the `new_note_url` when creating a new note is identical to loading the URL
normally.
