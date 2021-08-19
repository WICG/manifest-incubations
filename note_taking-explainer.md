# Note Taking Web Apps

Author: Glen Robertson (glenrob@chromium.org)

## Overview

Note-taking apps have special integrations in some User Agents and Operating
Systems as a convenient way for the user to take a quick note. Some examples:

* ["Quick Note" keyboard shortcut on Windows](
  https://support.microsoft.com/en-us/office/create-quick-notes-0f126c7d-1e62-483a-b027-9c31c78dad99)
* ["Note" action center button on Windows](
  https://www.windowscentral.com/how-change-note-button-action-open-other-note-taking-apps-windows-10)
* ["New note" stylus tools button on CrOS](
  https://support.google.com/chromebook/answer/7073299)
* ["Take a note" in Google Assistant on Android](
  https://support.google.com/assistant/answer/9053424)
* ["New Note" touch bar button or "Create a note" with Siri on OSX](
  https://support.apple.com/en-au/guide/notes/not9474646a9/mac)

However, these integrations currently work only for native apps. Such
integrations could be supported for web apps too, if they had a way to identify
themselves as note-taking apps and a URL to launch for taking new note. It
wouldn't cover all of the use-cases linked above (eg. voice assistant actually
adding items to the note) but it covers the core & common flow of adding a new
note.

This explainer proposes a way for web apps to:
* declare themselves as note-taking apps
* declare a URL to launch for taking a new note

which will allow UAs and OSs to provide convenient integrations with note-taking
web apps, if they choose to do so.

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

The user agent can choose (or let the user choose) what to do with web apps
identified as note-taking apps. For example, it may surface the apps in its own
UI or integrate those apps with some OS-level functionality for taking notes. If
multiple note-taking apps are identified, it could integrate them all or choose
(or let the user choose) some subset to integrate.

By using a dictionary we can easily add other note-taking app functionality or
extend the current functionality, and it keeps the manifest organised with
related functionality together. It also means note-taking can be specified
largely orthogonally to other manifest specification changes, which helps to
keep the manifest specification modular and manageable.

By launching the `new_note_url` using the same launch mechanism as
the `start_url` apps can use [declarative link capturing](
https://github.com/WICG/sw-launch/blob/main/declarative_link_capturing.md) to
determine what happens when the URL is launched. For example, the app could
choose to receive the launch as an event in an existing app instance, instead of
opening a new instance of the app, and the app could handle the event by showing
a new note in the existing UI. This allows apps a lot of flexibility in
customising their behaviour in response to a launch, without complicating this
API.

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

### An Event

Another option would have been to have a note-taking app register an event
handler for a "new note" event. It could potentially have a dictionary of
parameters to allow a high degree of flexibility and added functionality in the
future.

However, such flexibility actually reduces the ability for the API to declare
capabilities. Unlike an event registration, the proposed New Note URL is known
as soon as the manifest is parsed, and relatively fixed, so it is clear to the
UA when a given app starts/stops self-identifying as a note-taking app capable
of the "new note" action. By contrast, an event handler might only be registered
in certain conditions and provides no assurance of being registered again in the
future. If new functionality is added to the New Note URL in the future, it
could be added with explicit parameters or another URL declared in the manifest,
which again acts as a declaration of capabilities. By contrast for an event with
a dictionary of parameters, the UA cannot know whether the app is capable of
handling new parameters. For example, if adding a text parameter to a "new note"
event's parameters, older apps may ignore this extra parameter and lose some
input text.

An event also complicates launching. Opening a URL is a natural way to launch a
web app from outside the app (the UA or OS), whereas an event requires that the
app is first launched or is already running. An event allows more customisation
of the app's launch behaviour, but similar customisation is possible with launch
URLs using [declarative link capturing](
https://github.com/WICG/sw-launch/blob/main/declarative_link_capturing.md).

## Security and Privacy Considerations

This proposal would simply allow a web app to declare an additional URL (within
scope), which the user and/or user agent may choose to navigate to. So there are
minimal security & privacy effects: if the `new_note_url` is launched, then the
site can know that the URL was loaded, so it could be used for fingerprinting in
exactly the same way as the manifest `start_url` already allows. Launching
the `new_note_url` when creating a new note is identical to loading the URL
normally.
