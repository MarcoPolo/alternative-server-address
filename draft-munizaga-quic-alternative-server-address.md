---
title: "QUIC Alternative Server Address Frames"
category: std
docname: draft-munizaga-quic-alternative-server-address-latest

ipr: trust200902
area: "Transport"
workgroup: "QUIC"
keyword: Internet-Draft
venue:
  group: "QUIC"
  type: "Working Group"
  mail: "quic@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/quic/"
  github: "MarcoPolo/alternative-server-address"
  latest: "https://marcopolo.github.io/alternative-server-address/draft-munizaga-quic-alternative-server-address.html"

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
 -
    fullname: Marco Munizaga
    organization: Ethereum Foundation
    email: marco@marcopolo.io
 -
    fullname: Marten Seemann
    email: martenseemann@gmail.com

normative:
  RFC9000:


informative:
  I-D.ietf-quic-multipath:

--- abstract

This document specifies an extension to QUIC that allows a server to advertise
a prioritized set of alternative addresses. This allows a client to migrate the
connection as the availability or preference of server addresses changes.

--- middle

# Introduction

QUIC supports client-initiated connection migration, allowing a connection to
survive changes to the client's address and enabling the client to select among
available paths ({{Section 9 of RFC9000}}). A server can advertise a preferred
address during the handshake ({{Section 9.6 of RFC9000}}), but cannot update
that address or advertise additional addresses during the connection.

Some deployments have multiple server addresses whose availability or preference
can change over the lifetime of a connection. These include multihomed
endpoints, relays, proxies, and peer-to-peer systems.

This document defines an extension that allows a server to advertise a
prioritized and replaceable set of alternative addresses. The client remains
responsible for validating these addresses and initiating any connection
migration.

Address discovery and NAT traversal mechanisms, including hole punching, are
out of scope of this document.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Negotiating Extension Use

alternative_address (0xff0969d85c):

Clients advertise their support of this extension by sending the
alternative_address (0xff0969d85c) transport parameter ({{Section 7.4 of
RFC9000}}) with an empty value. Sending this transport parameter signals
to the server that the client understands the ALTERNATIVE_ADDRESS frame.

Servers MUST NOT send this transport parameter. A client that supports this
extension and receives this transport parameter MUST abort the connection with a
TRANSPORT_PARAMETER_ERROR.

Endpoints MUST NOT remember the value of this extension for 0-RTT.

# Path Validation

Advertising an alternative address does not create a new path or initiate
connection migration. As in {{Section 9 of RFC9000}}, clients remain responsible
for initiating all connection migrations. A client initiates path validation by
sending probing packets to an advertised address and only migrates after
validation succeeds. Packets from unadvertised server addresses are handled as
specified by RFC 9000, except while validating an advertised address as
described below. Such packets do not create new paths.

The response to a client-initiated PATH_CHALLENGE can arrive from a server
address other than the address being validated. {{Section 8.2.2 of RFC9000}}
requires the server to send the PATH_RESPONSE on the path where it received the
PATH_CHALLENGE, but prohibits the client from enforcing this requirement. A
matching PATH_RESPONSE received on any path validates the path on which the
PATH_CHALLENGE was sent ({{Section 8.2.3 of RFC9000}}). The server might also
send a PATH_CHALLENGE to validate the path in its sending direction.
Consequently, while validating an advertised address, a client MUST NOT discard
a successfully authenticated probing packet solely because it was received from
an unadvertised server address. Processing the packet does not validate its
source address or make that address eligible for migration.

# Alternative Address Frame

A server uses an ALTERNATIVE_ADDRESS frame to advertise its complete set of
alternative addresses and their priority relative to the current path. Each
frame replaces the state established by any previously processed
ALTERNATIVE_ADDRESS frame.

The frame uses the following format, following the conventions described in
{{Section 12.4 of RFC9000}}:

~~~
ALTERNATIVE_ADDRESS Frame {
  Type (i) = 0x1d5845e2,
  Sequence Number (i),
  Entry Count (i),
  Address Entry (..) ...,
}
~~~

The Entry Count field contains the number of Address Entry fields in the frame.
An Address Entry starts with an 8-bit Address Type and has one of the following
formats:

~~~
CURRENT_PATH Entry {
  Address Type (8) = 0x00,
}

IPV4 Entry {
  Address Type (8) = 0x01,
  Priority Hint (i),
  IPv4 Address (32),
  IPv4 Port (16),
}

IPV6 Entry {
  Address Type (8) = 0x02,
  Priority Hint (i),
  IPv6 Address (128),
  IPv6 Port (16),
}
~~~

The Priority Hint field is a QUIC variable-length integer. On each side of the
CURRENT_PATH entry, lower values indicate higher priority and entries with the
same value form a priority group in which the server expresses no preference.
Entries on each side MUST appear in ascending Priority Hint order. Priority Hint
values have meaning only relative to other values on the same side of the
CURRENT_PATH entry; values on opposite sides are unrelated. A server can assign
the same value to every entry.

The CURRENT_PATH entry is a sentinel representing the server address of the
current path and carries no priority hint, address, or port. A frame MUST contain
exactly one CURRENT_PATH entry and MUST contain each IP address and port tuple at
most once. Receipt of a frame that violates these requirements, does not order
entries as required, or contains an unknown Address Type MUST be treated as a
connection error of type FRAME_ENCODING_ERROR.

IPV4 and IPV6 entries before CURRENT_PATH have higher priority than the current
path. The client SHOULD promptly validate these addresses and migrate to a
validated address. Entries after CURRENT_PATH are backup addresses. The client
MAY validate paths to these addresses, but SHOULD NOT migrate to one solely
because it was advertised.

Priority hints are advisory. A client MAY use them to decide which addresses to
validate, which validations to perform in parallel, and which validated address
to use. A client MAY disregard priority hints based on local policy.

A server MUST use a larger Sequence Number for each address-set update. A client
MUST ignore an ALTERNATIVE_ADDRESS frame whose Sequence Number is not greater
than that of the most recently processed ALTERNATIVE_ADDRESS frame.
Therefore, a newer frame atomically replaces an older address set even if the
frames are received out of order. An address omitted from the newer frame is no
longer advertised by this extension. The client SHOULD stop probing or using a
non-current path associated with an address that is no longer advertised.

ALTERNATIVE_ADDRESS frames are ack-eliciting and MUST only be sent in the
application data packet number space.

## Address Selection and Reachability

The mechanism by which a server discovers and selects addresses to advertise is
outside the scope of this document. An advertised address is a candidate and
does not imply reachability from the client. A server SHOULD limit the
advertised set to addresses that it has reason to believe might be reachable,
and SHOULD update the set when it learns that an address is no longer usable.

Advertised addresses can include private-use, unique-local, or other
limited-scope addresses. A client MAY decline to probe an address according to
local policy. A client MUST successfully validate a path before sending
non-probing frames on it. The request forgery considerations in Sections 21.5.3
and 21.5.6 of {{RFC9000}} apply.

# Connection ID Management

Each endpoint SHOULD advertise an active_connection_id_limit that allows its
peer to supply enough connection IDs for all paths that the endpoint might probe
concurrently. This applies in both directions.

The server SHOULD ensure that the client has a sufficient number of available
and unused connection IDs, as the client will be unable to probe paths without
an unused connection ID. The server MAY bundle one or more NEW_CONNECTION_ID
frames with an ALTERNATIVE_ADDRESS frame. Likewise, the client SHOULD ensure
that the server has enough connection IDs to probe new paths.

# Interaction with Managing Multiple Paths for a QUIC Connection

The mechanism described in {{I-D.ietf-quic-multipath}} enables a QUIC connection
to use multiple paths simultaneously. This extension complements that mechanism
by allowing the server to advertise addresses for alternative paths.

# Security Considerations

## Request Forgery Attacks

The same considerations from {{Section 21.5 of RFC9000}} apply here as
well.

## DDoS - Thundering herd

A malicious server could wait until it has received a large number of clients,
and request a migration from all of them at the same time to a victim endpoint.
If the clients all migrate at the same time, they may overload or otherwise
negatively impact the victim endpoint.

Clients may mitigate this by randomly delaying the migration.

# IANA Considerations

## QUIC Transport Parameter

This document registers the alternative_address transport parameter in the "QUIC
Transport Parameters" registry established in {{Section 22.3 of
RFC9000}}. The following fields are registered:

Value:
: 0xff0969d85c

Parameter Name:
: alternative_address

Status:
: Provisional

Specification:
: This document

Change Controller:
: IETF (iesg@ietf.org)

Contact:
: Marco Munizaga (marco@marcopolo.io)

## QUIC Frame Types

This document registers the ALTERNATIVE_ADDRESS frame in the "QUIC Frame Types"
registry established in {{Section 22.4 of RFC9000}}. The following fields are
registered:

Value:
: 0x1d5845e2

Frame Type Name:
: ALTERNATIVE_ADDRESS

Status:
: Provisional

Specification:
: This document

Change Controller:
: IETF (iesg@ietf.org)

Contact:
: Marco Munizaga (marco@marcopolo.io)

--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.

# Questions
{:numbered="false"}

- Any new security considerations from allowing a dynamically chosen preferred
  address?
