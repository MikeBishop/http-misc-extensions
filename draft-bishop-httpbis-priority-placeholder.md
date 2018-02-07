---
title: Priority Placeholders in HTTP/2
abbrev: Placeholders in HTTP/2
docname: draft-bishop-httpbis-priority-placeholder-latest
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
  RFC7540:

informative:
  I-D.ietf-quic-http:



--- abstract

RFC7540 defines HTTP/2, including a method for communicating priorities. Some
implementations have begun using closed streams as placeholders when
constructing their priority tree, but this has unbounded state commitments and
interacts poorly with HTTP/QUIC.  This document proposes an extension to the
HTTP/2 priority scheme for both protocols.

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

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{!RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# The Priority Placeholder Extension to HTTP/2

This extension consists of an additional setting {{setting}}, changes to the set
of HTTP/2 frames {{frames}}, and modified state management logic on the server
{{logic}}.

## Priority Placeholder Setting {#setting}

An HTTP/2 peer that supports Priority Placeholders indicates this using the
HTTP/2 `SETTINGS_PLACEHOLDERS` (0xSETTING-TBD) setting.

When a value for the `SETTINGS_PLACEHOLDERS` setting is not set, this indicates
that the peer does not support the extension, and other protocol elements in
this document MUST NOT be used.  A client that supports this extension SHOULD
set this value to 0 (0x00).

A server which supports this extension MUST set this value to a non-zero number
indicating the number of placeholders it is willing to make available to the
client, which MUST be at most 2^31-1.  Clients MUST NOT use the protocol
elements in this document unless the server has indicated support by setting a
non-zero value.

### Mid-session updates

HTTP/2 permits settings to change during the course of a connection.  This
setting can be freely increased at any time without consequence, and servers
SHOULD NOT reduce the value during the lifetime of a connection.

If a client receives a reduction in the number of permitted placeholders, it
MUST assume that all placeholders over the new limit have been pruned from the
tree and SHOULD immediately issue PRIORITY and PLACEHOLDER_PRIORITY frames as
necessary to rebuild the priority tree as desired.  Once the SETTINGS frame has
been acknowledged, servers should treat the excess placeholders as inactive and
prune them following the same logic in {{logic}}.

## Frame Modifications {#frames}

### Existing Frame Types

When client and server have opted in to this extension, the HTTP/2 PRIORITY
frame and HEADERS frame contain one additional flag:

DEPENDENT_ON_PLACEHOLDER (0x2):
: When set, bit 1 indicates that the value in the Stream Dependency field is a
  Placeholder ID rather than a Stream ID.

In HEADERS, this flag MUST NOT be set if the PRIORITY flag is not set.

### PLACEHOLDER_PRIORITY Frame

The PLACEHOLDER_PRIORITY (type=0xFRAME-TBD) frame specifies the sender-advised
priority of a placeholder. It MUST be sent only on Stream 0.  The semantics of
the Stream Dependency, Weight, and E flag are the same as in the HTTP/2 PRIORITY
frame.

The flags defined are:

  E (0x01):
  : Indicates that the stream dependency is exclusive (see {{!RFC7540}}, Section
    5.3).

  DEPENDENT_ON_PLACEHOLDER (0x2):
  : When set, bit 1 indicates that the value in the Stream Dependency field is a
    Placeholder ID rather than a Stream ID.


~~~~~~~~~~  drawing
    0                   1                   2                   3
    0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |0|                    Placeholder ID (31)                      |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |0|                  Stream Dependency (31)                     |
   +-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
   |   Weight (8)  |
   +-+-+-+-+-+-+-+-+
~~~~~~~~~~
{: #fig-priority title="PRIORITY frame payload"}

The PLACEHOLDER_PRIORITY frame payload has the following fields:

  Prioritized Stream:
  : A 31-bit stream identifier for the request stream whose priority is being
    updated.

  Stream Dependency:
  : A 31-bit stream or placeholder identifier for the request stream that this
    stream depends on (see {{!RFC7540}}, Section 5.3).

  Weight:
  : An unsigned 8-bit integer representing a priority weight for the stream (see
    {{!RFC7540}}, Section 5.3). Add one to the value to obtain a weight between
    1 and 256.

A PLACEHOLDER_PRIORITY frame MUST have a payload length of nine octets.  A
PLACEHOLDER_PRIORITY frame of any other length MUST be treated as a connection
error of type PROTOCOL_ERROR if the sender has advertised support for this
extension, and ignored otherwise.


## Priority Management Logic {#logic}

This extension provides a mechanism for servers to limit how many
additional IDs which do not refer to an active request will be used to maintain
priority state.  Because the server commits to maintain these inactive IDs,
clients can use them with confidence that the server will not have discarded
the state without warning.

In exchange, the server knows it can aggressively prune inactive regions from
the priority tree, because placeholders will be used to "root" any persistent
structure of the tree which the client cares about retaining.  For
prioritization purposes, a node in the tree is considered "inactive" when the
corresponding stream has been closed for at least two round-trip times (using
any reasonable estimate available on the server).  This delay helps mitigate
race conditions where the server has pruned a node the client believed was still
active and used as a Stream Dependency.

Specifically, the server MAY at any time:

  - Identify and discard branches of the tree containing only inactive nodes
    (i.e. a node with only other inactive nodes as descendants, along with those
    descendants)
  - Identify and condense interior regions of the tree containing only inactive
    nodes, allocating weight appropriately

~~~~~~~~~~  drawing
    x                x                 x
    |                |                 |
    P                P                 P
   / \               |                 |
  I   I     ==>      I      ==>        A
     / \             |                 |
    A   I            A                 A
    |                |
    A                A
~~~~~~~~~~
{: #fig-pruning title="Example of Priority Tree Pruning"}

In the example in {{fig-pruning}}, `P` represents a Placeholder, `A` represents
an active node, and `I` represents an inactive node.  In the first step, the
server discards two inactive branches (each a single node).  In the second step,
the server condenses an interior inactive node.  Note that these transformations
will result in no change in the resources allocated to a particular active
stream.

Clients MUST assume the server is actively performing such pruning and MUST NOT
declare a dependency on a stream it knows to have been closed.

# Incorporating Placeholders in HTTP/QUIC

HTTP/QUIC {{!HQ=I-D.ietf-quic-http}} uses a different PRIORITY frame which already
permits selecting either a stream or a Push ID (a new concept in HTTP/QUIC) to be
prioritized or used as a dependency.  Expanding this frame to support placeholders
as well requires additional bits.

The PRIORITY frame currently uses two flag bits to indicate Request/Push
dependencies on Request/Push.  If the full matrix of dependencies is to be
supported (Request/Push/Placeholder dependent on Request/Push/Placeholder), four
bits would be required to represent the space, with several invalid flag
combinations being defined.  (If one combination were eliminated, three
flags would be sufficient to represent the remaining combinations, but the
semantics of individual flags would be unclear.)

HTTP/QUIC does not have the implicit closure of streams like HTTP/2.  While
client implementations could reset streams which they intend to use as priority
placeholders, there has been interest in creating greater clarity and
synchronization between the client and server views of the priority tree.


# Security Considerations {#security}

This extension is believed to improve security relative to [RFC7540], as it
helps to constrain a previously unbounded state commitment.

# IANA Considerations {#iana}

This document registers one new frame type and one new setting.

## SETTINGS_PLACEHOLDERS Setting

The `SETTINGS_PLACEHOLDERS` setting is registered in the "HTTP/2 Settings"
registry established in [RFC7540].

Name:
: SETTINGS_PLACEHOLDERS

Code:
: 0xSETTING-TBD

Initial Value:
: not set

Specification:
: This document.

## PLACEHOLDER_PRIORITY Frame

The `PLACEHOLDER_PRIORITY` frame is registered in the "HTTP/2 Frames" registry
established in [RFC7540].

Name:
: PLACEHOLDER_PRIORITY

Code:
: 0xFRAME-TBD

Specification:
: This document.

--- back

# Acknowledgements

A substantial portion of Mike's work on this draft was supported by Microsoft
during his employment there.
