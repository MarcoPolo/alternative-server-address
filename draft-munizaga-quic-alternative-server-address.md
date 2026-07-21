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

--- abstract

This document specifies an extension to QUIC that allows a server to advertise
an ordered set of alternative addresses.

--- middle

# Introduction

The QUIC transport protocol allows a client to migrate connections at any time
to any new address ({{Section 9 of RFC9000}}). This allows the connection
to survive changes to the client's address. A client can use this mechanism to
keep redundant paths available or transparently move to a different local
address. A server, in contrast, can not use alternative addresses as redundant
paths and has no way to dynamically signal a preferred address. In some
deployments, specifically peer to peer settings, adding this symmetry is useful.

This document specifies an extension to QUIC that allows a server to inform a
client of alternative, possibly preferred, addresses.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Motivation

In peer to peer networks, the role of server and client is arbitrary. An
endpoint may serve as a client in one connection and a server in another.
A peer acting as a server would like to communicate to its peer its alternative
addresses. The server peer does this for both redundancy (a peer may advertise a
globally reachable relayed unicast address as a backup) and to signal preference
(a peer may be using a proxy, and wish to migrate to a new proxy).

While it is not the primary goal, this extension may also assist in NAT
traversal by migrating to a dynamically chosen server address. A server could
have a client connect over a relay, and later migrate to a direct connection
after applying NAT traversal techniques. The specific NAT traversal techniques
are out of scope of this document.

TODO: Is the above NAT paragraph useful? Would it be better to leave this
implied?

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
  IPv4 Address (32),
  IPv4 Port (16),
}

IPV6 Entry {
  Address Type (8) = 0x02,
  IPv6 Address (128),
  IPv6 Port (16),
}
~~~

Entries are ordered from highest to lowest priority. The CURRENT_PATH entry is
a sentinel representing the server address of the current path and carries no
address or port. A frame MUST contain exactly one CURRENT_PATH entry and MUST
contain each IP address and port tuple at most once. Receipt of a frame that
violates these requirements or contains an unknown Address Type MUST be treated
as a connection error of type FRAME_ENCODING_ERROR.

IPV4 and IPV6 entries before CURRENT_PATH have higher priority than the current
path. The client SHOULD promptly validate these addresses and migrate to the
highest-priority address for which path validation succeeds. Entries after
CURRENT_PATH are backup addresses. The client MAY validate paths to these
addresses, but SHOULD NOT migrate to one solely because it was advertised.

A server MUST use a larger Sequence Number for each address-set update. A client
MUST ignore an ALTERNATIVE_ADDRESS frame whose Sequence Number is not greater
than that of the most recently processed ALTERNATIVE_ADDRESS frame.
Therefore, a newer frame atomically replaces an older address set even if the
frames are received out of order. An address omitted from the newer frame is no
longer advertised by this extension. The client SHOULD stop probing or using a
non-current path associated with an address that is no longer advertised.

# Frame properties

ALTERNATIVE_ADDRESS frames are ack-eliciting and MUST only be sent in the
application data packet number space.

The server SHOULD ensure that its peer has a sufficient number of available and
unused connection IDs, as the client will be unable to probe paths without an
unused connection ID. The server MAY bundle one or more NEW_CONNECTION_ID frames
with an ALTERNATIVE_ADDRESS frame. Likewise, the client should ensure the same
to allow the server to probe new paths.

# Interaction with the Multipath Extension for QUIC

This extension complements the Multipath extension for QUIC by allowing the
server to contribute more information to the client for alternative paths.

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
