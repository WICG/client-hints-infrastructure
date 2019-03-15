Client Hints is a content negotiation mechanism that enables the client to send
various details about the user's device, conditions and preferences to the
server. It enables the server to pick a resource that's the best fit for the
user in his current situation, to improve the user's performance or experience.

At the same time, exposing those details to servers can increase their ability
to passively fingerprint users without their consent, which is something we
need to avoid.  Therefore Client Hints is created as an opt-in mechanism, where
if the server intends to use those hints, it needs to actively request for
them.

This document outlines the Client Hints infrastructure, explains it at a higher
level and points to the various specification and draft proposals in which it
is officially defined. At the same time, it does not describe the various
features and hints which rely on the infrastructure. They will be defined in
their respective specifications.

# Why conneg?

When we need to choose a solution that will provide alternative resources
to the user's browser based on various factors, we are faced with a design
dillema: We can provide the browser with a list of all potential URLs and let
the browser choose the best one, or we can use **content negotiation** and pick
the best fit resource variant on the server.

The former option certainly has its place, and it is used successfully across
the web in examples like `<picture>`, `srcset`, `<video>`, etc.

At the same time, there are some use-cases where it is not sufficient.
Transformation and adaptation of the page's subresources in a manner that is
independent from the page's markup can result in more scalable and dedicated
solutions, that don't have to be assimilated to markup related workflows. 

By decoupling the resource selection from markup, we can enable external
services to perform those transformations automatically. We can also provide a
wider range of resources, as offering more resources may result in some
server-side costs, but those costs are not directly exposed to the user in the
form of markup bloat.

There are many potential dimensions by which we'd want the content adapted to
the user:

* Device characteristics 
   - Screen dimensions
   - Screen density
   - Memory and CPU capabilities
   - Range of colors the screen can display 
* Browser characteristics
   - User Agent major or full version
   - Device model, OS version and platform
   - Supported formats and codecs
* User preferences
   - Data saving preferences
   - Preferred language
* Network conditions
   - RTT
   - Effective bandwidth

The list above is not necessarily exhaustive, but it can give us an idea as to
why simply providing the browser with URLs for all the above dimensions and
their permutations may not be practical, at least in some cases.

# Why an opt-in?

Since we concluded that content negotiation is indeed useful, why do we want to
limit Client Hints to be an opt-in solution? Why can't we expose those details
to all servers, and let them do the right thing?

There are two reasons for that:
* Exposing all the details and dimensions to all servers runs a risk of
  bloating request headers. There are many potential details that can be useful
  for content negotiation, and we expect that list to grow over time. Sending
  all the hints all the time can quickly bloat HTTP requests, and make them
  significantly larger than they should be. To avoid request bloat, it is
  more efficient for servers to specifically request the headers they would
  take into account, and send only these hints with upgoing requests.
* Exposing those details by default significantly increases the risk of
  **passive user fingerprinting**, where each one of those details can be used
  to add anthropy bits to the user's "fingerprint", resulting in accurate
  identification of users across the web, even when they take measures to avoid
  such tracking (e.g. blocking of third-party cookies).

Therefore, for Client Hints, we have chosen to make it an opt-in mechanism, in
order to avoid those issues.  Unfortunately, that decision doesn't come without
its tradeoffs. Having an opt-in mechanism currently means that content
adaptation of the initial navigation request on the very first view is not
possible without hacks. We are hoping that in the future we'd be able to push
the opt-in process to happen earlier, and enable adaptation of the very-first
navigation request as well.

At the same time, for features which are critical for content negotiation of
navigation requests, browsers may choose to send them regardless of a server
opt-in, in case it deems exposing that information as something that doesn't
increase the passive fingerprinting surface.

## Privacy enhancing content negotiation???

Content negotiation is typically frowned upon as a mechanism that enables
passive fingerprinting, by adding different bits of data to different user's
requests by default, and enabling servers to use that to tell users apart
without leaving any trace of that activity.  Adding an opt-in enables us to
avoid that, as now servers need to tell the browsers which information they
need, making any such fingerprinting use detectable.

But, Client Hints can enable us to do more than that for user privacy, and turn
passive-fingerprinting-enabling content-negotiation mechanisms into opt-in-only
mechanisms.  That would effectively reduce the passive fingerprinting surface
on the web, and enable browser to keep closer tabs on entities that use that
information for seemingly nefarious reasons.

# The Client Hints infrastructure

How can servers opt into receiving hints from the client? How should client
send those hints? And how should that be handled across origins?

## Opt-in mechanism

Server can opt-in to Client Hints by using the HTTP response headers described
in the sections below. They can similarly opt-in by using the headers' HTML
equivalents, the `<meta>` HTML tag and its `http-equiv` attribute.

### `Accept-CH`

The `Accept-CH` header enables server to request specific hints from the
browser. The header's value is a comma separated list, where each value in that
list represents a request header hint that the server is interested in
receiving.

#### Example

If the server's response to the navigation request includes the `Accept-CH:
foo, bar` header, same-origin subresource requests on the page will include the
`Sec-Foo: foo-value` and `Sec-Bar: bar-value` request headers.

### `Accept-CH-Lifetime`

In order to enable hints for navigation requests (albeit not to the very-first
one), the `Accept-CH-Lifetime` header tells the browser to maintain the Client
Hints opt-in persistency over time.  The header's value is an integer that
represents the number of seconds that preference should be kept by the browser.

#### Example

If the server's response to the navigation request includes the `Accept-CH:
foo, bar` and `Accept-CH-Lifetime: 3600`, same-origin requests on *on that
origin* will include the `Sec-Foo: foo-value` and `Sec-Bar: bar-value` request
headers for the next hour, including the navigation requests of future
navigations.

## Same Origin Policy

The opt-in mechanism description above included the fact that the opt-in is
only applied to same-origin requests, and only applied when it is received on a
navigation resource. Why is that important?

As mentioned before, we don't want the mechanism to be used to increase
fingerprinting on the web.  The hints that CH provides should generally be
available through JS APIs (see the "Privacy Considerations" section).  That
means that for *active* resources (e.g. HTML), CH does not increase the active
fingerprinting surface. Those resources can already run scripts to exfiltrate
that data, and CH only provides them with a more convenient and performant way
to do that, when that data is needed for content negotiation purposes.

But, that also means that we don't want passive subresources (e.g. images) to
be able to exfiltrate the same data, and we certainly don't want them to be
able to exfiltrate it for the entire origin for a longer amount of time.

Therefore, by default, client hints are only applicable to the same origin as
the page navigation, and the opt-in is only available for navigation requests,
which are potentially active resources.

## Cross-origin hint delegation

If Client Hints are only being sent on same-origin requests, how can we handle
cross-origin requests?

As the use-case for client hints is to enable content negotiation at scale, and
as many optimization services are offered over different origins than the main
page's origin, cross-origin support is a vital part of client hints.

In order to support that use-case, we have defined delegation of client hints
to specific cross-origin hosts, using Feature Policy.

Servers can opt-in to such delegation by applying `ch-` prefixed policies for
the desired hints.

### Example

A server sending the following header `Feature-Policy: ch-example foo.com
bar.com; ch-example-2 foobar.org` will delegate the `example` hint to the
"foo.com" and "bar.com" origins and `example-2` to the foobar.org origin.  That
would enable those origins to receive those hints and perform content
adaptation based on them.

### Privacy implications

Why is it privacy safe for pages to delegate hints to certain third party
origins?

Since we're treating Client Hints as an active fingerprinting equivalent, we
are comfortable with the information it exposes to servers, as the same
information is already freely available in the equivalent JS APIs. Similarly,
third party delegation is safe because pages are already able to use other
means, such as link decoration, to achieve the same information sharing with
third parties, only in less performant ways.

## `Sec-` prefix Adding new request headers increases the risk that legacy
server systems already use those values for a different purpose. Changing the
request header values such legacy systems receive may result in server bugs.

While that risk is significantly mitigated by the opt-in mechanisms of Client
Hints, other specifications relying on Client Hints should reduce it even
further by adding a `Sec-` prefix to Client Hints request header names they
pick for their features.

Adding a `Sec-` prefix will also enable us to simplify the processing model, as
such headers cannot be added by a Service Worker, and potentially enable us to
add all such headers to CORS' safe-list, avoiding unwanted preflights for
requests with Client Hints.

## Caching considerations

When adapting content to specific client hints request headers, servers should
add the `Vary` header to their responses to indicate such adaptation to caches,
and make sure that such resources are not cached using only their URL as the
cache key.

This is also the reason that each Client Hint is represented using a separate
header, in order to reduce variance in caches in responses that may rely on
some hints, but not others.

# Privacy considerations

When implementing Client Hints, browsers should make sure that certain privacy
related precautions are being taken:

* Client Hint features should not be shipped unless there is a JS-based
  equivalent API, which enables developers access to the same data and is
  deemed privacy-safe to ship.
  - One reason for that is Extensible Web principles - we want the shipped
    features to be somewhat polyfillable (Even if `Sec-` headers cannot be
    added by Service Workers)
  - From a privacy perspective, since we consider Client-Hints to be a
    potential active fingerprinting vector, if information is already available
    through other active fingerprinting means, such as a Javascript API, it is
    safe to ship it through Client Hints as well.
* Browsers should turn off hints when users chose to turn off Javascript
  - If the user has turned off Javascript, that means that the JS-based
    equivalent active fingerprinting vectors have been disabled. Since that
    could have been the user's intention when turning off scripting, browsers
    should respect that and similarly turn off Client Hints.
* Browsers can choose to omit or to lie about certain Client Hints to increase
  their users' privacy
  - Browsers are free to take privacy-enhancing heuristics into account when
    deciding to respect the server's opt-in to receive them. Similar heuristics
    can also be used when deciding what values they should be sending.
* Accept-CH-Lifetime persistence should not outlast other origin state
  - Browsers typically provide their users with means to clear state regarding
    certain origins: for example, delete the cache for an origin or its
    cookies. When users take such an action, it is likely that they want to get
    rid of all implicit state the browser may hold regarding that origin. Since
    the `Accept-CH-Lifetime` opt-in hold origin state in the browser for a
    period of predetermined time, its associated state needs to be similarly
    evicted when such user actions are taken.

# Conclusion

Client hints is a powerful content negotiation mechanism that enables us to
adapt the content to the user's needs without compromising their privacy. It
does that through the use of server opt-in, to guaranty that access to the
information requires active and tracable action on the server's side.  As such,
the mechanism does not increase the web's current active fingerprinting
surface.

The Client Hints infrastructure can be further used to reduce the web's passive
fingerprinting surface, by converting common use-cases for today's passive
fingerprinting vectors (e.g. the `User-Agent` string) into Client Hints which
require a specific opt-in.
