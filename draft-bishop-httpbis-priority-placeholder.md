---
title: Priority Placeholders in HTTP/2
abbrev: Placeholders in HTTP/2
docname: draft-bishop-httpbis-priority-placeholders-latest
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
    organization: Microsoft
    email: michael.bishop@microsoft.com

normative:
  RFC7540:

informative:
  I-D.ietf-quic-http:



--- abstract

[RFC7540] defines HTTP/2, including a method for communicating priorities.
Some implementations have begun using closed streams as placeholders when
constructing their priority tree, but this has unbounded state commitments and
interacts poorly with HTTP/QUIC ([I-D.ietf-quic-http]).  This document proposes
an extension to the HTTP/2 priority scheme for both protocols.

--- middle

# Introduction        {#problems}

Stream Priority is described in [RFC7540], Section 5.3.  Priority is
communicated using PRIORITY frames and with reference to other streams, with
stream 0 being the root of the tree.  Each stream depends on one other stream
with a particular weight; these weights represent relative priorities among the
multiple children of a stream.

Unfortunately, the scheme as specified encourages servers to actively maintain
closed streams in the priority tree, since other streams might reference them
later.  This produces an unbounded state commitment on the part of the server
if it is to correctly reflect any possible reference the client might make.
While priorities are only advisory and the server is free to discard as much
state as it needs to, references to streams which are no longer in the server's
state are treated as references to the root of the tree.  This can result in
wildly different conceptions of the priority tree between client and server, a
situation which all parties would prefer to avoid.

## Notational Conventions

The words "MUST", "MUST NOT", "SHOULD", and "MAY" are used in this document.
It's not shouting; when they are capitalized, they have the special meaning
defined in {{!RFC2119}}.

# The Priority Placeholder Extension

This extension consists of an additional setting {{setting}}, new flags on the
PRIORITY frame {{frame}}, and modified state management logic on the server
{{logic}}.

## Priority Placeholder Setting {#setting}

## PRIORITY Frame Modifications {#frame}

## Priority Management Logic {#logic}

# Security Considerations {#security}

TBD.

# IANA Considerations {#iana}

TBD.

--- back
