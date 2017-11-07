---
title: ALTSVC Frame in HTTP/QUIC
abbrev: ALTSVC in HTTP/QUIC
docname: draft-bishop-httpbis-altsvc-quic-latest
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

normative:
  RFC7838:
  I-D.ietf-quic-http:

informative:
  RFC7540:



--- abstract

[RFC7838] defines the ALTSVC frame for HTTP/2 [RFC7540]. This frame is equally
applicable to HTTP/QUIC ([I-D.ietf-quic-http]), but needs to be separately
registered. This document describes the ALTSVC frame for HTTP/QUIC.

--- middle

# Introduction        {#problems}

[RFC7838] defines HTTP Alternative Services, which allow an origin's resources
to be authoritatively available at a separate network location, possibly
accessed with a different protocol configuration.  It defines two mechanisms
for transporting such information, an HTTP response header and an HTTP/2 frame
type.

[I-D.ietf-quic-http] describes the required updates for HTTP/2 frames to be used
with HTTP/QUIC.  Only a few modifications are required for the ALTSVC frame. No
modifications are required for the "Alt-Svc" header field.

# The ALTSVC HTTP/QUIC Frame

The ALTSVC HTTP/QUIC frame advertises the availability of an alternative service
to an HTTP/QUIC client.

An ALTSVC frame from a server to a client on stream 1 (not 0, as in HTTP/2)
indicates that the conveyed alternative service is associated with the origin
contained in the Origin field of the frame.

An ALTSVC frame from a server to a client on a stream other than stream 1
indicates that the conveyed alternative service is associated with the origin of
that stream.

The layout and semantics of the frame are identical to those of the
HTTP/2 frame defined in [RFC7838].  The ALTSVC frame type is 0xa (decimal 10),
as in HTTP/2.

# Security Considerations {#security}

This document introduces no new security considerations beyond those discussed
in [RFC7838] and [I-D.ietf-quic-http].

# IANA Considerations {#iana}

This document registers the ALTSVC frame type in the "HTTP/QUIC Frame Type"
registry ([I-D.ietf-quic-http]).

Frame Type:
: ALTSVC

Code:
: 0xa

Specification:
: This document

--- back

# Acknowledgements

A substantial portion of Mike's work on this draft was supported by Microsoft
during his employment there.
