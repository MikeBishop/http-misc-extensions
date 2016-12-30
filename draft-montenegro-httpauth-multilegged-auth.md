---
title: Multilegged Authentication for HTTP Multiplexing
abbrev: HTTP Multilegged Auth
docname: draft-montenegro-httpauth-multilegged-auth-latest
date: {DATE}
category: std

ipr: trust200902
area: Security
workgroup: Hypertext Transfer Protocol Authentication
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: G. Montenegro
    name: Gabriel Montenegro
    organization: Microsoft
    email: Gabriel.Montenegro@microsoft.com
 -
    ins: M. Bishop
    name: Mike Bishop
    org: Microsoft
    email: Michael.Bishop@microsoft.com


--- abstract

HTTP/2 intends to be compatible with all widely deployed HTTP features, 
including currently-deployed authentication schemes. However, some 
currently-defined schemes don't coexist well with multiplexing. This 
draft addresses some of the issues encountered when performing 
multilegged authentication over a multiplexed channel. 


--- middle

# Overview

This document defines multilegged authentication for a multiplexed 
version of HTTP, such as HTTP/2 ({{?RFC7540}}) or HTTP/QUIC 
({{?I-D.ietf-quic-http}}). 

{{?RFC4178}} defines the Simple and Protected GSS-API Negotiation 
Mechanism (SPNEGO), a GSS-API ({{?RFC2743}}) pseudo-security method for 
negotiating which of several GSS-API credentials a client has might be 
acceptable to the server. Several GSS-API authentication types, 
including SPNEGO, can require multiple round-trips between client and 
server for authentication to complete.

{{?RFC4559}} describes how SPNEGO-based authentication methods
can be used in HTTP.
HTTP contemplates such multi-legged authentication methods -- 
{{?RFC7235}} explicitly requires the use of status code 401 when a 
request contains "partial credentials (e.g., when the authentication 
scheme requires more than one round trip)." However, existing 
multi-legged authentication schemes such as SPNEGO assume that 
subsequent requests can be correlated because they occur in sequence on 
a single TCP connection. This correlation is not possible for 
multiplexed versions of HTTP. 

Current implementations address this by converting attempts to use these 
authentication schemes into HTTP_1_1_REQUIRED errors. However, this 
introduces multiple additional round trips into any scenario that 
requires authentication -- the client will first establish an HTTP/2 
connection, then encounter the fallback error code, establish an 
HTTP/1.1 connection, retry the request, and only then receive the 
authentication challenge to which they must respond. 

SCRAM ({{?RFC5802}}), a more recent authentication method, is also 
multi-legged. However, SCRAM solves this problem itself by incorporating 
a nonce into the server's challenge which the client MUST echo back to 
the server. This permits the server to know to which challenge the 
client is attempting to respond, if multiple were issued. 

The following table summarizes widely deployed authentication schemes, 
their authentication types, and the authentication level they provide: 

+-----------|------------------------|----------------------+
| Scheme    | Type of Authentication | Authentication Level |
+-----------|:----------------------:|:--------------------:+
| Basic     |      Per Request       |       Request        |
| Digest    |      Per Request       |       Request        |
| NTLM      |      Multilegged       |       Connection     |
| Kerberos  |      Multilegged       |Connection or Request |
| Negotiate |      Multilegged       |Connection or Request |
+-----------|------------------------|----------------------+

To enable better support of multilegged authentication, we propose that 
the shared state for which some authentication schemes rely upon the TCP 
connection (thereby making messages stateful) be moved into the HTTP 
layer, as SCRAM has done. 

## Requirements Language

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in {{!RFC2119}}.


# Multilegged Authentication in HTTP/1.X

As implied by its name, multilegged authentication requires multiple 
roundtrips to establish an authenticated communication channel between 
client and server. If the resource requested by a client requires 
authentication, the server initiates the authentication process as 
follows: 

~~~~~
  Client                                 Server
    |                                      |
    | -------- (0) HTTP GET Request ---->  |
    |                                      |
    | <------- (1) HTTP 401 -------------  |
    |              WWW-Authenticate header |
    |                                      |
    | -------- (2) HTTP GET Request ---->  |
    |              Authorization header    |
    |                                      |
    | <------- (3) HTTP 401 -------------  |
    |              WWW-Authenticate header |
    |                                      |
    | -------- (4) HTTP GET Request ---->  |
    |              Authorization header    |
    |                                      |
    | <------- (5) HTTP 200 OK-----------  |
    |              WWW-Authenticate header |
    |                                      |
    v                                      v
~~~~~
{: #figure-existing title="Multilegged authentication example"}

   1.  Server sends HTTP 401 response because the resource requested
       requires authentication.  Response includes `WWW-Authenticate`
       header describing acceptable authentication method(s) and
       possibly challenge information.

   2.  Client re-issues the HTTP GET request for the resource, including
       an Authorization header.

   3.  Server optionally responds with an HTTP 401 and `WWW-Authenticate`
       header if the client's credentials were incomplete.

   4.  If needed, client re-issues the HTTP GET Request for the resource,
       including an `Authorization` header with the additional information
       required to complete authentication.

   5.  If authentication succeeds, the server responds with an HTTP 200
       status code and including the requested resource.

On a multiplexed protocol, the request/response pairs 0/1, 2/3, and 4/5
will occur on different streams, with other requests occurring in parallel.
When the server sees request #4, it cannot correlate that Authorization
header with the `WWW-Authenticate` header on response #3 (as compared to
other `WWW-Authenticate` headers that may have been sent on the same
connection).


# Multilegged Authentication in the Presence of Multiplexing 

In a multiplexing protocol such as HTTP/2, multiple client requests can 
be outstanding at one time. In the event that multiple requests receive 
authentication challenges, the server cannot reliably determine to which 
challenge the client is responding in subsequent authentication 
attempts. Current implementations force a fallback to HTTP/1.1 to work 
around this limitation. 

{{figure-proposed}} below provides a breakdown of proposed network flows 
to implement multilegged authentication for multiplexing: 

~~~~~
                                       Multiplexing
   Client                               HTTP Server
      |                                      |
      | --------   HTTP GET Request ------>  |
      |                                      |
      | <------- (0) HTTP 401 -------------  |
      |              WWW-Authenticate header |
      |              (Auth-ID header)        |
      |                                      |
      | -------- (1) HTTP Get Request ---->  |
      |              Authorization header    |
      |              (Auth-ID header)        |
      |                                      |
      | <------- (2) HTTP 401 -------------  |
      |              WWW-Authenticate header |
      |              Auth-ID header          |
      |                                      |
      | -------- (3) HTTP Get Request ---->  |
      |              Authorization header    |
      |              Auth-ID header          |
      |                                      |
      | <------- (4) HTTP 200 OK-----------  |
      |              WWW-Authenticate header |
      |              (Persisted-Auth header) |
      |                                      |
      |                                      |
      v                                      v
~~~~~
{: #figure-proposed title="Proposed multilegged authentication"}

   1.  Server sends HTTP 401 response because the resource requested
       requires authentication.  If the `WWW-Authenticate` header contains
       any state, the server MUST include an `Auth-ID` header. (See
       {{header-auth-id}}).

   2.  Client re-issues the HTTP GET request for the resource, including
       the `Authorization` header.  If the server provided an `Auth-ID`
       header, the client MUST echo the same value on the subsequent
       request.

   3.  If additional information is required, server responds with an HTTP
       401 and a `WWW-Authenticate` header requesting additional information.
       An `Auth-ID` header MUST be included, since the additional information
       relies on existing state already provided by the client.

   4.  Client re-issues the HTTP GET request for the resource and MUST
       include the required `Authorization` header and the `Auth-ID`
       header, to inform the server that the request is part of a
       previously-initiated multilegged authentication process.

   5.  Authentication succeeds and the server returns the requested
       resource.  If the exchange authenticated the connection, the server
       SHOULD include a `Persisted-Auth` header (see {{header-persisted-auth}}).
       
## Request Management

Multilegged authentication schemes could authenticate individual 
requests or the HTTP/2 session. Clients SHOULD NOT attempt to 
authenticate individual requests when the HTTP connection has already 
been authenticated. 

The client MUST NOT send multiple requests in parallel with the same 
Auth-ID, and SHOULD NOT send multiple requests in parallel with 
different Auth-IDs if any of those requests is expected to result in 
connection-level authentication. If an HTTP/2 session has a stream in 
process of authenticating using a multilegged authentication scheme and 
the client does not know whether the request will authenticate the 
connection, the client SHOULD queue all subsequent requests which 
require authentication on the session until the multilegged 
authentication completes. If the client does not queue the requests, 
then it might unnecessarily authenticate streams in a session that has 
already been authenticated. However, if a server receives multiple 
authenticated requests from the same client, it SHOULD NOT block 
responses. 

If connection-based multilegged authentication succeeds a second time on 
a previously authenticated session, the server SHOULD discard the 
previous authentication context and authenticate the session with the 
newly negotiated authentication context. 

## The Auth-ID Header {#header-auth-id}

If the server responds to a request with a "401 Unauthorized" status 
code and the `WWW-Authenticate` header (required by {{!RFC7235}}) contains 
state in its `auth-data` value, the server MUST also include an Auth-ID 
header. If the request to which the server is responding contained an 
Auth-ID header, it MUST include an `Auth-ID` header in its response, and 
the value in the server's response SHOULD be the same.

A server MAY generate and add an `Auth-ID` header as soon as it knows
that the requested authentication scheme is multilegged.  The
client MUST add the `Auth-ID` header to all subsequent requests
required to complete the authentication process.

When a client sends a request with an Authorization header after 
receiving a "401 Unauthorized" status code bearing an `Auth-ID` header, it 
MUST include an identical `Auth-ID` header in the request if the client's 
Authorization header relies on any state from the server's 
WWW-Authenticate header, or if the `WWW-Authenticate` header was sent in 
response to a request bearing an `Auth-ID` header.

The `Auth-ID` header is defined as follows:

    Auth-ID = token; see [RFC7230], Section 3.2.6

The `Auth-ID` header represents a key into short-lived data on the server 
associated with the current connection, and MAY be selected by the 
server using any implementation-defined method. It is an opaque 
identifier, and clients MUST NOT attempt to interpet it. The Auth-ID 
mapping is destroyed when the authentication process completes, whether 
in success or failure. 

Servers SHOULD limit the number of incomplete security contexts per 
session, to protect against misbehaving clients that cause the server to 
create multiple authentication contexts but never complete the 
authentication process. Servers SHOULD define a maximum number of 
incomplete security contexts and discard inactive contexts.  If a request
contains an unknown or expired Auth-ID, the request should be treated
as containing no Auth-ID.

## The Persisted-Auth Response Header {#header-persisted-auth}

Some multilegged authentication schemes can result in per-request or 
per-connection (i.e., Kerberos or Negotiate) authentication. When a 
session is authenticated, servers SHOULD generate a `Persisted-Auth` 
header and send it along with the HTTP 200 OK response.

The `Persisted-Auth` header is defined as follows:

    Persisted-Auth = "true" / "false"

A `Persisted-Auth` header with an unknown value, or the absence of a 
`Persisted-Auth` header, MUST be treated as the value "false". The 
client MUST use the `Persisted-Auth` header to determine what action to 
take with queued or subsequent requests: 

1.  If the session was authenticated, as indicated by the
    presence of the `Persisted-Auth` header with the value "true",
    the client does not need to authenticate new streams it creates
    to service future requests on the authenticated session.

2.  If the session was not authenticated, as indicated by the
    absence of the `Persisted-Auth` header or the value "false",
    the client SHOULD remember the negotiated authentication scheme
    used for authentication.  The client SHOULD NOT block streams on the
    session when processing requests using the multilegged
    authentication scheme that previously resulted in per-request
    authentication.

Clients SHOULD assume that successful authentication with schemes that 
only support connection-based authentication (e.g. NTLM) always results 
in an authenticated session, even if the `Persisted-Auth` header is not 
present. 

The client CANNOT make any assumptions by the absence of the 
`Persisted-Auth` header until the authentication process is complete and 
it receives the final server response containing the requested resource. 


# Stateful Authentication via Proxies

Typically, authenticated connections via a forward proxy will occur 
using the CONNECT method ({{?RFC7231}}, Section 4.3.6) and the proxy 
will not be involved in the authentication flow. Authenticated connections
via a reverse proxy will often terminate authentication at the proxy,
with only authorized requests reaching the content server.

When the proxy speaks only HTTP/1.1, no multiplexing will occur and the 
considerations in this document do not apply. Proxy and gateway 
applications should take the consideration outlined by {{!RFC7230}} when 
forwarding messages between client and servers with different protocol 
version capabilities.

This section applies only to situations where the proxy has access to 
the requests but is not performing authentication itself, and is
using HTTP/2 to communicate with the client, the server, or both.


## HTTP/1.1 Client Authenticating to HTTP/2 Server via HTTP/2 Proxy

When the proxy uses HTTP/2 to connect to the server, it MUST ensure
that any authentication method that could result in per-connection
authentication is used only on a separate connection for each client.
This could require replaying requests on a new connection.

For per-request authentication methods, the server MUST associate each
received `Auth-ID` from the server to the client TCP connection and
insert the `Auth-ID` header on future requests originating from the
same connection which contain an `Authorization` header.


## HTTP/2 Client Authenticating to an HTTP/1.1 Server via HTTP/2 Proxy

HTTP/2 clients can multiplex streams within the authenticated HTTP/2 
client-proxy session, but the proxy MUST serialize these requests 
through HTTP/1.1 proxy-server connections. These connections MUST NOT be 
shared between different clients if per-connection authorization is in
use, and SHOULD NOT be shared between different clients even if
per-request authorization is in use.

The proxy MUST generate and insert an `Auth-ID` header corresponding to 
each TCP connection it maintains to the server, and add the `Auth-ID` 
header to any 401 response from the server. Client requests which 
include an `Auth-ID` header MUST be directed to the associated TCP 
connection, or a new connection if that one has closed. Requests which 
do not include an `Auth-ID` header MAY be relayed over any connection. 


## HTTP/2 Client Authenticating to an HTTP/2 Server via HTTP/2 Proxy

The proxy MUST bind the HTTP/2 connections it maintains with the client 
and server, using a separate connection to the server for each client. 
The proxy MUST relay the `Auth-ID` and `Persisted-Auth` headers unmodified 
between these connections.


# Authentication and multi-host sessions

Clients MUST NOT make requests to multiple origins when per-connection
authentication has been established to one origin.  If an authentication
scheme which might result in per-connection authentication is requested
on a connection which has already been used to make requests of multiple
origins, the client SHOULD repeat the request on a different connection.

Servers SHOULD respond to requests for different origins on an authenticated
connection with the 421 status code (defined in {{!RFC7540}}), requiring
clients to retry on a separate connection.

# Interaction with Legacy Peers

Servers and clients which do not implement this specification are expected
to treat attempts at multi-legged authentication as situations which
require a fallback to HTTP/1.1.  Clients which implement this specification
and encounter 

# Security Considerations

Implementers should be aware of the security considerations defined 
by the individual authentication schemes supported. The following are 
some general security considerations that are independent of the 
proposed authentication mechanism.

The proposed authentication mechanism is only used to provide 
authentication of a user to a server. It provides no facilities for 
protecting the HTTP headers or data including the Authorization and 
WWW-Authenticate headers that are used to implement this mechanism. 

Alternate mechanisms such as TLS can be used to provide confidentiality. 
Hashes of the TLS certificates can be used as channel bindings to secure 
the channel. In this case, clients would need to enforce that the 
channel binding information is valid. 

If an HTTP proxy is used between the client and server, it MUST take 
care to not share authenticated connections between different 
authenticated clients to the same server. If this is not honored, then 
the server can easily lose track of security context associations. 

# IANA Considerations

HTTP header fields are registered within the "Message Headers"
registry maintained at
<http://www.iana.org/assignments/message-headers/>.

This document defines the following HTTP header fields in the
"Provisional Message Header Field Names" registry (see {{?RFC3864}}).

   +-------------------|----------|-------------|---------------------+
   | Header Field Name | Protocol | Status      | Reference           |
   +-------------------|----------|-------------|---------------------+
   | Auth-ID           | http     | provisional | {{header-auth-id}}  |
   | Persisted-Auth    | http     | provisional | {{header-persisted-auth}}
   +-------------------|----------|-------------|---------------------+

The change controllers are the authors of this document.

--- back

# Acknowledgements

The authors of the original version of this draft were Jonathan Silvera,
Matthew Cox, Ivan Pashov, Osama Mazahir, and Gabriel Montenegro.

Thanks to the following individuals who provided helpful feedback and 
contributed to discussions on this document: Paul Leach, Nathan Ide 
and Rob Trace. 

