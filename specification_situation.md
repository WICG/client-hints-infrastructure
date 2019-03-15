The specification of Client Hints is divided between different standardization
bodies and different standards.  To make matters worse, since the feature does
not yet have multiple browser implementation support, some of the specification
for the feature lives in PRs on the relevant standards.

This document's purpose is to make it clearer how all those pieces fit
together, and be used as a pointer for folks that want to get a better
understanding of the current specification situation.

# IETF

The [Client Hints
Internet-Draft](https://httpwg.org/http-extensions/client-hints.html) is
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

The current state of the Client Hints specification in Fetch is admittedly confusing.
Some bits of the infrastructure definition, as well as definitions of features that rely on the infrastructure, are already part of the standard. At the same time, those definitions are mostly out-of-date and not aligned with the current implementation (e.g. do not include the `Accept-CH-Lifetime` mechanism).
The latest defintions, aligning the specification with the implementation, are defined in [Fetch#773](https://github.com/whatwg/fetch/pull/773). A diff of the PR can be previewed [here](https://whatpr.org/fetch/773/939817c...a50febc.html).

Let's examine the changes that the PR makes to the standard.

### [CORS safe-list](https://whatpr.org/fetch/773/939817c...a50febc.html#cors-safelisted-request-header)

The changes to the safe-list below make sure that requests which start with `Sec-` are considered safe and do not trigger preflights. As such requests are necessarily created by the browser, cannot be forged by the user and are likely to be safe for server implementations, we believe that it is a safe choice.

### [client-hints set renaming](https://whatpr.org/fetch/773/939817c...a50febc.html#concept-request-client-hints-list)

As part of the related HTML PR, we've changed the previous client hints list to client hints set. This aligns with that change.

### [Image density response concept](https://whatpr.org/fetch/773/939817c...a50febc.html#concept-response-image-density)

This defines the concept of image density and attaches it to a reponse.

### [client hints set definition](https://whatpr.org/fetch/773/939817c...a50febc.html#concept-fetch)

This defines the client hints set concept as well as a list of its valid values, which are the various features relying on the Client Hints infrastructure.

### [Fetch integration](https://whatpr.org/fetch/773/939817c...a50febc.html#concept-fetch)

This change integrates the Client Hints logic into the fetch processing steps. It clones the client-hints set from the client's global object onto the request, and adds hints to requests, while making sure that they are allowed to be added based on the set Feature Policy.

### [Image density setting](https://whatpr.org/fetch/773/939817c...a50febc.html#ref-for-concept-response-header-list①⑤)

This change sets the image density of the Response based on the response's Content-DPR header.

### [Post-redirect potential header removal](https://whatpr.org/fetch/773/939817c...a50febc.html#concept-http-redirect-fetch)

This step makes sure that cross-origin redirects are not adding Client Hints headers if the set Feature Policy does not allow them to do that. It does that by removing those headers from such redirects.

<!--
<script>
/*
const fetch_changes = [
  { "summary": "CORS safe-list",
    "description": "The changes to the safe-list below make sure that requests which start with `Sec-` are considered safe and do not trigger preflights. As such requests are necessarily created by the browser, cannot be forged by the user and are likely to be safe for server implementations, we believe that it is a safe choice.",
    "url": "#cors-safelisted-request-header",
  },
  { "summary": "client-hints set renaming",
    "description": "As part of the related HTML PR, we've changed the previous client hints list to client hints set. This aligns with that change.",
    "url": "#concept-request-client-hints-list",
  },
  { "summary": "Image density response concept",
    "description": "This defines the concept of image density and attaches it to a reponse.",
    "url": "#concept-response-image-density",
  },
  { "summary": "client hints set definition",
    "description": "This defines the client hints set concept as well as a list of its valid values, which are the various features relying on the Client Hints infrastructure.",
    "url": "#client-hints-list",
  },
  { "summary": "Fetch integration",
    "description": "This change integrates the Client Hints logic into the fetch processing steps. It clones the client-hints set from the client's global object onto the request, and adds hints to requests, while making sure that they are allowed to be added based on the set Feature Policy.",
    "url": "#concept-fetch",
  },
  { "summary": "Image density setting",
    "description": "This change sets the image density of the Response based on the response's Content-DPR header.",
    "url": "#ref-for-concept-response-header-list①⑤",
  },
  { "summary": "Post-redirect potential header removal",
    "description": "This step makes sure that cross-origin redirects are not adding Client Hints headers if the set Feature Policy does not allow them to do that. It does that by removing those headers from such redirects.",
    "url": "#concept-http-redirect-fetch",
  },
];
(()=> {
  // This is something I did after changing the format of the presented links, their markup and the way they are presented multiple times. Once presentation will settle, it may make sense that have that data be in HTML. Or not. We'll see.
  const base_url = "https://whatpr.org/fetch/773/939817c...a50febc.html";
  const list = document.getElementById("fetch_changes_list");
  for (const change of fetch_changes) {
    const html = "<div><a href=" + base_url + change["url"] + "><h3>" + change["summary"] + "</h3></a><p>" + change["description"] + "</p></div>";
    const template = document.createElement("template")
    template.innerHTML = html;
    const element = template.content || template;
    list.appendChild(element.firstChild);
  }
})();
*/
</script>
-->

## HTML


# W3C

## Feature Policy
