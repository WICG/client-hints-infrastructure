<base target="_blank">

The specification of Client Hints is divided between different standardization
bodies and different standards.  To make matters worse, since the feature does
not yet have multiple browser implementation support, some of the specification
for the feature lives in PRs on the relevant standards.

This document's purpose is to make it clearer how all those pieces fit
together, and be used as a pointer for folks that want to get a better
understanding of the current specification situation.

# High-level

The specification of the Client Hints infrastructure and features is divided to
the following specifications and proposals:

* IETF [Client Hints Internet-Draft](https://httpwg.org/http-extensions/client-hints.html)
   - Defines the Client Hints infrastructure headers (`Accept-CH` and
     `Accept-CH-Lifetime`), their format and how servers should behave with
     regards to them.
* WHATWG HTML specification ([PR](https://github.com/whatwg/html/pull/3774))
   - Defines the web concepts related to Accept-CH and Accept-CH-Lifetime,
     their processing on top-level navigation responses as well as defines the
     related caches that make sure origin persistency is maintained.
   - Defines image related Client-Hint features: `DPR`, `Viewport-Width`,
     `Width` and the related `Content-DPR` response header.
* WHATWG Fetch specification ([PR](https://github.com/whatwg/fetch/pull/773))
   - Integrates the HTML web concepts to Fetch's algorithms to make sure that
     opted-in hints are added to requests for same-origin or delegated-to
     cross-origin requests.
   - Makes sure hints are removed from not delegated-to cross-origin requests
     after redirections.
   - Defines all `Sec-` prefixed requests as CORS safe.
   - Defines the concept of image response density, to support the
     `Content-DPR` response header.
* W3C Feature Policy specification
   - Defines infrastructure that enables 3rd party delegation for Client Hints.


# IETF 

The [Client Hints Internet-Draft](https://httpwg.org/http-extensions/client-hints.html) is
maintained as part of the HTTPWG at the IETF.  It defines the various headers
used by the Client Hints infrastructure and provides formal requirements on
implementing client and servers.  As it is defined in terms of generic clients,
rather than browsers or user agents, it does not provide more-specific and
web-related processing models.


# WHATWG

Much of the web-related processing model is defined in terms of WHATWG
specifications, and specifically, Fetch and HTML.  Because of the WHATWG's
requirement for multiple implementation interest, those definitions currently
live as spec PRs.

This section describes those PRs and enables easy and annotated inspection of
their contents.

## Fetch

The current state of the Client Hints specification in Fetch is admittedly
confusing.  Some bits of the infrastructure definition, as well as definitions
of features that rely on the infrastructure, are already part of the standard.
At the same time, those definitions are mostly out-of-date and not aligned with
the current implementation (e.g. do not include the `Accept-CH-Lifetime`
mechanism).  The latest defintions, aligning the specification with the
implementation, are defined in
[Fetch#773](https://github.com/whatwg/fetch/pull/773). A diff of the PR can be
previewed [here](https://whatpr.org/fetch/773/939817c...56c7c7b.html).

Let's examine the changes that the PR makes to the standard.

### [Fetch integration](https://whatpr.org/fetch/773/939817c...56c7c7b.html#concept-fetch)

This change integrates the Client Hints logic into the fetch processing steps.
It clones the client-hints set from the client's global object onto the
request, and adds hints to requests, while making sure that they are allowed to
be added based on the set Feature Policy.

### [Post-redirect potential header removal](https://whatpr.org/fetch/773/939817c...56c7c7b.html#concept-http-redirect-fetch)

This step makes sure that cross-origin redirects are not adding Client Hints
headers if the set Feature Policy does not allow them to do that. It does that
by removing those headers from such redirects.

### [CORS safe-list](https://whatpr.org/fetch/773/939817c...56c7c7b.html#cors-safelisted-request-header)

This change to the safe-list makes sure that requests which start with `Sec-`
are considered safe and do not trigger preflights. As such requests are
necessarily created by the browser, cannot be forged by the user and are likely
to be safe for server implementations, we believe that it is a safe choice.

### client hints set

#### [client hints set renaming](https://whatpr.org/fetch/773/939817c...56c7c7b.html#concept-request-client-hints-list)

As part of the related HTML PR, we've changed the previous client hints list to
client hints set. This aligns with that change.

#### [client hints set definition](https://whatpr.org/fetch/773/939817c...56c7c7b.html#concept-fetch)

This defines the client hints set concept as well as a list of its valid
values, which are the various features relying on the Client Hints
infrastructure.

### Image density

#### [Image density response concept](https://whatpr.org/fetch/773/939817c...56c7c7b.html#concept-response-image-density)

This defines the concept of image density and attaches it to a reponse.

#### [Image density setting](https://whatpr.org/fetch/773/939817c...56c7c7b.html#ref-for-concept-response-header-list①⑤)

This change sets the image density of the Response based on the response's
Content-DPR header.

## HTML

### [Initializing a new Document object](https://whatpr.org/html/3774/e32a6f8...44b682a/browsing-the-web.html#initialise-the-document-object)

* Initializes Client Hints set on the Document based on its environment
  serttings object's origin. 
* Parse `Accept-CH` if we're in a secure context and add the results to the
  document's client hints set.
* If we are in a secure context and the navigation is a top-level navigation,
  parse `Accept-CH-Lifetime` and add a new entry to Accept-CH-Lifetime cache.

### [Accept-CH-Lifetime cache](https://whatpr.org/html/3774/e32a6f8...44b682a/offline.html#accept-ch-lifetime-cache)

This defines the Accept-CH-Lifetime per-origin cache, and its related
processing models.

### [Image-related Client Hints headers](https://whatpr.org/html/3774/e32a6f8...44b682a/images.html#image-related-client-hints-request-headers)

This defines the Client Hints image-related request headers: `DPR`,
`Viewport-Width` and `Width`, as well as the corresponding response header,
`Content-DPR`.

### http-equiv

#### [http-equiv attributes](https://whatpr.org/html/3774/e32a6f8...44b682a/indices.html#attributes-3)

This references the `accept-ch` and `accept-ch-lifetime` `http-equiv` attribute
values.

#### [Pragma directives](https://whatpr.org/html/3774/e32a6f8...44b682a/semantics.html#pragma-directives)

Defines the `Accept-CH` and `Accept-CH-Lifetime` pragma directives and their
processing models.

### client hints set definitions

#### [Environment settings object's client hints set](https://whatpr.org/html/3774/e32a6f8...44b682a/webappapis.html#concept-settings-object-client-hints-set)

Defines the client hints set on the environment settings object.

#### [Window's client hints set](https://whatpr.org/html/3774/e32a6f8...44b682a/window-object.html#script-settings-for-window-objects:concept-settings-object-client-hints-set)

Defines the processing for client hints set when a window environment settings
object is created. 

#### [WorkerGlobalScope's client hints set](https://whatpr.org/html/3774/e32a6f8...44b682a/workers.html#concept-workerglobalscope-client-hints-set)

Defines the client hints set on WorkerGlobalScope.

#### [Worker's client hints set](https://whatpr.org/html/3774/e32a6f8...44b682a/workers.html#script-settings-for-workers:concept-settings-object-client-hints-set)

Defines the processing for client hints set when a worker environment settings
object is created. 

#### [Docuemnt's client hints set](https://whatpr.org/html/3774/e32a6f8...44b682a/dom.html#concept-document-client-hints-set)

Defines the document's client hints set concept.

### Definitions from other specifications

####
[Infrastructure](https://whatpr.org/html/3774/e32a6f8...44b682a/infrastructure.html)

Adds references to "field-name", "delta-seconds" and "client hints set".

####
[References](https://whatpr.org/html/3774/e32a6f8...44b682a/references.html)

Adds references to the Client Hints I-D and to Structured Headers.

# W3C

## Feature Policy

Feature Policy added the
[Should request be allowed to use feature?](https://w3c.github.io/webappsec-feature-policy/#should-request-be-allowed-to-use-feature)
algorithm which is the infrastructure that enables the use of Feature Policy as
a way to delegate hints to third parties.
