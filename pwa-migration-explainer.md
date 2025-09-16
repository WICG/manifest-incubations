# PWA Origin Migration

## Authors:

*   Marijn Kruisselbrink (Google)

## Participate

*   **Issue Tracker:** [WICG/manifest-incubations on GitHub](https://github.com/WICG/manifest-incubations/issues) (for spec-related issues)
*   **Chromium Implementation Bug:** [crbug.com/380486617](https://issues.chromium.org/380486617) (for implementation-specific discussion)

## Introduction

When a user installs a Progressive Web App (PWA), its identity and security context are tightly bound to its web origin (e.g., `app.example.com`). This presents a significant challenge for developers who need to change their PWA's origin due to rebranding, domain restructuring, or technical re-architecture. Currently, such a change forces users to manually uninstall the old app and reinstall the new one, leading to a disruptive experience and potential user loss. This proposal introduces a mechanism for developers to seamlessly migrate an installed PWA to a new, same-site origin, preserving user trust and permissions.

## User-Facing Problem

Imagine you have the "SocialApp" app installed on your computer from `www.example.com/social`. One day, the company decides to move the app to its own dedicated home at `social.example.com`. Without a migration mechanism, the app you have installed would either break or redirect you to the new site in a generic browser window, losing its app-like feel. You would have to figure out that you need to uninstall the old app and install the new one from the new address. You might also lose your settings, like whether you've allowed the app to send notifications.

This feature aims to make that transition seamless. Instead of a broken experience, the app would notify you of an available update. With your approval, the app would relaunch from its new home at `social.example.com`, with your notification settings intact.

### Goals

*   **Seamless User Experience:** Users should be able to transition to a new PWA origin with a simple, non-disruptive update flow.
*   **Developer Enablement:** Provide a clear mechanism for developers to migrate their PWAs for reasons like rebranding (e.g., "ConnectApp" to "ConnectX"), SEO optimization (e.g., `photo-app.example.com` to `photos.example.com/app`), or backend architecture changes (e.g., `mail.example.com` to `app.example.com`).
*   **Permission Persistence:** User-granted permissions (e.g., notifications, location) should be securely and transparently migrated to the new origin where possible.
*   **Security:** The migration process must be secure, preventing malicious takeovers or phishing attacks.

### Non-goals

*   **Cross-Site Migration:** While we may explore solutions for cross-origin migration in the future, this initial proposal is limited to same-site migrations. Allowing migration between different sites (e.g., from `example.com` to `another-site.com`) would introduce significantly more security risks. For example, if a site were compromised, an attacker could migrate its users to a malicious phishing site, potentially transferring user trust and any granted permissions. The same-site restriction ensures that both the old and new origins are controlled by the same entity, reducing the risk of this category of attack.
*   **Data Migration:** User data stored locally (e.g., in IndexedDB, Cache Storage) is not within the scope of this migration. Developers are responsible for migrating this data, typically via server-side logic.

## Proposed Approach

The solution is a manifest-based handshake between the old and new origins. A developer signals the migration by adding new members to the web app manifests of both the old and new applications. The migration can be discovered in two ways:

1.  **By visiting the new app:** The user navigates to the new site (for example because the old site redirects to the new site). The browser fetches its manifest, sees a `migrate_from` field pointing to the old, installed app, and initiates the migration flow for the old app.
2.  **By visiting the old app:** The browser checks for an update to the old app's manifest and sees a `migrate_to` field. The browser fetches the manifest for the new site, verifyeis it has a matching `migrate_from` field, and initiated the migration flow for the old app.

### Dependencies on non-stable features

This proposal is designed to work with and depends on the [Predictable App Updating](https://github.com/WICG/manifest-incubations/blob/gh-pages/predictable-app-updating.md) proposal. The user-facing update prompts and the timing of migration checks will align with the mechanisms defined in that feature.

### Solving the PWA Migration with Manifests

**1. The "New" App Confirms the Migration (Required):**

The developer adds a `migrate_from` member to the manifest of the PWA at the new origin. This confirms that it is the legitimate successor to the old app. For the browser to process the `migrate_from` field, the manifest must also contain an `id` field.

*Manifest on `social.example.com`:*
```json
{
  "name": "SocialApp",
  "id": "/",
  "start_url": "/",
  "migrate_from": [
    "https://www.example.com/social/"
  ]
}
```

**2. The "Old" App Declares the Migration (Optional):**

To proactively trigger the update flow without waiting for the user to visit the new site, the developer can add a `migrate_to` member to the manifest of the PWA at the original origin. The `migrate_to` member contains both the `id` of the app the site wants to migrate to, as well as an `install_url` that can be used by the browser to fetch a manifest matching the given `id`.

*Manifest on `www.example.com`:*
```json
{
  "name": "SocialApp (Legacy)",
  "id": "/social/",
  "start_url": "/social/start",
  "migrate_to": {
    "id": "https://social.example.com/",
    "install_url": "https://social.example.com/download?usp=migrate"
  }
}
```

Once the migration is discovered (via either method), the browser will present the user with an update prompt. The developer can control the user experience with the optional `behavior` field in the `migrate_from` object:
*   `"suggest"`: The user is (passively) notified of the update but can ignore it.
*   `"force"`: The next time the user launches the app being migrated (after a manifest is parsed with this behavior specified), they are presented with a blocking dialog requiring them to either migrate or uninstall the app.

### Permission Migration and User Experience

A key part of this proposal is the migration of user-granted permissions from the old origin to the new one. To maintain user trust and transparency, this does not happen silently in the background.

It's possible that a user has already interacted with the new origin in a regular browser tab and has set different permissions than those for the old PWA. To handle this safely and prevent unexpected granting of capabilities, the migration will use the **most restrictive** permission setting. For example, if a user had denied camera access on the old PWA's origin but had previously allowed it on the new origin, the resulting permission for the migrated app will be 'denied'. This ensures the user's most cautious intent is respected.

After the user has approved the app migration and launches the app on its new origin for the first time, the browser should present a UI element (for example, a prompt or a highly visible notification) explaining that the permissions granted to the old app can be carried over to the new one. The migration of these permissions will only complete after the user acknowledges this UI.

This confirmation step ensures the user is explicitly aware that permissions are moving to a new URL and remains in control. It also provides a consistent point of information for migrations that may have occurred automatically on a secondary device via browser sync.

### Example User Interface

To illustrate the user flow, here are some ASCII art mockups of what the UI could look like.

**"Suggest" Behavior Dialog:**

This dialog can be triggered when a migration is available and the developer has not marked it as "force".

```
+-----------------------------------------------------+
| A new version of this app is available.             |
|-----------------------------------------------------|
|                                                     |
| Review the URL change to make sure you're           |
| comfortable with it before updating.                |
|                                                     |
| URL: https://www.example.com/social/ ->             |
|      https://social.example.com/                    |
|                                                     |
| [ Ignore ]     [ Uninstall ] [ Relaunch to update ] |
+-----------------------------------------------------+
```

For "force" update, a similar dialog would appear on next launch, with the major difference being that the "Ignore" button is absent.

**Post-Migration Permission Notification:**

This notification appears after the migration is complete to confirm the permission change. Permissions won't be migrated until the user interacts with this dialog.

```
  /---------------------------------------------\
 (  Check permissions for this app           [X] )
 (-----------------------------------------------)
 (  This app's URL has updated and permissions   )
 (  you gave now apply to any page that is part  )
 (  of social.example.com                        )
  \______________________[ Change ]__[ Got it ]_/
```

### Handling Redirects and Pre-Migration Updates

A common migration strategy is to redirect all traffic from the old origin to the new one. This creates a problem: if the browser cannot access the old origin's pages, it can no longer fetch the old manifest to process important updates that might be needed *before* the user migrates, such as changes to `scope_extensions`.

To solve this, the `migrate_from` field in the **new** app's manifest can be an object that includes an optional `install_url`. This URL must not redirect to the new app, and point to the **old** app's manifest. This gives the browser a stable endpoint to keep the old, unmigrated app up-to-date.

*Manifest on `social.example.com` with optional `install_url`:*
```json
{
  "name": "SocialApp",
  "id": "/",
  "start_url": "/",
  "migrate_from": [
    {
      "id": "https://www.example.com/social/",
      "install_url": "https://www.example.com/social/install.html?noredirect=true"
    }
  ]
}
```

### Relationship to Predictable App Updating

PWA Origin Migration is a specialized form of a PWA update. It is designed to work harmoniously with the broader [Predictable App Updating](https://github.com/WICG/manifest-incubations/blob/gh-pages/predictable-app-updating.md) proposal.

Developers can combine an origin migration with other manifest changes, such as updating the app's `name` or `icon`. When using the `"suggest"` behavior, the update dialog will present all these changes to the user for review, just like a standard update.

However, there is a critical security consideration for `"force"` migrations. To prevent a developer from forcing a user to accept a potentially deceptive name or icon change, the browser will enforce a separation of concerns. During a **forced migration, the browser will only apply the URL change**. If the new manifest also contains changes to security-sensitive fields like `name` or `icon`, the browser will defer these changes. After the migration is complete, these deferred changes will be processed as a standard, ignorable update, allowing the user to review them separately and preserving user control over the app's identity.

## Alternatives considered

### Require a 'cooling off' period

A mandatory waiting period (e.g., 2 weeks) after a migration is declared was considered to help prevent malicious activity if a site is compromised. However, this was deemed overly complex and potentially not useful, as there's no guarantee a developer would notice a compromise within that period. The **same-site restriction** is a much stronger and more reliable security guarantee, as it ensures both origins are controlled by the same entity.

### Require a two-way reference

We are currently proposing to only require a one-way reference from the new manifest to the old one. An alternative we considered is to require a two-way reference, where the old manifest must point to the new one, and the new one must point back.

The main advantage of a two-way reference would be enhanced security. A clear handshake between both sites would make it more difficult for a malicious actor to initiate a migration. However, this benefit is less critical given that the proposal is already restricted to same-site migrations. The downsides of this approach are:
* In difficult scenarios where the old site is already redirecting to the new one, it can be hard to host a manifest on the old site.
* For very old installs, the developer might not maintain the old site / manifest anymore.
* It can be slow, as the browser needs to fetch the old manifest to verify the migration, even if the user navigated to the new site directly.

Given these downsides, we believe that a one-way reference is the better solution.

### Require an Explicit `id` in both manifests

Another alternative considered was to make the `id` manifest member mandatory for not just the new, but also the old app. In this model, the browser would ignore the `migrate_to` and `migrate_from` fields entirely if an explicit `id` was not present in the manifest. This would enforce the best practice of using a stable identifier, preventing developers from accidentally creating new app identities by changing the `start_url`, which would break the migration. However, this was rejected as it would create significant friction for developers with existing PWAs that do not have an `id` set. They would be forced to update their old app first and wait for it to propagate before they could begin a migration, adding complexity and delay to the process. The current proposal is more flexible and accommodating to existing applications.

### Never require an `id`

We also considered not requiring an `id` field in the manifest at all for migrations. The migration process itself does not technically depend on the `id` field. The primary motivation for requiring it is to encourage the adoption of a best practice that provides a stable identity for a PWA, preventing [potential issues in the future](https://docs.google.com/presentation/d/1QqriRwxqzo8Dqwj3BOixE6AWIm0Kr4y9-ScOm-MKhMs/edit?slide=id.g2f9677cb0f8_0_0#slide=id.g2f9677cb0f8_0_0). However, making this a requirement for an unrelated feature could be seen as using the migration mechanism to enforce an orthogonal best practice.

### Differentiated Rules for Same-Site Migrations

We considered that not all same-site migrations carry the same level of risk. For instance, migrating from a specific subdomain like `maps.example.com` to a broader one like `www.example.com` could unintentionally widen the scope of permissions in a way a user might not expect. We explored introducing stricter requirements or different UI treatments for these "scope-widening" migrations. Ultimately, we opted for a single, consistent rule for all same-site migrations to keep the feature simple and predictable for both developers and users, relying on the explicit, post-migration permission UI to ensure user clarity and consent.

## Accessibility, Privacy, and Security Considerations

### Accessibility

The user-facing UI for the migration (update notifications, confirmation dialogs) will be built using existing, accessible browser components. Even when the site requests `"force"` behavior for the migration, the browser should not interrupt the user with what they are doing. Instead the migration UI should be triggered for example the next time the user launches the app being migrated.

### Privacy

The primary privacy consideration is the migration of permissions. A user who granted permissions to `old.example.com` might not want `new.example.com` to have them. This is mitigated in two ways:
1.  The **same-site restriction** ensures the same entity controls both origins.
2.  The user is **explicitly informed** via UI that their permissions will be moved to the new URL after the migration is complete, and no permissions are migrated until the user acknowledges this.

### Security

The primary security threat is a hostile takeover, where a malicious actor could try to migrate a legitimate PWA to a phishing site. The **same-site restriction** is the critical defense here. Since an attacker cannot control a different eTLD+1, they cannot hijack a PWA. For example, `example.com` could not be migrated to `evil-phishing-site.com`.

## Stakeholder Feedback / Opposition

*   **Major App Developers:** Positive. Several have expressed a strong need for this feature to support their product roadmaps. Their feedback has been crucial in defining the requirements, particularly the need for both "suggested" and "forced" migration behaviors.
*   **Browser Implementors:** No signals. This is a new proposal being incubated.

## Security and Privacy Self-Review

This section contains answers to the [W3C Security and Privacy Self-Review Questionnaire](https://w3c.github.io/security-questionnaire/).

### 2.1 What information does this feature expose, and for what purposes?
This feature does not directly expose user information to websites. It allows a website (`new.example.com`) to declare a relationship with another (`old.example.com`). The browser uses this information, combined with its own knowledge of whether the user has the old PWA installed, to initiate a migration UI. The new site does not receive a confirmation that the user has the old app installed; the entire process is mediated by the user through the browser's UI.

### 2.2 Do features in your specification expose the minimum amount of information necessary to implement the intended functionality?
Yes. The feature only exposes the developer-declared relationship between two origins. No user-specific information is exposed to the web.

### 2.3 Do the features in your specification expose personal information, personally-identifiable information (PII), or information derived from either?
No.

### 2.4 How do the features in your specification deal with sensitive information?
Not applicable. No sensitive information is exposed.

### 2.5 Does data exposed by your specification carry related but distinct information that may not be obvious to users?
No.

### 2.6 Do the features in your specification introduce state that persists across browsing sessions?
Yes, the result of a successful migration is persistent: the installed PWA's identity is permanently changed from the old origin to the new one. This is a change to existing persistent state (the app installation), not the introduction of a new type of storage.

### 2.7 Do the features in your specification expose information about the underlying platform to origins?
No.

### 2.8 Does this specification allow an origin to send data to the underlying platform?
No.

### 2.9 Do features in this specification enable access to device sensors?
Not directly. It allows for the migration of *permissions* to access sensors (e.g., camera, microphone, location) from the old origin to the new, same-site origin. This permission migration only occurs after the user has approved the origin migration and then explicitly consented to the permission transfer via a post-migration UI prompt.

### 2.10 Do features in this specification enable new script execution/loading mechanisms?
No.

### 2.11 Do features in this specification allow an origin to access other devices?
No.

### 2.12 Do features in this specification allow an origin some measure of control over a user agent’s native UI?
Yes. The core purpose of the feature is to change the origin associated with an installed PWA's window and identity. This is a significant change to the user's app experience, and it is explicitly communicated to the user for their approval before it occurs.

### 2.13 What temporary identifiers do the features in this specification create or expose to the web?
None.

### 2.14 How does this specification distinguish between behavior in first-party and third-party contexts?
This feature is only relevant in a first-party context, as a web app manifest is only applicable to a top-level browsing context.

### 2.15 How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?
This feature does not apply to Private Browsing or Incognito mode, as PWA installation is generally not available in those modes.

### 2.16 Does this specification have both "Security Considerations" and "Privacy Considerations" sections?
Yes, they are present in this document.

### 2.17 Do features in your specification enable origins to downgrade default security protections?
No.

### 2.18 What happens when a document that uses your feature is kept alive in BFCache (instead of getting destroyed) after navigation, and potentially gets reused on future navigations back to the document?
The migration process is managed by the browser's systems, not by a specific document. The discovery of a migration might be triggered by fetching a manifest from a document, but the subsequent UI and state changes are not tied to that document's lifecycle. Therefore, a document involved in the discovery process being placed in BFCache will not affect the migration.

### 2.19 What happens when a document that uses your feature gets disconnected?
Similar to the BFCache case, the migration process is not tied to the lifecycle of any single document. If a document that triggered a migration check is disconnected (e.g., an iframe is removed), the browser-level migration process will proceed unaffected.

### 2.20 Does your spec define when and how new kinds of errors should be raised?
The specification will need to define error handling for cases such as a `migrate_to` field pointing to a URL that returns a 404 error, or a manifest that is invalid or does not contain the required reciprocal `migrate_from` field. These errors will be handled by the browser and will result in the migration not being offered to the user. They could be exposed to developers in the DevTools console but will not be exposed to the web.

### 2.21 Does your feature allow sites to learn about the user’s use of assistive technology?
No. The UI surfaces for this feature (dialogs, notifications) will be implemented using standard, accessible browser components and will not introduce any mechanism for detecting the use of assistive technology.

### 2.22 What should this questionnaire have asked?
N/A
