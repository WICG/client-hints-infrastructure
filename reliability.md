# Client Hint Reliability

## Authors

* David Benjamin
* Aaron Tagliaboschi


## Introduction

[HTTP Client Hints](https://www.rfc-editor.org/rfc/rfc8942.html) can replace passive fingerprinting surfaces with server-requested (and [potentially deniable](https://github.com/bslassey/privacy-budget)) client headers. However, the client may have out-of-date information on the server preferences when it sends a request. On the first page load, the client may not know to send any hints at all. This document describes a pair of mechanisms to fix this:

1. an HTTP-header-based retry to ensure critical Client Hints are reliably available
1. a connection-level optimization to avoid the performance hit of a retry in most cases


### Goals

The design should:

* Ensure requests use up-to-date server preferences
* Avoid an extra network round-trip in the common case
* Be robust to a client declining to send a Client Hint (user preferences, etc.)
* Require minimal server complexity


### Non-Goals

This design does _not_ force the client to send a Client Hint. It still may not support the hint or choose not to send it. This design also only aims to avoid a round-trip cost _most of the time_. Round-trips are sometimes unavoidable.


## Critical-CH

Some Client Hints are optimizations, while others meaningfully change the page. For example, a site may use `Device-Memory` to serve simple and complex variants, and `Viewport-Width` for a server-side rendering optimization. If only the first request lacks `Device-Memory`, the site will jarringly switch versions between page loads. The server could try to detect this and self-redirect, but this will loop if the client declined to send the hint, or simply didn’t implement it.

We move retry to the client with a new response header, [`Critical-CH`](https://tools.ietf.org/html/draft-davidben-http-client-hint-reliability-00#section-3), with a list of _critical_ hints. If, after processing the `Accept-CH` header, the client would have sent a critical hint, it retries the request. Otherwise, it uses the response as-is. E.g, the initial request may be:

```
    GET / HTTP/1.1
    Host: example.com

    HTTP/1.1 200 OK
    Content-Type: text/html
    Accept-CH: Device-Memory, DPR, Viewport-Width
    Vary: Device-Memory, Viewport-Width
    Critical-CH: Device-Memory
    (Body has default complex version of page)
```


The client records the `Accept-CH` preferences. It would have sent each of these hints and `Device-Memory` is critical, so it retries the request and receives the simpler version of the page.

```
    GET / HTTP/1.1
    Host: example.com
    Device-Memory: 0.5
    DPR: 2
    Viewport-Width: 320

    HTTP/1.1 200 OK
    Content-Type: text/html
    Accept-CH: Device-Memory, DPR, Viewport-Width
    Vary: Device-Memory, Viewport-Width
    Critical-CH: Device-Memory
    (Body has simpler version of page)
```


## Connection-level settings

Fundamentally, any HTTP request using server information needs a round-trip to get that information. However, we can reuse the TLS handshake round-trip and send client hint preferences in the same flight as the TLS ServerHello, when TLS 1.3 starts encryption. This avoids a round-trip most of the time, but there are edge cases where a round-trip is unavoidable.

Here we describe one possible encoding. See below for other options. Note that web developers would not be directly interacting with these mechanisms. They would be implemented by server software, with the web developer configuring it somewhere.


### ACCEPT\_CH

In HTTP/2 and HTTP/3, the server can send auxiliary frames such as [SETTINGS](https://httpwg.org/specs/rfc7540.html#SETTINGS) with parameters for the connection. We define an [`ACCEPT_CH`](https://tools.ietf.org/html/draft-davidben-http-client-hint-reliability-00#section-4) frame with a list of (origin, accept-ch) tuples. When the client sends an HTTP request, if the client has received an `ACCEPT_CH` frame and the origin matches an entry in the list, it uses the matching server preferences. (See also [connection vs cache ordering](#connection-vs-cache-ordering).)


### Application-Layer Protocol Settings

While it is [possible](#accept_ch-in-half-rtt-data), in some modes, to send HTTP/2 frames with the ServerHello, this is not required or reliable. HTTP/2 and HTTP/3 clients do not wait for these frames before sending requests. We fix this by introducing an [Application-Layer Protocol Settings (ALPS)](https://tools.ietf.org/html/draft-vvv-tls-alps-00) extension for TLS 1.3 which lifts protocol-specific server settings into the EncryptedExtensions message. In [HTTP/2 and HTTP/3](https://tools.ietf.org/html/draft-vvv-httpbis-alps-00), we use them to carry the `ACCEPT_CH` frame and others.


### Example

A client that has never connected to a server before will pick up the Client Hint request with no additional round-trip cost:

```
    ClientHello
    + alps
                                         ServerHello
                                 EncryptedExtensions
         + alps=(https://example.com, Device-Memory)
                                                 ...
                                            Finished
    Finished
    GET / HTTP/2.0
    Host: example.com
    Device-Memory: 0.5
                                     HTTP/2.0 200 OK
                                 Vary: Device-Memory
                            Accept-CH: Device-Memory
                          Critical-CH: Device-Memory
```

Note that, although the server sends `Critical-CH`, the client will not retry because it already sent `Device-Memory`.

## Key scenarios

### First load with updated server

Sites running server software with `ACCEPT_CH` and ALPS support, Client Hints would be available in the first request as above.

### First load without updated server

Sites running older software can continue to use `Critical-CH` for Client Hint reliability, at a round-trip cost on the first page load. Once the server is updated to support the connection-level mechanisms, this round-trip cost will go away.

### Unsupported or declined hint

Suppose, in the above example, the client will not send `Device-Memory`. It may not support it, or decline it for privacy reasons. Although the `ACCEPT_CH` frame requests it, the client does not send it:

```
    GET / HTTP/2.0
    Host: example.com
```

The server responds as best it can without the hint and sends header-level preferences.

```
    HTTP/2.0 200 OK
    Vary: Device-Memory
    Accept-CH: Device-Memory
    Critical-CH: Device-Memory
```

The client evaluates the new `Accept-CH` header, determines it would not do anything different, and uses the response as-is. The user gets a consistent experience without accidental infinite redirect loops or extra round-trip costs.


### Non-critical hints

Some Client Hints may not be worth a round-trip. For instance, a page may use the `Viewport-Width` hint for some server-side rendering optimization. If the hint is missing, the result still renders correctly. On the first visit, the `ACCEPT_CH` frame will still usually provide the hint anyway. If, however, the page load hits an edge case where this does not work, the HTTP request is:

```
    GET / HTTP/1.1
    Host: example.com

    HTTP/1.1 200 OK
    Content-Type: text/html
    Vary: Viewport-Width
    Accept-CH: Viewport-Width
```

Although the page requests `Viewport-Width`, it is not critical, so the browser uses the response as-is. It remembers the `Viewport-Width` preference, so later page loads send the hint.


### Site changes

A site’s HTTP frontend and content are often separated. It may use a CDN or the site may just be a collection of files on the filesystem. This means the site’s `Accept-CH` preferences may be updated without dropping existing HTTP connections.

A client with an open HTTP connection would have both its `Accept-CH` cache and the connection-level `ACCEPT_CH` frame out-of-date. Without other information, it will send Client Hints based on the old preferences. Here, the `Critical-CH` mechanism restores reliability. This is an example where the connection-level optimization is not sufficient.

A client without an open HTTP connection would have outdated `Accept-CH` cache but see an up-to-date `ACCEPT_CH` frame when it connects for the next request. Depending on the order between the two (see [discussion](#connection-vs-cache-ordering)), this may avoid the round-trip or lean on `Critical-CH`.


## Detailed design discussion


### Why two mechanisms?

`Critical-CH` costs a round-trip, so making `ACCEPT_CH` + ALPS always work would seem preferable. However, this is not always possible:

* The site may be running older software and have difficulty adopting `ACCEPT_CH` + ALPS.
* If the [site changes](#site-changes) while connections are open, the connection-level settings will also be out-of-date.
* Multiple origins may share connections with [HTTP/2 cross-name pooling](https://httpwg.org/specs/rfc7540.html#reuse). Some origins may not be in the `ACCEPT_CH` frame, particularly if the server uses wildcard subdomains. (Note the browser may [partition connection pools](https://docs.google.com/document/d/1V8sFDCEYTXZmwKa_qWUfTVNAuBcPsu6FC0PhqMD6KKQ/edit), which would limit these scenarios.)

Thus we provide `Critical-CH` as a simple baseline mechanism, with connection-level settings as an added optimization.


### TLS 1.3 0-RTT

TLS 1.3 includes a 0-RTT optimization which allows the client to send application data after the TLS ClientHello without waiting for the server. This saves a round-trip, but the client will not have received a `ACCEPT_CH` frame yet.

0-RTT is only possible after the initial connection to the server, so, in most cases, the client will already have up-to-date Accept-CH preferences cached. Neither `Critical-CH` nor the `ACCEPT_CH` frame is necessary.

In some edge cases, the cache may be out-of-date. The server may since have changed, or as a consequence of some cross-name pooling behaviors. The `Critical-CH` header again restores reliability, at the cost of a retry. (Note: ALPS itself must interact with 0-RTT cleanly. It is likely that the server would decline 0-RTT when the saved settings are stale.)


### Retry limits

To avoid infinite loops, the client should not retry more than once per request. Additionally, it should only retry GET requests.


### Connection vs cache ordering

With two sources of Accept-CH information, we must decide what order to resolve them in. If the connection information always overrides the header cache, a client with a long-lived connection from before the site made a change will not pick up new values. This would pay round-trips repeatedly. If the header cache overrides connection information, we avoid this but may unnecessarily pay a round-trip once if the site changed while there wasn’t a connection open (see “Site changes” above). We could also pick more complex options like unioning them or tracking which was received more recently.


## Considered alternatives


### Alternate connection-level mechanisms

The proposal above uses an `ACCEPT_CH` HTTP frame and a TLS extension (ALPS) to deliver it reliably. Here we discuss alternate wire formats.


#### Client-hint-specific TLS extension

We could simply define an `http_accept_client_hints` TLS extension. This would additionally work for HTTP/1.1. However, this pattern adds more TLS API surface every time we need connection-level metadata. Some site administrators rarely update libraries, so adding one general-purpose extension for HTTP parameters seems preferable.

The loss of HTTP/1.1 coverage seems less important. Any connection-level change would require a server software upgrade. HTTP/2 is five years old now and a substantial performance win. Sites concerned with performance should deploy it anyway.


#### SETTINGS and EXTENDED\_SETTINGS

An earlier iteration of this design used an HTTP/2 and HTTP/3 SETTINGS value rather than a frame. However, SETTINGS values can only be integers, so this would not work. There was an [EXTENDED\_SETTINGS](https://tools.ietf.org/id/draft-bishop-httpbis-extended-settings-01.html) proposal which allows variable-width settings values, which would work as an alternative to a dedicated frame. (See [related discussion](https://lists.w3.org/Archives/Public/ietf-http-wg/2016JulSep/0127.html).)


#### ACCEPT\_CH in half-RTT data

We could add `ACCEPT_CH` without ALPS. However, clients cannot reliably wait for it without a round-trip penalty:

* In TLS 1.2 with False Start, waiting for server data adds a round-trip delay (negating the benefit of False Start).
* In TLS 1.3, the server can send frames in half-RTT data, but there is no guarantee it will do so. There is also no way for the client to tell if the server does this, or how many frames to wait for. Any HTTP/2 client that waits for `ACCEPT_CH` thus risks paying a round-trip delay.

The client could opportunistically read `ACCEPT_CH` from half-RTT data, but without a delimiter, this is unreliable even with servers that use half-RTT data. If it is reliable enough in practice, the gaps could be filled `Critical-CH`. However, many servers do not support half-RTT in 1-RTT connections, and this seems needlessly flaky. A more robust solution for HTTP parameter negotiation seems worthwhile.


#### Half-RTT data indicator

Finally, we could use half-RTT data, but introduce a TLS extension for the server to indicate the end of half-RTT data with some post-handshake message. The client can then reliably wait for it without risking a round-trip delay.

This could work, but it seems needlessly complex, particularly when integrated with HTTP/3. (TLS post-handshake messages in QUIC are not ordered relative to application data.) Streaming half-RTT data on the server also has some [API challenges](https://mailarchive.ietf.org/arch/msg/tls/hymweZ66b2C8nnYyXF8cwj7qopc/). Finally, while this option does not modify HTTP/2’s wire image, it modifies the I/O patterns. This still means a change in HTTP/2 implementations and the protocol itself.

#### DNS

We could try to get the signal even earlier, such as through the [HTTPS DNS record](https://tools.ietf.org/html/draft-ietf-dnsop-svcb-https-02). This has security and operational issues. DNS records are largely not authenticated by the origin today, so this would allow the resolver to tamper with the web page. As this is an out-of-band signal, there are also operational difficulties for servers making sure the DNS information and server preferences are in sync. Today, web developers do not need to carefully synchronize their DNS records with web content. Finally, it is likely that coverage will be incomplete. An [old experiment](https://www.imperialviolet.org/2015/01/17/notdane.html) suggested, at the time, 4–5% of users could not even fetch TXT records.

An in-band signal in the connection avoids these concerns while still delivering the data before the first HTTP request.


### Alternate retry mechanisms


#### Server-triggered retry

Rather than a client-triggered retry, the server could perform the retry instead. For instance, the server could return a self-redirect if a critical client hint was missing. This has two problems:

First, a missing Client Hint does not immediately imply a retry. The client may not support the Client Hint, or it may have intentionally omitted it due to user preferences, [Privacy Budget](https://github.com/bslassey/privacy-budget), or other client policy. The client could expose richer information to implement this (e.g. include the list of _all_ supported Client Hints in each request), but this wastes bandwidth and results in even more complex server logic.

Second, we wish to minimize developer obligations here. A server-triggered retry may be complex and easily trigger an infinite loop if wrong. Moreover, this code is not exercised after the first page load, so routine testing would not reveal mistakes. The `Critical-CH` header is the minimal additional overhead over the already required `Vary` header.


#### Reuse Accept-CH or Vary instead of a new header

The server is already required to send `Accept-CH` and `Vary` headers with lists of Client Hints, so we could reuse those. However, this would result in more retries than necessary. `Accept-CH` may contain Client Hints used by other resources that this one ignores. `Vary` may contain Client Hints that this resource used, but only as an optimization.


#### UA heuristics on Client Hint criticality

The above alternative could be combined with built-in client assumptions that, e.g., the `Viewport-Width` Client Hint is never critical, while the User-Agent Client Hints always are.


## References & acknowledgements

This design was based on discussions with Ilya Grigorik, Nick Harper, Victor Vasiliev, and Yoav Weiss.
