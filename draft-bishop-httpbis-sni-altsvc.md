---
title: The "SNI" Alt-Svc Parameter
abbrev: SNI Alt-Svc
docname: draft-bishop-httpbis-sni-altsvc-latest
date: {DATE}
category: std

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



--- abstract

HTTP Alternative Services provides a mechanism for an origin to declare that its
content is accessible via some other combination of host, port, and protocol.
In the process of using such an alternative, an observer can identify that the
client is requesting resources from a particular hostname.

This document extends HTTP Alternative Services, in combination with Secondary
Certificate Authentication, to enable clients not to disclose the origin to
which they intend to connect.

--- middle

# Introduction        {#problems}

Confidentiality and authentication during communication are primary goals of
using TLS to secure traffic on the Internet.  However, due to the nature of TLS,
certain information is inherently not confidential -- notably, the hostname and
the corresponding certificate of the origin to which the client is connecting
are transferred unencrypted in the Server Name Indication extension
{{!SNI=RFC6066}} and the server's Certificate message {{!TLS12=RFC5246}}.

While the client identity can be obscured by using TLS renegotiation immediately
after the handshake (in TLS 1.2) or by using TLS 1.3
{{!TLS13=I-D.ietf-tls-tls13}}, the server is not afforded such privacy
considerations.

Servers may also have wildcard certificates which do not enumerate specific
subdomains, but clients will disclose the first subdomain used on a connection
via the SNI extension when establishing the connection.

{{?SNIEncryption=I-D.ietf-tls-sni-encryption}} discusses a potential solution to
these issues in Section 3, HTTP Co-Tenancy Fronting, but notes both
discoverability and server authentication issues with that approach. This
document provides a mechanism to address both limitations.

## Usage

In {{!AltSvc=RFC7838}}, once a client has received a validated Alternative
Service record for an origin, it "SHOULD use that alternative service for all
requests to the associated origin as soon as it is available, provided the
alternative service information is fresh (Section 2.2) and the security
properties of the alternative service protocol are desirable, as compared to the
existing connection." However, the client "MUST have reasonable assurances that
the alternative service is under control of and valid for the whole origin ...
established through use of a TLS-based protocol with the certificate checks
defined in {{!RFC2818}}."  This causes the origin to be disclosed in the SNI
extension while connecting to the alternative, and the origin's certificate to
be returned by the alternative, creating the same privacy issues as connecting
directly to the origin.

The extension described in {{extension}} enables an origin to declare that
reasonable assurances should be obtained, not by requesting the desired hostname
in the TLS handshake, but by requesting it via
{{!SecondaryCerts=I-D.bishop-httpbis-http2-additional-certs}}.  The validation
checks from {{!RFC2818}} are applied to this certificate.

Because the entire exchange happens inside TLS, a passive observer cannot
identify the hostname(s) the client might be requesting.

## Notational Conventions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

The key words "MUST (BUT WE KNOW YOU WON'T)", "SHOULD CONSIDER", "REALLY SHOULD
NOT", "OUGHT TO", "WOULD PROBABLY", "MAY WISH TO", "COULD", "POSSIBLE", and
"MIGHT" in this document are to be interpreted as described in {{!RFC6919}}.

Field definitions are given in Augmented Backus-Naur Form (ABNF), as defined in
{{!RFC5234}}.

# The "sni" Alt-Svc Extension {#extension}

When an origin wishes to nominate a "fronting server", it includes the `sni`
parameter in its alternative service entry.

Syntax:

    sni = ( host / empty-string )
    empty-string = DQUOTE DQUOTE

`host` is defined in Section 3.2.2 of {{!RFC3986}}.

When processing such an alternative, clients SHOULD present the hostname given
in the `sni` parameter in the SNI extension during the TLS handshake. If the
hostname given is an IP literal or an empty string,
clients SHOULD omit the SNI extension from
the TLS handshake.  The server MUST return a valid certificate which covers
at least one of the following:

- The hostname indicated in the SNI extension
- The hostname of the origin that published the alternative
- The hostname used for connecting to the alternative

The client MUST validate the certificate in the handshake for authenticity
according to {{!RFC2818}} and ensure that it is valid for at least one of these
names.  Clients SHOULD NOT accept certificates issued to the IP address of the
alternative unless the alternative is specified as an IP literal.

If the certificate is not valid for the origin's hostname, the client MUST NOT
make requests to any origin corresponding to this certificate. In this case, the
client SHOULD send a `CERTIFICATE_REQUEST` frame including an SNI extension
indicating the origin which published the alternative service immediately upon
connecting.  If no corresponding `CERTIFICATE` frame is presented by the server
after a reasonable timeout, or if the server's SETTINGS frame does not include
the `SETTINGS_HTTP_CERT_AUTH` setting, the client MUST consider the alternative
connection to have failed.

# Examples

## SNI of Colocated Domain

Suppose a client has received the following Alt-Svc entry for
sensitive.example.com in the past:

    h2="innocence.org:443";ma=2635200;persist=true;sni=innocence.org

If the client now wishes to make a request to
https://sensitive.example.com/private, it would perform a DNS resolution for
innocence.org. The client would then open a TCP connection to the resulting IP
address and begin a TLS handshake.

In the client's TLS handshake, it would request a certificate for the hostname
"innocence.org".  The TLS server would present such a certificate, issued by an
authority trusted by the client. The client will validate the certificate for
the name "sensitive.example.com".  When validation fails, the client will try to
validate the certificate for the name "innocence.org", which will succeed.
After validation succeeds, the client will send a `CERTIFICATE_REQUEST` frame
asking that the server also authenticate with a certificate for
sensitive.example.com.

After receiving the `CERTIFICATE` frame proving possession of a certificate for
sensitive.example.com, the client will verify that this certificate is trusted.
If so, the client will proceed to send HTTP/2 requests to the server requesting
the resource https://sensitive.example.com/private.

## Wildcard Subdomains

Suppose a client has received the following Alt-Svc entry for
sensitive.example.com in the past:

    h2="www.example.com:443";ma=2635200;persist=true;sni=www.example.com

If the client now wishes to make a request to
https://sensitive.example.com/private, it would perform a DNS resolution for
www.example.com, the specified alternative. The client would then open a TCP
connection to the resulting IP address and begin a TLS handshake.

In the client's TLS handshake, it would request a certificate for the hostname
www.example.com.  The TLS server would present a certificate which included
www.example.com as one of the covered hostnames.

Suppose that the certificate with which the server authenticated also contained
a Subject Alternative Name of "*.example.com".  Because the certificate covers
the desired origin, the client would perform validity checks on this
certificate.

If the certificate is trusted, the client will proceed to send HTTP/2 requests
to the server requesting the resource https://sensitive.example.com/private.

## Omitting SNI

Suppose a client has received the following Alt-Svc entry for
sensitive.example.com in the past:

    h2="alternative.example.com:443";ma=2635200;persist=true;sni=""

If the client now wishes to make a request to
https://sensitive.example.com/private, it would perform a DNS resolution for
alternative.example.com, the specified alternative. The client would then open a
TCP connection to the resulting IP address and begin a TLS handshake.

In the client's TLS handshake, it would omit the Server Name Indication
extension.  The TLS server would present a certificate according to its
configured defaults.

The server would supply a certificate that covers sensitive.example.com, for example
because it contains a Subject Alternative Name of "*.example.com", and the client
would perform validity checks on this certificate.

If the supplied certificate does not cover sensitive.example.com, or is not
valid, the client will terminate the connection.

## SNI of Unrelated Domain

Suppose a client has received the following Alt-Svc entry for
sensitive.example.com in the past:

    h2=":443";ma=2635200;persist=true;sni=other.example

If the client now wishes to make a request to
https://sensitive.example.com/private, it would perform a DNS resolution for
sensitive.example.com (the Alt-Svc entry does not specify a different hostname).
The client would then open a TCP connection to the resulting IP address and
begin a TLS handshake.

In the client's TLS handshake, it would request a certificate for the hostname
"other.example".  The TLS server does not a have a certificate for this hostname,
but it would return a certificate for sensitive.example.com, issued by an
authority trusted by the client, and the client will successfully validate the
certificate for the name "sensitive.example.com".

Note that an active attacker could identify this server by sending a Client
Hello with the same SNI value and observing the certificate the server uses to
authenticate. The server could mitigate this by authenticating with a
certificate for other.example.

# Security Considerations

{{!AltSvc}} permits clients to ignore unrecognized parameters.  As a result,
servers publishing records with the `sni` parameter cannot be assured that
clients will not include their origin in the SNI header when connecting to the
nominated alternative.  If, for security reasons, an origin wishes its identity
never to be disclosed when the alternative is being used, an alternative
mechanism would be required to ascertain client support before generating the
Alt-Svc record.

Clients will need to connect directly to the origin at least once in order to
receive the Alt-Svc entry via an HTTP header or `ALTSVC` frame, thus disclosing
their use of the origin to the network on the first connection. This could be
mitigated by future work defining a way to publish alternative services in a
mechanism which can be retrieved confidentially, such as via DNS in combination
with {{?RFC7858}} or {{?DoH=I-D.ietf-doh-dns-over-https}}.

However, servers which publish Alt-Svc records over unencrypted channels (HTTP
connections without TLS) or channels without client authorization (DNS, or
publicly accessible HTTP resources) enable active observers to build a map of
fronting servers by collecting Alt-Svc advertisements.  Servers SHOULD CONSIDER
this trade-off in deciding when and how to make Alt-Svc records available to
unauthenticated parties.

While concealing information from passive observers is beneficial, low-effort
active attacks still exist.  If an attacker can collect the actual server
identity by sending a Client Hello with the same SNI value, the usefulness of
this technique is limited.  Server deployments SHOULD reserve sensitive domains
for use with Secondary Certificates or conceal them inside wildcards in order to
mitigate this.

# IANA Considerations

The "Hypertext Transfer Protocol (HTTP) Alt-Svc Parameter Registry" defines the
name space for parameters, as described in {{!AltSvc}}.  It is maintained at
<http://www.iana.org/assignments/http-alt-svc-parameters>.

This document registers the following parameter:

Name:
: `sni`

Specification:
: This document

--- back

# Acknowledgements

Conversations with Benjamin Schwartz helped to flesh out this idea.
