---
title: GREASE for HTTP/2
abbrev: GREASE for HTTP/2
docname: draft-bishop-httpbis-grease-latest
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



--- abstract

Reserves several values in the HTTP/2 registries to exercise the requirement
that clients and servers ignore unknown values.

--- middle

# Introduction        {#problems}

{{?UseIt=I-D.thomson-use-it-or-lose-it}} observes that extension and negotiation
mechanisms which aren't exercised regularly can be found not to work when they
are later employed by an extension to the protocol.
{{?GREASE=I-D.ietf-tls-grease}} is one mitigation which originated in TLS,
registering multiple values in various TLS registries which can be sent
prospectively by clients.

The common requirement of the different spaces described by these documents is
the requirement that recipients ignore unrecognized values.  By reserving a
scattered set of codepoints to have no defined meaning, clients and servers can
inject values from these ranges into connections on a regular basis and exercise
this requirement.

HTTP/2 {{!HTTP2=RFC7540}} frame types and settings employ a similar mechanism of
ignoring unknown values. This makes HTTP/2 a good candidate to employ grease on
connections. The need for such a technique was demonstrated recently by an
HTTP/2 implementation which closed the connection upon receipt of an unknown
setting.


# Using GREASE in HTTP/2

## GREASE for Frame Types

Frame types of the format `0xb + (0x1f * N)` are reserved for use as grease.
These frames have no semantic meaning, and SHOULD be send instead of using
padding on DATA or HEADERS frames where possible.  They MAY also be sent on
connections where there is no application data currently being transferred.
Endpoints MUST NOT consider these frames to have any meaning upon receipt.

The flags, the payload, and the length of the frames SHOULD be selected
randomly, subject to implementation-defined limits on the length.

{{!HTTP2}} is ambiguous about whether unknown frame types are permitted on
streams in the "idle", "reserved", "closed", or "half-closed (local)" states.
As a result, some implementations could legitimately consider this to be an
error.  Therefore, these frames SHOULD NOT be sent on streams in those states.

## GREASE for SETTINGS

Settings identifiers of the format `0x?a?a` are reserved for use as grease.
Such settings have no defined meaning.  Endpoints SHOULD include at least one
such setting in their initial SETTINGS frame, and MAY send new SETTINGS frames
during the connection containing additional grease values.  Endpoints MUST NOT
consider such settings to have any meaning upon receipt.

Because the setting has no defined meaning, the value of the setting SHOULD be
selected randomly.


# Security Considerations {#security}

The ability to design, implement, and deploy new protocol mechanisms can be
critical to security.

# IANA Considerations {#iana}

## Frame Types

This document reserves a range of entries in the "HTTP/2 Frame Type" registry
defined in {{!HTTP2}}.  Each code of the format `0xb + (0x1f * N)` for values of
N in the range (0..7) (that is, `0xb`, `0x2a`, etc., through `0xe4`) MUST NOT be
assigned by IANA for any purpose.

## Settings

This document reserves a range of entries in the "HTTP/2 Settings" registry
defined in {{!HTTP2}}.  Each code of the format `0x?a?a` where each `?` is any
octet (that is, `0x0a0a`, `0x0a1a`, etc. through `0xfafa`) MUST NOT be assigned
by IANA for any purpose.

--- back

# Acknowledgements
{:numbered=false}

This draft arose from a discussion in the QUIC WG with Lucas Pardue, Ryan
Hamilton, and Martin Thomson.
