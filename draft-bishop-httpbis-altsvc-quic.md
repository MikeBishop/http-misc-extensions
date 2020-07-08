---
title: Existing HTTP/2 Extensions in HTTP/3
abbrev: HTTP/2 Extensions in HTTP/3
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

informative:
  RFC7540:



--- abstract

The ALTSVC and ORIGIN frames for HTTP/2 are equally applicable to HTTP/3, but
need to be separately registered. This document describes the ALTSVC and ORIGIN
frames for HTTP/3.

--- middle

# Introduction {#problems}

Existing RFCs define extensions to HTTP/2 which remain useful in HTTP/3.
Appendix A.2.3 of {{!HTTP3=I-D.ietf-quic-http}} describes the required updates
for HTTP/2 frames to be used with HTTP/3.

{{!ALTSVC=RFC7838}} defines HTTP Alternative Services, which allow an origin's
resources to be authoritatively available at a separate network location,
possibly accessed with a different protocol configuration.  It defines two
mechanisms for transporting such information, an HTTP response header and an
HTTP/2 frame type.

{{!ORIGIN=RFC8336}} defines the HTTP/2 ORIGIN frame, which indicates what
origins are available on a given connection.  It defines a single HTTP/2 frame
type.

# Basic Mapping Conventions

Where HTTP/2 reserves Stream 0 for frames related to the state of the
connection, HTTP/3 defines a pair of unidirectional streams called "control
streams" for this purpose.  Where the existing RFCs indicate that a frame
should be sent on Stream 0, this should be interpreted to mean the HTTP/3
control stream.

# The ALTSVC HTTP/3 Frame {#frame-altsvc}

The ALTSVC HTTP/3 frame advertises the availability of an alternative service to
an HTTP/3 client.

An ALTSVC frame from a server to a client on the server's control stream
indicates that the conveyed alternative service is associated with the origin
contained in the Origin field of the frame.

An ALTSVC frame from a server to a client on a stream other than the control
stream indicates that the conveyed alternative service is associated with the
origin of that stream.

The layout and semantics of the frame payload are identical to those of the
HTTP/2 frame defined in {{!ALTSVC}}.  The ALTSVC frame type is 0xa (decimal 10),
as in HTTP/2.

# The ORIGIN HTTP/3 Frame {#frame-origin}

The ORIGIN HTTP/3 frame allows a server to indicate what origin(s)
({{?RFC6454}}) the server would like the client to consider as members of the
Origin Set (Section 2.3 of {{!ORIGIN}}) for the connection within which it
occurs.

The ORIGIN frame is sent from servers to clients on the server's control stream.

The layout and semantics of the frame payload are identical to those of the
HTTP/2 frame defined in {{!ORIGIN}}.  The ORIGIN frame type is 0xc (decimal 12),
as in HTTP/2.

# Security Considerations {#security}

This document introduces no new security considerations beyond those discussed
in {{!ALTSVC}} and {{!HTTP3}}.

# IANA Considerations {#iana}

This document registers two frame types in the "HTTP/3 Frame Type"
registry ({{!HTTP3}}).

| ---------------- | ------ | -------------------------- |
| Frame Type       | Value  | Specification              |
| ---------------- | :----: | -------------------------- |
| ALTSVC           |  0xa   | {{frame-altsvc}}           |
| ORIGIN           |  0xc   | {{frame-origin}}           |
| ---------------- | ------ | -------------------------- |
{: #iana-frame-table title="Registered HTTP/3 Frame Types"}

--- back

# Acknowledgements

A portion of Mike's work on this draft was supported by Microsoft during his
employment there.
