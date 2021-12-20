# TAG Security & Privacy Questionnaire Answers for *Markup based Client Hints delegation for third-party content* #

* **Author:** arichiv@chromium.org
* **Questionnaire Source:** https://www.w3.org/TR/security-privacy-questionnaire/#questions

## Questions ##

**1. What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?**

This feature exposes all the same information exposed today via the User-Agent string. See the response to question 8 for more details. Developers typically use this information for differential content serving, or to work around bugs. See https://github.com/WICG/ua-client-hints#use-cases for more use cases in more detail.

The key difference between the UA string and UA-CH is that it moves the model from passive to active. Rather than a site passively receiving all possible information for all requests, UA-CH requires the site to make active requests for hints it needs in such a way that a User Agent may observe such calls and intervene, depending on UA privacy policies or user preferences.

**2. Is this specification exposing the minimum amount of information necessary to power the feature?**

We believe so, yes.

**3. How does this specification deal with personal information or personally-identifiable information or information derived thereof?**

It does not deal directly in PII, however information provided by the User-Agent string may be used for passive fingerprinting. The EFF estimates 10 bits of entropy can be obtained from the UA string. UA-CH provides a path forward to reduce the default passive entropy sent for all requests. By moving this information to an active surface, UAs could monitor and apply privacy interventions based on principled UA policies such as Safari’s [Intelligent Tracking Prevention](https://webkit.org/tracking-prevention/), Firefox’s [Enhanced Tracking Protection](https://blog.mozilla.org/en/products/firefox/latest-firefox-rolls-out-enhanced-tracking-protection-2-0-blocking-redirect-trackers-by-default/), Chrome’s [Privacy Budget](https://github.com/bslassey/privacy-budget) proposal, or similar.

**4. How does this specification deal with sensitive information?**

If a user agent considers certain UA hints to be sensitive (for example, bitness, device model, or CPU architecture), this specification encourages them to return the empty string or a fictitious value (see the end of https://wicg.github.io/ua-client-hints/#http-ua-hints). UAs are free to identify which hints might be sensitive, and in which contexts, according to their privacy policies (ITP, ETP, Privacy Budget, etc.).

**5. Does this specification introduce new state for an origin that persists across browsing sessions?**

This specification does in the sense that it provides additional hints that may be in the Accept-CH cache (provided by Client Hints Infrastructure): https://wicg.github.io/client-hints-infrastructure/#accept-ch-cache-definition

**6. What information from the underlying platform, e.g. configuration data, is exposed by this specification to an origin?**

In addition to information about a user agent, this specification may expose the brand name, version, architecture, bitness, and “mobileness” of the underlying operating system, as well as device model (for non-desktop platforms). The same info can be obtained directly (or inferred indirectly) from the UA string today, e.g., the string “(Windows NT 10.0; Win64; x64)” denotes a 64-bit browser running on Windows 10 (64-bit). With UA-CH, only the platform name and “mobileness” is exposed by default. To get the other information, an origin needs to explicitly request it.

**7. Does this specification allow an origin access to sensors on a user’s device**

No.

**8. What data does this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.**

In addition to the data described in question 6, this specification may expose UA brand name, UA major version, and UA full version. All of this information is already exposed in the User-Agent header.

The difference between UA-CH and the UA string, is that an origin needs to request this data -- with the exception of platform name, UA brand name, UA major version, and platform mobileness. It’s not sent by default for all requests (and subresource requests). The UA is free to grant, deny, or even to return the empty string or fake data for such requests, according to their privacy policies (ITP, ETP, Privacy Budget, etc.).

**9. Does this specification enable new script execution/loading mechanisms?**

No.

**10. Does this specification allow an origin to access other devices?**

No.

**11. Does this specification allow an origin some measure of control over a user agent’s native UI?**

No.

**12. What temporary identifiers might this specification create or expose to the web?**

Each new client hint in this specification can be abused to store 1 extra bit in the Accept-CH cache, as described at https://wicg.github.io/client-hints-infrastructure/#accept-ch-cache-definition. No other new identifying information is exposed.

**13. How does this specification distinguish between behavior in first-party and third-party contexts?**

This specification inherits the following characteristics from Client Hints Infrastructure, “Origin opt-in applies to same-origin assets only and delivery to third-party origins is subject to explicit first party delegation via Permissions Policy, enabling tight control over which third party origins can access requested hint data.” Hints need to be explicitly delegated by developers from a first- to third-party context.

**14. How does this specification work in the context of a user agent’s Private Browsing or "incognito" mode?**

It will work the same. Accept-CH preferences will not be maintained between “private” and “non-private” browsing sessions. See https://wicg.github.io/client-hints-infrastructure/#accept-ch-cache-definition

**15. Does this specification have a "Security Considerations" and "Privacy Considerations" section?**

Yes. https://wicg.github.io/ua-client-hints/#security-privacy

**16. Does this specification allow downgrading default security characteristics?**

No.

**1. What information might this feature expose to Web sites or other parties, and for what purposes is that exposure necessary?**

This feature allows origins to delegate client hints to third parties via HTML markup (a meta tag), something only currently possible via HTTP headers. This is being added to better support adaptable third party content (e.g., [variable fonts](https://developer.mozilla.org/en-US/docs/Web/CSS/CSS_Fonts/Variable_Fonts_Guide), [color vector fonts](https://www.chromestatus.com/feature/5638148514119680), and [responsive images](https://github.com/w3c/webappsec-permissions-policy/issues/55#issuecomment-406627096)) used in contexts which permit modification of markup but not HTTP headers.

The only added risk with markup is the possibility that third parties could inject delegation requests into the first party document that the first party had not intended. To mitigate that, we require that the delegation is only respected before javascript is executed, and not after. With this limitation, no actor not previously able to delegate client hints (via HTTP) should now be able to do so (via HTML).

**2. Do features in your specification expose the minimum amount of information necessary to enable their intended uses?**

Yes, it exposes no information not already possible to expose within the client hints system. It simply allows for HTML markup instead of HTTP headers.

**3. How do the features in your specification deal with personal information, personally-identifiable information (PII), or information derived from them?**

It does not deal directly in PII, however information provided by client hints can reveal device or browser characteristics. User agents can send the empty string if they wish to withhold requested client hints.

**4. How do the features in your specification deal with sensitive information?**

It does not handle sensitive information.

**5. Do the features in your specification introduce new state for an origin that persists across browsing sessions?**

No, this specification does not write changes to the Accept-CH cache: https://wicg.github.io/client-hints-infrastructure/#accept-ch-cache-definition

**6. Do the features in your specification expose information about the underlying platform to origins?**

Yes, but not by default. It makes it possible for a first party to intentionally delegate client hints to a third party.

**7. Does this specification allow an origin to send data to the underlying platform?**

Yes, the specification allows the origin to indicate which features (client hints) the origin wishes to receive and/or delegate to third party origins.

**8. Do features in this specification enable access to device sensors?**

No

**9. What data do the features in this specification expose to an origin? Please also document what data is identical to data exposed by other features, in the same or different contexts.**

See the answer to question 6. For a list of features see https://wicg.github.io/client-hints-infrastructure/#policy-controlled-features.

**10. Do features in this specification enable new script execution/loading mechanisms?**

No

**11. Do features in this specification allow an origin to access other devices?**

No

**12. Do features in this specification allow an origin some measure of control over a user agent’s native UI?**

No

**13. What temporary identifiers do the features in this specification create or expose to the web?**

Nothing beyond what's currently capable with Client Hints.

**14. How does this specification distinguish between behavior in first-party and third-party contexts?**

This specification inherits the following characteristics from Client Hints Infrastructure, “Origin opt-in applies to same-origin assets only and delivery to third-party origins is subject to explicit first party delegation via Permissions Policy, enabling tight control over which third party origins can access requested hint data.” Hints need to be explicitly delegated by developers from a first- to third-party context.

**15. How do the features in this specification work in the context of a browser’s Private Browsing or Incognito mode?**

It will work the same. Accept-CH preferences will not be maintained between “private” and “non-private” browsing sessions. See https://wicg.github.io/client-hints-infrastructure/#accept-ch-cache-definition

**16. Does this specification have both "Security Considerations" and "Privacy Considerations" sections?**

Yes. https://wicg.github.io/client-hints-infrastructure/#privacy

**17. Do features in your specification enable origins to downgrade default security protections?**

No

**18. How does your feature handle non-"fully active" documents?**

N/A

**19. What should this questionnaire have asked?**

N/A

