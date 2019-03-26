Client Hints is collection of HTTP and user-agent features that enables
privacy-preserving proactive content negotiation with explicit third party
delegation mechanism:

* Proactive content negotiation at the HTTP layer (defined in the
  [IETF I-D](https://httpwg.org/http-extensions/client-hints.html))
  enables servers to request delivery of specific hints to enable optimized and
  automated selection of resources based on the user's device conditions and
  preferences, and enables client to decide which hints requests they respect
  with a per-origin granularity.  
* Origin scoped, opt-in nature of negotiation (to be defined as part of the
  [HTML](https://github.com/whatwg/html/pull/3774) and
  [Fetch](https://github.com/whatwg/fetch/pull/773) specifications) enables the
  client to only advertise requested hint data (e.g. user agent and device
  characteristics) to select secure-transport origins, instead of appending
  such data on every outgoing request.  
* Origin opt-in applies to same-origin assets only and delivery to third party
  origins is subject to explicit first party delegation via Feature Policy,
  enabling tight control over which third party origins can access requested
  hint data.

The goal of Client Hints is to **reduce active fingerprinting** on the web while
**enabling scalable and privacy preserving content adaptation** between client
and server, via a standardized set of content negotiation primitives at the
HTTP and user agent levels.

This document outlines the Client Hints infrastructure, explains it at a higher
level and points to the various specification and draft proposals in which it
is officially defined. It does not describe the various features and hints
which rely on the infrastructure. They will be defined in their respective
specifications.

# The Client Hints infrastructure

How can servers opt into receiving hints from the client? How should client
send those hints? And how should that be handled across origins?

## Opt-in mechanism

Servers can opt-in to Client Hints by using the HTTP response headers described
in the sections below. For security and privacy reasons, the opt-in must be
received on the response of the top-level navigation, over a secure connection.
They can similarly opt-in by using the headers' HTML equivalents, the `<meta>`
HTML tag and its `http-equiv` attribute.

### `Accept-CH`

The `Accept-CH` header enables servers to request specific hints from the
browser. The header's value is a comma separated list, where each value in that
list represents a request header hint that the server is interested in
receiving.

When an `Accept-CH` opt-in is received on the top-level navigation, same-origin
subresource requests on that page will receive the requested hints. 

#### Example

If the server's response to the navigation request includes the `Accept-CH:
foo, bar` header, same-origin subresource requests on the page will include the
`Sec-Foo: foo-value` and `Sec-Bar: bar-value` request headers.

### `Accept-CH-Lifetime`

In order to enable delivery of hints for future navigation requests to the
origin, on top on subresource requests, the server can use `Accept-CH-Lifetime`
header to communicate an integer value, which represents the number of seconds,
indicating a preference for how long this policy should be kept by the
browser.

#### Example

If the server's response to the navigation request includes the `Accept-CH:
foo, bar` and `Accept-CH-Lifetime: 3600`, same-origin requests *on that origin*
will include the `Sec-Foo: foo-value` and `Sec-Bar: bar-value` request headers
for the next hour, including the navigation requests of future navigations.

## Same Origin Policy

The opt-in mechanism description above includes the fact that the opt-in is
only applied to same-origin requests, and only when it is received on a
navigation resource. Why is that important?

As mentioned before, we don't want the mechanism to be used to increase
fingerprinting on the web.  The hints that Client Hints provides should
generally be available through Javascript APIs (see the 
[Privacy Considerations](#privacy-considerations) section for more details).

That means that for *active* resources (e.g. HTML), Client Hints does not
increase the active fingerprinting surface. Those resources can already
exfiltrate that data, via scripts, styles or certain elements (e.g. `<img
srcset>`).  Client Hints only provides them with a more convenient and
performant way to do that, when that data is needed for content negotiation
purposes.

But, that also means that we don't want passive cross-origin subresources (e.g.
images) to be able to exfiltrate the same data without explicit permission, and
we certainly don't want them to be able to exfiltrate it for the entire origin,
beyond the lifetime of the current navigation.

Therefore, by default, Client Hints opt-in is only valid when delivered on
top-level navigation requests, and applies only to same-origin resources by
default. Hint access to cross-origin requests requires explicit delegation by
the first-party origin.

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
information is already freely available in the equivalent Javascript APIs.
Similarly, third party delegation is safe because pages are already able to use
other means, such as link decoration, to achieve the same information sharing
with third parties, only in less performant ways.

## `Sec-` prefix

Adding new request headers increases the risk that legacy server systems
already use those values for a different purpose. Changing the request header
values such legacy systems receive may result in server bugs.

While that risk is significantly mitigated by the opt-in mechanisms of Client
Hints (as those same servers would be opting in to get those headers), we feel
it is required to mitigate it even further. So, Client Hints request headers
should be preceded by `Sec-` prefix.

The `Sec-` prefix also ensures that these headers are only generated by the
browser and not added by potentially-malicious developers. That gives servers
further guarantees when processing those headers. It may also enable us to
simplify the related Fetch processing model, as it clearly indicates that these
are headers that were added by the user agent.

## Caching considerations

When adapting content to specific client hints request headers, servers should
add the `Vary` header to their responses to indicate such adaptation to caches,
and make sure that such resources are not cached using only their URL as the
cache key.

This is also the reason that each Client Hint is represented using a separate
header, in order to reduce cache variance in responses that may rely on
some hints, but not others.

# Security and privacy considerations

There are a few key mechanisms we already discussed that are part of the Client
Hints infrastructure, and which enable secure and privacy preserving deployment
of Client Hints:

* Server opt-ins must be delivered on a top-level navigation request, over a
  secure connection.
* Hints are delivered only to same-origin requests, over a secure connection.
* If hints are required for certain third party hosts, the first-party content
  can explicitly delegate specific hints to specific servers.
* Hints are `Sec-` prefixed, to provide servers with more confidence regarding
  the values they deliver, as well as to avoid legacy server bugs.

Beyond that, when implementing Client Hints, browsers should make sure that
certain privacy related precautions are being taken:

* Client Hint features should not be shipped unless there is a Javascript-based
  equivalent API, which enables developers access to the same data and is
  deemed privacy-safe to ship
  - One reason for that is Extensible Web principles &mdash; we want the
    shipped features to be somewhat polyfillable (Even if `Sec-` headers cannot
    be added by developers)
  - As discussed above, from a privacy perspective, we consider Client-Hints to
    be a potential active fingerprinting vector equivalent. Therefore, it is
    only safe to ship with it hints which provide information that is already
    available through other active fingerprinting means, such as a Javascript
    API.
* Browsers should turn off hints when users choose to turn off Javascript
  - If the user has turned off Javascript, that means that the Javascript-based
    equivalent active fingerprinting vectors have been disabled. Since that
    could have been the user's intention when turning off scripting, browsers
    should respect that and similarly turn off Client Hints.
* Browsers can choose to omit or to lie about certain Client Hints to increase
  their users' privacy
  - Browsers are free to take privacy-enhancing heuristics into account when
    deciding to respect the server's opt-in to receive them. Similar heuristics
    can also be used when deciding what values they should be sending.
* Accept-CH-Lifetime persistence should not outlast other types of origin state
  - Browsers typically provide their users with means to clear state regarding
    certain origins: for example, delete the cache for an origin or its
    cookies. When users take such an action, it is likely that they want to get
    rid of all implicit state the browser may hold regarding that origin. Since
    the `Accept-CH-Lifetime` opt-in hold origin state in the browser for a
    period of predetermined time, its associated state needs to be similarly
    evicted when such user actions are taken.

# Motivation and trade-offs

## Proactive content negotiation and its benefits

When we need to choose a solution that will provide alternative resources to
the user's browser based on various factors, we are faced with a design
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

## An opt-in solution and its tradeoffs

Client Hints requires that the server explicitly advertise and request a set of
hints it would like to receive. This makes such requests explicit and does not enable
passive fingerprinting using those hints, which is one of the key and guiding
requirements for Client Hints.

Beyond that, exposing all details and adaptation dimensions to all servers runs
a risk of bloating request headers. There are many potential details that can
be useful for content negotiation, and we expect that list to grow over time.
Sending all the hints all the time can quickly result in bloat, and make
requests significantly larger than they should be. To avoid that, it is more
efficient for servers to specifically request the headers they would take into
account, and send only these hints with upgoing requests.

Unfortunately, that decision doesn't come without tradeoffs. Client Hints as an
opt-in mechanism currently means that content adaptation of the initial
navigation request on the very first view is not possible, at least not without
hacks.

At the same time, for features which are critical for content negotiation of
navigation requests, browsers may choose to send them regardless of a server
opt-in, in case it deems exposing that information as something that doesn't
increase the passive fingerprinting surface.

_Note:_ there may be other, out of band, opt-in mechanisms in the future that
could enable delivery of hints on first navigation to new origins, such as
Origin Policy.

## Privacy enhancing content negotiation

Content negotiation is typically viewed as a mechanism that enables
passive fingerprinting, by adding different bits of data to different user's
requests by default, and enabling servers to use that to tell users apart
without leaving any trace of that activity.

The Client Hints infrastructure and its opt-in enables us to avoid that, as
servers need to tell the browsers which information they need, making any such
fingerprinting use detectable.

But, Client Hints can also enable us to do more than that for user privacy, and
turn passive-fingerprinting-enabling content-negotiation mechanisms (e.g. The
`User-Agent` or `Accept-Language` request headers) into opt-in-only mechanisms.
That would effectively reduce the passive fingerprinting surface on the web,
and enable browser to keep closer tabs on entities that use that information
for seemingly nefarious reasons.

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
