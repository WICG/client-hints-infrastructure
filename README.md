Client Hints is collection of HTTP and user-agent features that enables
privacy-preserving, proactive content negotiation with an explicit third-party
delegation mechanism:

<!--
I'm having trouble disentangling the first two bullet points here. The thrust of both of them seems to be a description of the flow: hints must be specifically asked for by the 1P server in order to be sent by the client (who has ultimate control over who gets what).

You break this flow in two to talk about the two different layers (HTTP/Fetch+HTML), but the only thing the second paragraph really adds, re: how it works *on the web,* is that hints must not be sent to insecure origins.

Also jammed in there is the idea that information that's currently advertised to everyone, on the web, could be hidden within this flow – but that seems like a separate idea about how these mechanisms could be USED, and not a description of the infrastructure itself. At least maybe break that out into a separate bullet?

-->
* Proactive content negotiation at the HTTP layer (defined in the
  [IETF I-D](https://httpwg.org/http-extensions/client-hints.html))
  enables servers to request delivery of specific hints, in order to enable optimized and
  automated selection of resources based on a user's device, conditions, and
  preferences, and lets clients decide which hint requests they want to grant,
  with per-hint and per-origin granularity.
* The origin-scoped, opt-in nature of negotiation (to be defined as part of the
  [HTML](https://github.com/whatwg/html/pull/3774) and
  [Fetch](https://github.com/whatwg/fetch/pull/773) specifications (<!-- aren't “origin-scoped” and “opt-in” intrinsic to how the HTTP layer works? those limits seem to fall out of that spec, not these ones -->)) enables
  clients to advertise requested hint data (e.g. user agent and device
  characteristics) selectively <!-- "enables to only" sounded weird to me, as
  "enabling" should grant new capabilities, but "only" connotes restriction --> to secure-transport origins <!-- kind of makes it sound like this is the only limiting factor... -->, instead of appending such data on every outgoing
  request.
* Origin opt-in applies to same-origin assets only and delivery to third party
  origins is subject to explicit first party delegation via Feature Policy <!-- dunno whether you want to link to the FP spec, or the bit of the Fetch spec that ties into it, here -- but you should link to something -->,
  enabling tight control over which third party origins can access requested
  hint data.

The goal of Client Hints is to **reduce active fingerprinting** on the web <!-- maybe a few words here about how, as this doesn't really follow from what we've established so far. "...on the web by giving clients full visibility and control over what data they share with which servers, while also..." --> while
**enabling scalable <!-- Nitpick: I'm a little confused by what you mean by scalable here. Adding-hints-isn't-a-problem-for-spec-designers or adapting-content-isn't-a-problem-for-authors? Both? --> and privacy-preserving content adaptation** between client
and server, via a standardized set of content negotiation primitives at the
HTTP and user agent levels.

This document outlines the Client Hints infrastructure, explains it at a high
level and points to the various specification and draft proposals in which it
is officially defined. It does not describe the various features and hints
which rely on this infrastructure <!-- after reading this all through... this is not really an explainer like https://w3ctag.github.io/explainers. Instead, it does two jobs: 1) explain the mechanisms at a fairly detailed level, and how they fit together 2) make a strong, coherent argument about the privacy implications. It doesn't really say much about the use cases.  Maybe I'll try to write up a separate doc... -->. They will be defined in their respective
specifications.

# The Client Hints infrastructure

How can servers opt into receiving hints from the client? How should clients
send those hints? And how should that be handled across origins?

## Opt-in mechanism

In order to receive Client Hints, servers must opt-in by using the HTTP response headers described
in the sections below. For security and privacy reasons, the opt-in must be
received on the response of the top-level navigation, over a secure connection.
Servers can similarly opt-in by using the headers' HTML equivalents, the `<meta>`
HTML tag and its `http-equiv` attribute.

### `Accept-CH`

The `Accept-CH` header enables servers to request specific hints from the
browser. The header's value is a comma separated list, where each value in that
list represents a request header hint that the server is interested in
receiving.

When an `Accept-CH` opt-in is received on the top-level navigation, same-origin
subresource requests on that page will <!-- clients still have leeway here -- is it worth being explicit about that by saying "may" or something, here? --> receive the requested hints. 

#### Example

If a server's response to a navigation request includes the `Accept-CH:
foo, bar` header, same-origin subresource requests on the page will include the
`Sec-Foo: foo-value` and `Sec-Bar: bar-value` request headers.

### `Accept-CH-Lifetime`

In order to enable delivery of hints for future navigation requests to the
same origin (not just subresource requests) servers can use the `Accept-CH-Lifetime`
header to communicate an integer value, which represents the number of seconds,
indicating a preference for how long this policy should be kept by the
client.

#### Example

If the server's response to the top-level navigation request includes the `Accept-CH:
foo, bar` and `Accept-CH-Lifetime: 3600` headers, same-origin requests *to that origin*
will <!-- this clause needs to specify that it's the client doing the requesting, and again... maybe make client's ultimate control about whether to send hints / no guarantees for servers explciit --> include the `Sec-Foo: foo-value` and `Sec-Bar: bar-value` request headers
for the next hour – on both subresource requests and navigation requests.

## Same-Origin Policy

There are two important restrictions on the opt-in mechanism description above:

- opt-ins will only be granted when the opt-in headers headers are received with a navigation resource
- after the opt-in, hints will only be sent with same-origin requests
<!-- broke this up into what I thought you meant... -->

Why are these limitations important?

As mentioned before, we don't want the mechanism to be used to increase
fingerprinting on the web.  The information that Client Hints provide should
generally be available through Javascript APIs (see the 
[Privacy Considerations](#privacy-considerations) section for more details).

That means that for *active* resources (e.g. HTML), Client Hints does not
increase the active fingerprinting surface. Servers serving active resources can already
exfiltrate the data that hints provide, via scripts, styles or certain elements (e.g. `<img
srcset>`).  Client Hints only provides servers with a more convenient and
performant way to do get that information, and insures that authors and user agents
have ultimate control over which servers get what information. <!-- changed the meaning here, but I think this is a stronger argument! -->

Client Hints also allow this information to be collected on *passive* sub-resources (e.g.
images). We don't want third parties to be able to exfiltrate data about the client/user without explicit permission from the first party. And we certainly don't want them to be able to exfiltrate it for the entire first-party origin,
beyond the lifetime of the current navigation.

Therefore, by default, Client Hints opt-in is only valid when delivered on
top-level navigation requests, and, by default, only applies only to same-origin resources. Cross-origin requests must only receive hints when explicit permission is given by the first-party origin.

## Cross-origin hint delegation

Why do we need cross-origin Client Hints?

As the purpose of Client Hints is to enable content negotiation at scale, and
as many optimization services are offered over different origins than the main
page's origin (e.g., CDNs, resource-specific hosting services, or dedicated, resource-specific, first-party sub-domains), cross-origin support is a vital part of Client Hints.

In order to support these use-cases, we have defined delegation of Client Hints
to specific cross-origin hosts, using Feature Policy.

Servers can opt-in to such delegation by applying `ch-` prefixed feature policies for
the desired hints.

### Example

A first-party server sending the following header `Feature-Policy: ch-example foo.com
bar.com; ch-example-2 foobar.org` will delegate the `example` hint to the
"foo.com" and "bar.com" origins and the `example-2` hint to the "foobar.org" origin.  So, the client would know that it had explicit permission from the first party to send these hints to these third parties, so that these third parties could perform content
adaptation based on them.

### Privacy implications

Why is it privacy-safe for pages to delegate hints to certain third party
origins?

Since we're treating Client Hints as an active fingerprinting equivalent, we
are comfortable with the information it exposes to first-party servers, as the same
information is already freely available in the equivalent Javascript APIs.
Similarly, third party delegation is safe because first parties are already able to use
other means (such as link decoration) to achieve the same information sharing
with third parties, in less convenient and performant ways.

In short, first parties already have the power to share information about the client with third parties. Third party delegation of Client Hints provides a cleaner pathway for that sharing, and ensures that first parties, and ultimately clients, are aware and in control of what is being shared with who.

## `Sec-` prefix

Adding new request headers increases the risk that legacy server systems
already use those values for a different purpose. Changing the request header
values that such legacy systems receive may result in server bugs.

While that risk is significantly mitigated by the opt-in mechanisms of Client
Hints (as servers would be opting in to get Client Hints headers), we feel
it is required to mitigate it even further. So, Client Hints request headers
should be preceded by the `Sec-` prefix.

The `Sec-` prefix also ensures that these headers are only generated by the
browser and not added by potentially-malicious developers. That gives servers
further guarantees when processing those headers. It may also enable us to
simplify the related Fetch processing model, as it clearly indicates that these
are headers that were added by the user agent.

## Caching considerations

When adapting content to specific Client Hints request headers, servers should
add the `Vary` header to their responses, with a value of each Client Hint header used, in order to indicate such adaptation to caches,
and make sure that Client-Hint-negotiated resources are not cached using only their URL as the
cache key.

This is also the reason that each Client Hint is represented using a separate
header. Ensuring that each hint has its own header reduces cache variance in responses that may rely on
some hints, but not others.

# Security and privacy considerations

There are a few key mechanisms we already discussed that are part of the Client
Hints infrastructure, which enable secure and privacy-preserving deployment
of Client Hints:

* Server opt-ins must be delivered on a top-level navigation request, over a
  secure connection.
* Hints are only delivered with same-origin requests, over a secure connection.
* If the first party wants hints to be delivered to certain third-party hosts, the first-party content
  can explicitly delegate specific hints to specific hostnames.
* Hints are `Sec-` prefixed, to provide servers with more confidence regarding
  the values they deliver, as well as to avoid legacy server bugs.

Beyond that, when implementing Client Hints, browsers should make sure that
certain privacy-related precautions are being taken:

* Client Hint features should not be shipped unless there is a Javascript-based
  equivalent API, which enables developers access to the same data and is
  deemed privacy-safe to ship
  - This follows Extensible Web principles &mdash; we want the
    shipped features to be polyfillable (Even if `Sec-` headers cannot
    be added by developers)
  - As discussed above, from a privacy perspective, we consider Client Hints to
    be a potential active fingerprinting vector equivalent. Therefore, it is
    only safe to ship it with hints that provide information which is already
    available through other active fingerprinting means, such as a Javascript
    API.
* Browsers should turn off hints when users choose to turn off Javascript
  - If the user has turned off Javascript, Javascript-based
    active fingerprinting vectors have been disabled. Since that
    could have been the user's intention when turning off scripting, browsers
    should similarly turn off Client Hints.
* Browsers can choose to omit or to lie about certain Client Hints to increase
  their users' privacy
  - Browsers are free to take privacy-enhancing heuristics into account when
    deciding to respect the server's opt-in to receive them. Similar heuristics
    can also be used when deciding what values to send.
* Accept-CH-Lifetime persistence should not outlast other types of origin state
  - Browsers typically provide their users with means to clear state regarding
    certain origins: for example, ways to delete the cache for an origin or clear its
    cookies. When users take such an action, it is likely that they want to get
    rid of all of the implicit state that the browser may hold regarding that origin. Since
    the `Accept-CH-Lifetime` opt-in holds origin state in the browser for a
    period of predetermined time, its associated state needs to be similarly
    evicted when users take such actions.

# Motivation and trade-offs

## Proactive content negotiation and its benefits

<!-- not clear who "we" is here. servers? authors? spec designers? -->
When we need to choose a solution that will provide alternative resources to
the user's browser based on various factors, we are faced with a design
dillema: either we can provide the browser with a list of all potential URLs and let
the browser choose the best one, or we can use **content negotiation** and pick
the best-fit resource variant on the server.

The former option certainly has its place, and it is used successfully across
the web in examples like `<picture>`, `srcset`, `<video>`, etc.

At the same time, there are some use-cases where it is not sufficient.
Transformation and adaptation of the page's subresources in a manner that is
independent from the page's markup can result in more scalable and dedicated <!-- what do you mean, "dedicated"? maybe "focused"? or "more scalable solutions that are independent of and not entagled with markup" ? -->
solutions, that don't have to be assimilated into markup related workflows. 

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
   - Data-saving preferences
   - Preferred language
* Network conditions
   - RTT
   - Effective bandwidth

The list above is not necessarily exhaustive, but it gives us an idea as to
why simply providing a browser with a (possibly very long) list of resources which have been adapted across multiple dimensions of variability may not be practical, at least in some cases.

## An opt-in solution and its tradeoffs

Client Hints requires that servers explicitly advertise and request sets of
hints that they would like to receive. This makes such requests explicit and does not enable
passive fingerprinting using those hints, which is one of the key and guiding
requirements for Client Hints.

Indiscriminately broadcasting user data carries other downsides, as well. Exposing all details and adaptation dimensions to all servers runs
a risk of bloating request headers. There are many potential details that can
be useful for content negotiation, and we expect that list to grow over time.
Sending all of the hints all the time can quickly result in bloat, and make
requests significantly larger than they should be. To avoid that, it is more
efficient for servers to specifically request the headers that they would take into
account, and for clients to send only these hints with outgoing requests.

Unfortunately, that decision doesn't come without tradeoffs. Client Hints as an
opt-in mechanism currently means that content adaptation of the initial
navigation request on the very first view is not possible, at least not without
hacks.

At the same time, for features which are critical for content negotiation of
navigation requests, browsers may choose to send hints regardless of a server
opt-in, when they deem that the information in question doesn't
increase the passive fingerprinting surface.

_Note:_ there may be other, out-of-band, opt-in mechanisms in the future that
could enable delivery of hints on first navigation to new origins, such as
Origin Policy.

## Privacy enhancing content negotiation

Content negotiation is typically viewed as a mechanism that enables
passive fingerprinting, by adding different bits of data to different user's
requests, by default, and enabling servers to use those differences to tell users apart
without leaving any trace of that activity.

Client Hints’ opt-in mechanism enables us to avoid this problem, as
servers need to tell the browsers which information they need, making any such
fingerprinting user-detectable.

But, Client Hints can also enable us to do more than that for user privacy, by layering the opt-in mechanism on to passive-fingerprinting-enabling content-negotiation headers that are already in common use (e.g. the
`User-Agent` or `Accept-Language` request headers).
Restricting the amount of user-identifying information that browsers send by default would effectively reduce the passive fingerprinting surface on the web,
and Client Hints’ opt-in mechanism would enable browsers to keep closer tabs on entities that use that information
for seemingly nefarious reasons.

# Conclusion

Client Hints provides a powerful content negotiation mechanism that enables us to
adapt content to users' needs without compromising their privacy. It
does that by requiring server opt-in, which guarantees that access to the
information requires active and tracable action on the server's side.  As such,
the mechanism does not increase the web's current active fingerprinting
surface.

The Client Hints infrastructure can be further used to reduce the web's passive
fingerprinting surface, by converting common use-cases for today's passive
fingerprinting vectors (e.g. the `User-Agent` string) into Client Hints which
require a specific opt-in.
