Client Hints is a content negotiation mechanism that enables negotiation of various details about the user's device, conditions and preferences from the client to the server, so that the server is
able to pick a resources that's the best fit for the user in his current situation.  At the same time, exposing those details to servers can increase their ability to passively fingerprint users
without their consent, which is something we would prefer to avoid.  Therefore Client Hints is created as an opt-in mechanism, where if the server intends to use those hints, it needs to actively
request for them.

This document outlines the Client Hints infrastructure, explains it at a higher level and points to the various specification and draft proposals in which it is officially defined.
This document does not describe the various features and hints which rely on the infrastructure. They will be defined in their respective specifications.

# Why conneg?

When we need to choose for a solution that will provide alternative resources to the user's browser based on various factors, we are faced with a design dillema: We can provide the browser with a list
of all potential URLs and let the browser choose the best one, or we can use **content negotiation** and pick the best fit resource variant on the server.

The former option certainly has its place, and it is used successfully across the web in examples like `<picture>`, `srcset`, `<video>`, etc.

At the same time, there are some use-cases where it is not sufficient.  Transformation and adaptation of the page's subresources in a manner that is independent from the page's markup can result in
more scalable and dedicated solutions, that don't have to be assimilated to markup related workflows. 

By decoupling the resource selection from markup, we can enable external services to perform those transformations automatically. We can also provide a wider range of resources, as offering more
resources may result in some server-side costs, but those costs are not directly exposed to the user in the form of markup bloat.

There are many potential dimensions by which we'd want the content adapted to the user:

* Device characteristics 
   - Screen dimensions
   - Screen density
   - Memory and CPU capabilities
   - Range of colors the screen can display 
* Browser characteristics
   - User Agent version and other details
   - Supported formats and codecs
* User preferences
   - Data saving preferences
   - Language preferences
* Network conditions
   - RTT
   - Effective bandwidth

The list above is not necessarily exhaustive, but it can give us an idea as to why providing the URLs for all the above dimensions (and their permutations) to the browser may not be practical in some
cases.

# Why an opt-in?

Since we concluded that content negotiation is indeed useful, why do we want to limit Client Hints to be an opt-in solution? Why can't we expose those details to all servers, and let them do the right
thing?

There are two reasons for that:
* Exposing all the details and dimensions to all servers runs a risk of bloating request headers. There are many potential details that can be useful, and we expect that list to grow over time.
Sending all hints all the time can quickly bloat HTTP requests, and make them significantly larger than they should be. It is better for servers to specifically request the headers they would take
into account, and only send them.
* Exposing those details by default significantly increases the risk of **passive user fingerprinting**, where each one of those details is used to add anthropy bits to the user's "fingerprint", resulting
in accurate identification of users across the web, even when they take measures to avoid such tracking (e.g. blocking of third-party cookies).

Therefore, for Client Hints, we have chosen to make it an opt-in mechanism, in order to avoid those issues.
Unfortunately, that decision doesn't come without its tradeoffs. Having an opt-in mechanism currently means that content adaptation of the initial navigation request on the very first view is not
possible without hacks. We are hoping that in the future we'd be able to push the opt-in process to happen earlier, and enable adaptation of the very-first navigation request as well.

# Opt-in mechanism

## Accept-CH

## Accept-CH-Lifetime

## Same Origin Policy

## Cross-origin hint delegation

# Practical considerations

## Each hint in its own header
## `Sec-` prefix
## Caching considerations

# Privacy considerations

# Conclusion
