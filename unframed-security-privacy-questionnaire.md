# [Self-Review Questionnaire: Security and Privacy](https://w3c.github.io/security-questionnaire/)

Security and privacy questionnaire for
[unframed](https://github.com/WICG/manifest-incubations/blob/gh-pages/unframed-explainer.md).

--------------------------------------------------------------------------------

1.  **What information does this feature expose, and for what purposes?**

    No new information of significance.

    The features introduces the "unframed" display mode for Isolated Web Apps
    (IWAs). The information of whether a window is in "unframed" mode or not is
    exposed now to the application.

2.  **Do features in your specification expose the minimum amount of information
    necessary to implement the intended functionality?**

    Yes.

3.  **Do the features in your specification expose personal information,
    personally-identifiable information (PII), or information derived from
    either?**

    No.

4.  **How do the features in your specification deal with sensitive
    information?**

    The feature does not handle sensitive information.

5.  **Does data exposed by your specification carry related but distinct
    information that may not be obvious to users?**

    No.

6.  **Do the features in your specification introduce state that persists across
    browsing sessions?**

    Yes.

7.  **Do the features in your specification expose information about the
    underlying platform to origins?**

    Indirectly, yes.

    The feature is only available in Chrome on ChromeOS, so one can infer the
    platform from the availability of this feature.

8.  **Does this specification allow an origin to send data to the underlying
    platform?**

    No.

9.  **Do features in this specification enable access to device sensors?**

    No.

10. **Do features in this specification enable new script execution/loading
    mechanisms?**

    No.

11. **Do features in this specification allow an origin to access other
    devices?**

    No.

12. **Do features in this specification allow an origin some measure of control
    over a user agent's native UI?**

    Yes.

    The "unframed" display mode removes the user agent native UI surrounding the
    web contents. This allows the IWA to fully customize its UI surface
    edge-to-edge, similar to the experience allowed to native applications. An
    attacker can abuse "unframed" to spoof the user agent, other apps, or the
    OS.

13. **What temporary identifiers do the features in this specification create or
    expose to the web?**

    None.

14. **How does this specification distinguish between behavior in first-party
    and third-party contexts?**

    N/A.

    This feature is controlled via the application manifest, which can't be
    modified by third-parties. A third-party may have read-only access to the
    manifest JSON (so it knows what display modes are configured) or to the
    current display mode of the application via APIs like `window.matchMedia`.

15. **How do the features in this specification work in the context of a
    browser’s Private Browsing or Incognito mode?**

    IWAs cannot be opened in incognito mode.

16. **Does this specification have both "Security Considerations" and "Privacy
    Considerations" sections?**

    Yes, here is
    [the relevant section in the explainer](https://github.com/WICG/manifest-incubations/blob/gh-pages/unframed-explainer.md#security--privacy-considerations).

17. **Do features in your specification enable origins to downgrade default
    security protections?**

    No.

18. **What happens when a document that uses your feature is kept alive in
    BFCache (instead of getting destroyed) after navigation, and potentially
    gets reused on future navigations back to the document?**

    N/A.

    This feature has no impact on the Back/Forward Cache. The display mode is a
    property of the browser window itself, not the individual document.

    It's worth noting that out-of-scope navigation is disallowed within IWA
    windows. The browser will handle the navigation in a new tab instead.

19. **What happens when a document that uses your feature gets disconnected?**

    N/A.

    This feature does not change how the browser handles document disconnects.

20. **Does your spec define when and how new kinds of errors should be raised?**

    No.

21. **Does your feature allow sites to learn about the user's use of assistive
    technology?**

    No.

22. **What should this questionnaire have asked?**

    *   **How can the user close an unframed window if the application becomes
        unresponsive or behaves maliciously?**

        The host OS retains control over the window. Standard system-level
        actions remain functional, like keyboard shortcuts (e.g., Alt+F4 to
        close, Alt+Tab to switch windows), gestures (such as overview mode), or
        right-click and close the app in the OS shelf.

        We also allow apps to have windows in unframed mode alongside windows in
        other display modes, like standalone. This means apps can have the title
        bar in windows where it makes sense, while other windows are unframed
        where needed.

        Lastly, apps can implement a custom in-app title bar in unframed windows
        if needed.

    *   **Does this feature depend on, or affect, any user agent permissions?**

        The "unframed" display mode requires the `window-management` permission.
        It must be granted to the origin by the user or via enterprise policy.

        In the standalone display mode there is a shortcut for users to control
        app permissions via the 3-dot menu in the title bar. In "unframed" this
        menu is not available since the title bar is gone altogether. On
        ChromeOS users can manage app permissions by right-clicking the app icon
        in the OS shelf and selecting "App Info". Other OSs may not offer a
        similar workaround, so the feature is only launching on ChromeOS for the
        foreseeable future.

