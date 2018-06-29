---
title: HTTP/2 Server Push Use Cases
abbrev: Push Use Cases
docname: draft-bishop-httpbis-push-cases-latest
date: {DATE}
category: info

ipr: trust200902
area: Applications
workgroup: HTTPbis
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
  -
    ins: M. Bishop
    name: Mike Bishop
    organization: Akamai
    email: mbishop@evequefou.be

informative:
  RFC7540:
  RFC5988:
  RFC8030:
  I-D.ietf-httpbis-cache-digest:
  FETCH:
    title: Fetch Living Standard
    target: "https://fetch.spec.whatwg.org"



--- abstract

HTTP/2 defines the wire mechanics of Server Push. Though the mechanics of how a
pushed resource is delivered are well-specified, the use cases that describe
which resources can be pushed, in what states, and for what purpose are not
described in HTTP/2.  As a result, support between implementations varies
widely.

This document attempts to enumerate interesting scenarios, in hopes that a more
concrete taxonomy can assist the community in arriving at a standard set of
supported scenarios.

--- middle

# Introduction        {#problems}

HTTP is fundamentally a request/response mechanism.  Each request specifies some
action (often content retrieval) which the user agent wishes the server to
perform on a resource and return the result of the action.

However, when a response triggers a subsequent request, network and processing
delays cause this subsequent request not to begin immediately.  The client must
receive the initial response (0.5 RTT network delay), parse it and identify the
next request (implementation- and page-dependent), then send the request back to
the server (0.5 RTT network delay) before the server can begin to satisfy the
request.

Some mechanisms attempt to reduce the client processing time by enumerating
important follow-up requests in HTTP headers or at the beginning of the response
payload.  However, these techniques still incur at least 1 RTT of network delay
before the server receives the subsequent request.  Depending on the distance
between client and server, this delay might be significant.

HTTP/2 defines Server Push, a paradigm in which the server can also return the
results of actions the user agent did *not* request.  In order to preserve the
request-response semantics of HTTP, the server first "promises" something by
supplying the request to which it will later provide a response as if the user
agent had made the supplied request. Push is also restricted to methods which
are defined to be safe and cacheable.  (Though the response itself need not be
cacheable.)

A push is always associated directly with a client-generated request. No
chaining is permitted (a push cannot be associated with another push), nor can a
push be promised after the client-initiated stream has closed, though a promised
push can be fulfilled at any subsequent time.

This document is an attempt to enumerate categories of resources and situations
in which a server might choose to push a resource.

# Types of Resources

## Browser Scenarios

The most visible use case for HTTP is the web browser.  Because a web page is
composed of a large number of resources, often spread across different domains,
browsers issue a complex series of requests in the course of rendering a page.
However, because browsers are generating these requests based on the combination
of server-supplied content (the initial response) and local state (the contents
of the cache, if any), the server is able to partially predict the client's
rendering behavior.  Efforts to standardize the download process between clients
([FETCH]) and to supply more information about the local state to the server
([I-D.ietf-httpbis-cache-digest]) will improve the server's ability to predict
such subsequent requests.

However, web pages are increasingly dynamic, and this makes the browser's job of
determining what is logically connected with the initial request more complex.

### Static DOM contents

This is almost universally-supported among browsers.  If the resource directly
references another resource, either via Link headers ([RFC5988]) or via the
entity body, almost all browsers will accept a match for this resource served as
a push.

Such resources are typically CSS, JavaScript, images, or other elements required
for the rendering or functionality of the page.

### Updated DOM contents

After the DOM is constructed, it can be locally modified.  Some web sites
consist of very sparse static content, then rely on the execution of scripts
locally to create DOM elements or update existing elements to reference new
target resources.

### Requests made by local script execution {#browser-script}

The locally executed script, rather than updating DOM elements, is also able to
directly generate new HTTP requests (typically so that they can populate DOM
elements by script).  Many sites have expressed use-cases for pre-answering
queries their script would generate in response to user action.

For example, the page might contain a list of e-mails, with script populating
the contents of the e-mail once the user clicks a list entry.  If a site author
knows the user is likely to click one of the first three unread e-mails, the
contents of those e-mails are reasonable candidates for Server Push.

### Future Navigations

Certain web pages naturally lend themselves to obvious next steps.  This might
be a checkout flow on a retail site, or multi-page content in an article or series.
A web site could promise some elements of the likely future navigation to the browser,
satisfying them only once everything relevant to the current page has been transferred.

The benefit of using push for this seems dubious, since the latency savings of
waiting for the browser to know about the future page are not present.  A better
solution, in many cases, would be the `prefetch` or `prerender` Link
relationships.


## Non-Browser Scenarios

Non-browser scenarios can typically be analyzed as an API.

### Subsequent API Requests

There is substantial overlap between non-browser scenarios and the in-browser
script scenario described in {{browser-script}}.  In both cases, the server is
predicting subsequent API calls based on where it believes the client is in the
process of execution.

If the server can predict subsequent actions, it can pre-satisfy the most likely
cases in order to minimize latency.

### Streaming APIs

Some APIs do not fit well with the request-response model of HTTP, but are
better understood as a subscription to an event or a data stream.  Push can be
used in this model, as in [RFC8030], by opening a request which never completes
and pushing new data to the client as it becomes available.

Some implementations of push do not make incoming push streams available to
clients unless the client makes a request for the corresponding URL.  As a
result, clients and servers implementing this pattern might wish to also use the
original request to stream back the resource identifiers which are being pushed.

# Types of Pushes

In addition to pushing different resources, push can be delivered in different
ways and for different purposes.

## Pre-Satisfying Requests

The most common and most widely supported variety of push simply provides a
request that the client is expected to make, along with a successful (typically
`200`) response carrying the content.

## Cache Population

Pushing resources into the client cache has been a subject of some controversy.
Almost all browsers implement a separate and ephemeral cache of pushed resources
and do not allow pushed resources to write directly into the HTTP cache,
theoretically as a mitigation against populating the cache with content that
will never actually be needed.

However, mechanisms such as Link headers ([RFC5988]) with the `rel=preload`
attribute can cause the browser to request any resource, and these headers are
also under the server's control.  As a result, any client that implements
{{pre-satisfying-requests}} will also enable cache population with one
additional step.

## Cache Revalidation

While no user agents are known to support this, a server could hypothetically
pre-validate stale items in the client's cache by pushing a conditional request
(If-Not-Match or If-Not-Modified) for a resource it believes is already in the
cache and a 304 (Not Modified) response.

Such a push could be consumed directly by the cache (updating the validity of
the cached object) or treated as a pre-satisfied request should the cache
attempt to revalidate an expired object.  The former will tend to extend the
lifetime of the cached object on each use, while the latter will not affect the
object unless it has expired.

A push which attempts to revalidate content which is no longer cached could be
treated equivalently to a preload link, a suggestion that this content might
soon be needed. Triggering a follow-up request for the full body might improve
performance.

## Cache Invalidation

Alternatively, the server could push an unconditional request for the resource,
but use the HEAD method instead of GET.  If interpreted appropriately by a user
agent, this could revalidate a matching item in the cache, or invalidate an item
even if the user agent currently believes the item to be fresh.

# Security Considerations

There probably are some. The existing mitigations against gratuitous pushes are
largely ineffective (see {{cache-population}}), but clients MUST verify that
servers are authoritative for any resources they attempt to push, regardless of
the scenario being considered.

--- back
