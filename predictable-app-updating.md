# Predictable Web App Updating \- Explainer

Author: Dibyajyoti Pal (dibyapal@chromium.org)

# **Introduction**

This explainer proposes a consistent way for developers to identify when to fully update their PWA in a safe and resourceful manner that is backwards compatible. This is done by updating the [manifest spec](https://www.w3.org/TR/appmanifest/#web-application-manifest) to have a specific field for making updates more deterministic and consistent. The proposal attempts to do so in a way that:

1. Specifies the update check algorithm to ensure the user agent can use minimal network resources.
2. Prevent user confusion by showing the update UX less often.

Doing so helps to further bridge the gap between PWAs and native apps.

# **Background**

[PWAs](https://web.dev/explore/progressive-web-apps) are app experiences built on the web, and like native apps, they support updating themselves. Developers have the option of changing any field in the manifest (including the [security sensitive fields](https://www.w3.org/TR/appmanifest/#dfn-security-sensitive-members)), and the user agent can then apply the changes in a way as defined in the [manifest updating spec](https://www.w3.org/TR/appmanifest/#updating). For example, changes to the security sensitive manifest properties **require** the user agent to explicitly get user permission, which is why Chromium shows an update dialog to the user. This helps prevent [phishing risks](#phishing).

Updates on PWAs are important because they allow:
- Rebranding via icon and name changes.
- Icon changes via changing icon urls in the manifest.
- Minor visual changes in the icon even if the url has stayed the same (due to dynamic re-encoding by CDNs).

gFor all these use-cases however, the **detection** of when an update should happen is not clearly defined in the spec, leading to [problems](#chromium-problems). The next session attempts to dive into these problems, and [propose](#proposal) a solution that all user agents can implement without running into the same problems.

# [**Chromium PWA update detection, and its problems**](#chromium-problems)

Currently, detecting that a PWA needs an update goes like this:

- On page load of an url within the [scope of an installed PWA](https://www.w3.org/TR/appmanifest/#scope-member), a manifest update is triggered. 
- The manifest, and the resources defined in it, are downloaded and compared with the installed PWA and its resources.  
- If there is a difference, an update happens.  
  - If the difference is in security sensitive fields, like the name, icon or short name, the user agent shows a UX notifying the user that an update is supposed to happen.  
  - The UX shows the differences between the old and the new sensitive fields, and asks the user to either accept the changes or uninstall the app.

![Update dialogs](./images/current-chromium-dialogs.png)

## Problem: Developers have no control over when the update dialog may show up

The dialog shows up whenever Chromium sees the new manifest & detects changes. Developers have to accept that every change to security sensitive members could trigger this. They cannot, for example, make a number of incremental changes, and then trigger one update dialog at the end once they are all done.

## Problem: Update check wastes bandwidth, requiring a throttle

Network resources are wasted by performing icon downloads over and over again just to see if an update is needed. Chromium's current implementation is forced to mitigate this problem by introducing a [throttle](https://web.dev/articles/manifest-updates) to reduce wasted downloads for non-updates. 

This also causes confusion among developers testing manifest updates for their sites, as updates are throttled to once per day. This required addition of a new [flag](https://web.dev/articles/manifest-updates#cr-desktop-test) to bypass the throttle.

## Problem: Basic icon diff triggers an update dialog too often 

Sometimes, icons would change in use-cases that wouldn’t really require an intrusive UX dialog to be shown to the user. Some use-cases where that would happen are:

- Developers making minor changes to icons which would not be visibly noticeable.  
- CDNs dynamically re-encoding icons.

Both these would trigger manifest updates, with the end user seeing no “visible” difference in the update UX, and would be confused about why this was popping up and interrupting their workflow. This led the behavior to be treated as spammy.

Chrome on Android solved it with a stop gap where PWA updates were automatic if the visual difference between the downloaded icon and the local icon was less than 10%.

# **Goals**

The current manifest update process gets the job done, but if it could be [specified properly](https://github.com/w3c/manifest/issues/384) to know when that should happen and be backwards compatible, it would certainly help developers. As such, the goals to solve are:

* [**Developer control**](#devc): Support developers to allow them to update their installed experiences:
  - Do not negatively affect developers who do not change their icon urls.
  - Empower developers to choose when to update their installed experiences.
* [**Preventing unnecessary user interruption**](#useint): Users should not see an update dialog more than necessary to confirm security-sensitive changes.
* [**Consistency**](#consistency): Provide a consistent way to detect when a manifest update should happen, which can be implemented across all user agents.
* [**User agent flexibility**](#uaf): It should be possible for user agents to use their judgement to block updates for known bad sites, allow known trusted apps to update without UX, or allow tiny visual changes to icons without requiring UX.
* [**Reduce network traffic**](#traffic): Unnecessary network traffic should be minimized.
* [**Prevent manifest id foot-gun**](#footgun): Encourage developers to set the manifest 'id' field, preventing a known [foot-gun](https://github.com/w3c/manifest/issues/1148).

# [**Proposal: Introduce 'update\_token', ignore icon changes by default**](#proposal)

The proposal is to introduce a new field in the manifest called `update_token`, whose existence (and non-existence) will be used to trigger manifest updating logic, based on the following guidelines:

* `update_token` is parsed if-and-only-if an 'id' field is set. This helps solve the goal of preventing the [`manifest id foot-gun`](#footgun).

This has already been discussed extensively in [TPAC 2022](https://www.w3.org/2022/09/13-webapps-minutes.html#t02) and [TPAC 2023](https://docs.google.com/document/d/1QDqllh8inOcIkTrblERm4HRKYh8Ce9Lu5S7WhOygs_0/edit?tab=t.0#heading=h.sg3uxnf61udd), with consensus received from developers and WebKit.

```

{
  "name": "The Best App",
  "id": "app",
  "start_url": ".",
  "display": "standalone",
   ....
   "update_token": "foo1"
}

```

## Pre-requisites:

- An app corresponding to the manifest has already been installed.

## Token based update detection

* If `seen_manifest.update_token` and is changed from the `saved_manifest.update_token,`then a manifest update is triggered.  
* If an `seen_manifest.update_token` is provided but is unchanged from the app’s `saved_manifest.update_token`, no update is triggered.  
  * This includes fields outside of `name, short_name` and `icons`, AKA fields that are not tied to the identity of the manifest and is thus not secure.

## Default (non-token-based) update detection

Without the presence of an `update_token` in the manifest, **ONLY** icon updates will be allowed if there are changes in the icon url specified in the manifest. This is similar to the behavior shown by [Cache-Control:Immutable](https://caniuse.com/mdn-http_headers_cache-control_immutable). The reason why the `Cache-Control` is not being used is the absense of backwards compatibility with existing PWAs on Android, where updates can only be triggered by changing the icon bitmap and not the url.

## Behavior

The below table introduces all possible combinations of behavior that can happen based on the status of an `existing update token` that is in the saved manifest vs a `new update token` that is newly seen in the manifest.

![Update behavior](./images/proposed-update-behavior.png)

## Future Considerations

### Javascript API for developers

Introducing a javascript API similar to [`beforeinstallprompt`](https://wicg.github.io/manifest-incubations/#beforeinstallpromptevent-interface) would provide developers with more fine grained control over when to trigger an update for their API (or even put UX) on their site to trigger the update process. For our proposal, let's call it `appupdateprompt`.

```
let deferredUpdatePrompt;
window.addEventListener('appupdateprompt', (e) => {
  e.preventDefault();
  // Save the event to trigger it later.
  deferredUpdatePrompt = e;
  // Show customized update prompt for PWA.
  showAppUpdateDialog();
});
```

The developer can then choose to tie it to an update button on their UX:

```
// Gather the data from custom update UI
updateButton.addEventListener('click', async () => {
  deferredPrompt.prompt();
  // Find out whether the user confirmed the installation or not
  const { outcome } = await deferredPrompt.outcome;
  // Act on the user choice, as if the outcome is either `accepted` or `dismissed`
  if (outcome === 'accepted') {
    console.log('Update UX accepted');
  } else if (outcome === 'dismissed') {
    console.log('Update UX dismissed');
  }
});
```

# **How does this solve the problem?**

Let’s review the goals again to see how this proposal meets them: 

> Goal: [`Consistency`](#consistency), [`User Agent Flexibility`](#uaf)

The presence of a different value of `update_token` compared to the one saved, and icon urls changing are the only 2 use-cases that can trigger a manifest update. Since this will be specified in the manifest, user agents can choose to implement their own behavior around this.

> Goal: [`Developer Control`](#devc), [`Preventing unnecessary user interruption.`](#useint)

The users should only see the dialog when the developer wants them to. To do so, the developer has to do either of the 2 things specified (AKA change the  `update_token` value or change the icon urls).

> Goal: [`Reduce network traffic`](#traffic)

The most network heavy traffic is downloading icons, which will only happen when the developer wants them to, and not up to once per day every time the app is accessed.

# **Alternatives considered**

## Use a `version` field to trigger manifest updates

Add the concept of versioning to the web app manifest, so that the developer can change "versions" of their installed experiences to trigger updates. This has been proposed earlier in the manifest spec multiple times (like [manifest#1036](https://github.com/w3c/manifest/issues/1036) and [manifest#1157](https://github.com/w3c/manifest/issues/1157)).

Pros:
- This would also help support this [FR about changelogs](https://github.com/w3c/manifest-app-info/issues/1).

Cons:
- The term is highly overloaded, and it is [hard to get consensus on the data type](https://github.com/w3c/manifest/issues/446#issuecomment-904359501) and what its purpose is. Compared to that, `update_token` is highly confined to a specific use-case, and is simpler to reason about.
- Leads to complications around supporting a lot of things like [version downgrades](https://github.com/w3c/manifest/issues/446#issuecomment-905725612), upgrades as well as [ overall interpretation of the field](https://github.com/w3c/manifest/issues/1036).

## Allow end users to ignore updates

End users can turn off “updates” for their installed app from the settings page of their app.

Cons: 

- Users don’t get new functionalities specified by the developer, like `file_handlers`.  
- Developer complexity is too high to “support” old versions of the same PWAs, leading to incompatible feature sets across different “versions” of the manifest used by users.  
- User agents would have to build a custom UI and logic trigger updates when the user is ready, or to save that a user has ignored one. Otherwise the only solution for the end user would be to uninstall and then reinstall.  
- Old application configurations are often connected with security vulnerabilities.

## Only update on manifest\_url change

While this solution is simple to implement, it breaks existing update behavior, and is also cumbersome for developers, who would have to update where they serve their manifest from every time a change is made.

# **Notes**

- Since [any website is an installable application](https://www.w3.org/TR/appmanifest/#installable-web-applications), if a user agent allows installation of a site that never specified an explicit manifest, update can still occur if the manifest ids matches.  
- The user agent can perform the following tasks if they want:  
  - Notify the end user that the app has been updated, like native apps do.  
  - Provide a warning string in the console for developers if they change icons but forget to add/update the `update_token`.

## [Phishing](#phishing)

A malicious actor can set up an innocent site to come across as non-malicious, and once the user installs it, they can trick the user by acting as a different site by updating itself silently. Some examples of how this can happen:
- **Buyout**: Where a malicious actor buys a non-malicious site (e.g. Wordle) and updates itself to mimic a bank app.
- **Bait-and-switch**: A site for a simple use-case (like an innocent calculator app) updates itself to mimic a bank app.

This is currently handled on Chromium by:
- Showing a dialog if the security sensitive fields have changed.
- Using [Safe Browsing](https://support.google.com/chrome/answer/9890866?hl=en&co=GENIE.Platform%3DAndroid) on Chrome. 