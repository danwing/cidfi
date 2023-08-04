---
title: "QUIC CID Flow Indicator (CIDFI)"
abbrev: "CIDFI"
category: std


docname: draft-wing-cidfi-latest
submissiontype: IETF
v: 3
area: "Network"
workgroup: "Network Working Group"
keyword:
 - customer experience
 - bandwidth
 - priority
 - enriched feedback
 - media streaming
 - realtime media
 - QoS
 - 5G
 - Wi-Fi
 - WiFi
 - DSCP

venue:
#  group: "Transport Area Working Group"
#  type: "Working Group"
#  mail: "tsvwg@ietf.org"
#  arch: "https://mailarchive.ietf.org/arch/browse/tsvwg/"
  github: "danwing/cidfi"
#  latest: "https://danwing.github.io/cidfi/draft-wing-cidfi.html"

stand_alone: yes
pi: [toc, sortrefs, symrefs, strict, comments, docmapping]

author:
 -
    fullname: Dan Wing
    organization: Cloud Software Group, Inc.
    abbrev: Cloud Software Group
    country: United States of America
    email: ["danwing@gmail.com", "dan.wing@cloud.com"]
 -
    fullname: Tirumaleswar Reddy
    organization: Nokia
    city: Bangalore
    region: Karnataka
    country: India
    email: "kondtir@gmail.com"
 -
    fullname: Mohamed Boucadair
    organization: Orange
    street: Rennes
    code: 35000
    country: France
    email: "mohamed.boucadair@orange.com"


normative:

informative:
  pathologies:
    title: Exploring DSCP modification pathologies in the Internet
    author:
      -
        name: Ana Custura
      -
        name: Raffaello Secchi
      -
        name: Gorry Fairhurst
    date: 2018-05
    target: https://www.sciencedirect.com/science/article/pii/S0140366417312835


--- abstract

Conveying metadata about network conditions and metadata about
individual packets between the network and server can improve the user
experience especially on wireless networks which incur significant
variability.

This document describes how a client and server can cooperate so
that their QUIC stream can be augmented with information about network
conditions and augmented with information about packet importance.

--- middle

# Introduction

This document gives an overview of CIDFI (pronounced "sid fee") which is
a system of several protocols that provide both the QUIC {{!RFC9300}} client and
the QUIC server with a list of the local network's CIDFI-aware network elements.
Both the client and the server authorize the use of each of the CIDFI-aware
network elements.  After authorization and activation of those CIDFI network
elements, the server sends metadata to
the network element and the network element sends metadata to the
server.  The metadata includes:
  - network characteristics from the network element to the server, such as bandwidth
  - DSCP mappings of a server's QUIC Connection IDs from the server towards the QUIC client

Below is a network diagram of a CIDFI system showing two
bandwidth-constrained networks (or links) depicted by "B" and
CIDFI-aware devices immediately upstream of those links, and another
bandwidth-constrained link between a smartphone handset and its Radio
Access Newwork (RAN).  This diagram shows the same protocol and same mechanism
can operate with or without 5G, and can operate with different administrative
domains such as WiFi and an edge router.

~~~~~
                           |                     |
+--------+     +--------+  |  +--------+         |
| CIDFI- |     | CIDFI- |  |  | CIDFI- |         |
| aware  |     | aware  |  |  | aware  |         |
| QUIC   +--B--+ WiFi   +--B--+ edge   +-+       |
| client |     | access |  |  | router |  \      |       +--------+
+--------+     | point  |  |  +--------+   \     |       | CIDFI- |
               +--------+  |              +-+-------+    | aware  |
                           |              | routers +----+ QUIC   |
+-----------+              |              +-+-------+    | server |
| CIDFI-    |              |  +--------+   /     |       +--------+
| aware     |              |  | CIDFI- |  /      |
| QUIC      +-------B------|--+ aware  +-+       |
| client    |              |  | RAN    |         |
| (handset) |              |  +--------+         |
+-----------+              |                     |
                           |                     |
 home/enterprise network   |    ISP network      |  server network
~~~~~
{: artwork-align="center" title="Network Diagram"}

This document attempts to solve the problems described in
{{?I-D.joras-sadcdn}}, {{?I-D.reddy-tsvwg-explcit-signal}}, and
{{?I-D.kaippallimalil-tsvwg-media-hdr-wireless}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

CID:
: QUIC Connection Identifier.  Unless short or long are called out
explicitly, this term covers both the value of the long header
(Section 17.2 of {{!RFC9300}}) Connection ID and the length and value
of the short header Connection ID (Section 17.3 of {{RFC9300}}).

side channel:
: communication between network element and server, or network element and client,
which occurs within the same 5-tuple as the (primary) QUIC channel between client
and server.


# Design Goals

Privacy:
: Only disclose packet importance to necessary network elements.  The
importance of a packet is signaled by its QUIC CID and the mapping of a CID to
its importance is communicated over an encrypted channel between the sender
and the network element.

Integrity:
: The QUIC CID is integrity protected by QUIC itself, and cannot
be modified by on-path network elements.

Internet Survival:
: Considering DSCP has poor survival rate across
the Internet {{pathologies}} and each network has its own DSCP
definition, the protocol described in this document allows signaling a
mapping from QUIC CID to a specific DSCP code point. The network
element can use this mapping to inform its traditional DSCP packet
handling with higher confidence of the sender's DSCP intent.

Side-Channel to Same Server:
: The side-channel communcation is time-sensitive and needs to reach
the same server.  A design using the same 5-tuple for the side-channel
as for the primary client-server channel provides best chance to reach
the same server through load balancers.

Client Authorization:
: The client needs to authorize network participation in CIDFI.


# Network Preparation: DNS SVCB records

The local network is configured to respond to DNS SVCB
{{!I-D.ietf-dnsop-svcb-https}} queries for _cidfi.cidfi.arpa with the
DNS names of that network's and upstream network's CIDFI network
elements.  If upstream networks, such as ISP networks, also support
CIDFI, those are aggregated into the local DNS server's response by
normal behavior of recursive DNS resolvers.

When multihoming, the multihome-capable CPE needs to aggregate
all upstream networks' _cidfi.cidfi.arpa responses into the response
sent to its locally-connected clients.


# Client Operation on Network Attach or Topology Change

On network attach or detected topology change (see {{topology}}), the
client determines if the network supports CIDFI and authorzes those
network elements.

## Client learns local network supports CIDFI {#discovery}

> ** Note: For this section, a different approach using probe packets is described in {{probe}}
  but the authors are currently not pursuing that technique (additional traffic, additional
  delay to establish CIDFI on a new connection).  It is retained for discussion.

The client determines if the local network provides CIDFI service by:

* issuing a query to the local DNS server for _cidfi.cidfi.arpa. with
the SVCB resource record type (64) {{I-D.ietf-dnsop-svcb-https}}.  If
this succeeds, processing skips to {{client-authorizes}}.

* If the above DNS response returned no answer, the client determines
its public IP address (e.g., https://api.ipify.org or STUN to a
publicly-accessible STUN server) and issues a DNS SVCB query for
_cidfi in its reverse DNS name.  For example if its public address is
203.0.113.1 it would issue an SVCB query for
_cidfi.1.113.0.203.in-addr.arpa.  This provides a way to immediately
deploy CIDFI, without requiring customer premise equipment support for
DNS SVCB resource records or CIDFI aggregation.  However, it has the
disadvantage of additional network traffic each time a client attaches
to a network.  See also {{discuss-in-addr}}.


If both techniques above failed it indicates the local network does
not support CIDFI and processing stops.


## Client authorizes CIDFI network elements {#client-authorizes}

The SVCB response from the previous step will contain one or more
CIDFI servers.  For example a CIDFI server belonging to the local
Wi-Fi network and another CIDFI server belonging to the Internet
Service Provider (ISP).  In cases of (active/active or active/standby)
multihoming, multiple ISPs might be provided.

The client authorizes each of the CIDFI endpoints using its local
policy.  This policy might prompt the user, allow certain names (e.g.,
*.example.net if the user's ISP is configured to be example.net),
connect to the CIDFI servers and validate certificate or certificate
chains, and so forth.

After authorizing a subset of the CIDFI servers, the client connects
to that subset of authorized CIDFI servers and obtains their
certificate fingerprints and capabilities (ability to report network
constraints, ability to perform CID-to-DSCP mapping) for use when
communicating with a CIDFI-aware QUIC server.

TODO: specify the encoding of the above information.


# Operation on Each Connection to a Server

On each connection to a server:

  1. On connection to server, client learns server supports CIDFI.
  2. Client contacts the network elements learned from step (1) and requests
     their participation.
  3. Network elements connect to the server by using the same 5-tuple
     as the client's existing QUIC session with the server.  This is the metadata
     side channel.
  4. Over that side channel, the QUIC server communicates its QUIC CID-to-DSCP mappings to the network
     elements and the network elements send network performance data to the server.

These steps are described below.

> Note: the client is also a sender, and can also perform all these
functions in its direction.  This functionality will be expanded in
later versions of this document.  For example, a mobile device
connected to Wi-Fi with 5G backhaul might be running an interactive
audio/video application needs to indicate to its internal Wi-Fi driver
and to the 5G modem its mapping from QUIC CID to DSCP and the
application needs to receive network performance metrics.



## Client Learns Server Supports CIDFI {#server-supports-cidfi}

On initial connection to a QUIC server, the client includes a new QUIC
transport parameter CIDFI which is remembered for 0-RTT.

If the server does not indicate CIDFI support, processing stops.

If the server indicates CIDFI support, then:

 * the CIDFI server sends the TLS SNI it wants to use for the
   incoming side-channel communications from the CIDFI network
   element.  This is necessary because the CIDFI network element does
   not have visibility to the SNI of the primary QUIC connection
   ({{?I-D.ietf-tls-esni}}).  See also {{side-channel-certificate}}
   for an alternative approach that has better privacy properties.

 * the server sends the QUIC Connection IDs it will use for its
   server-to-network element connection, which are reserved for its
   use (that is, cannot be used by the QUIC client for its primary
   communication to the server).

 * the client sends the certificate fingerprints of the authorized
   network element(s) to the server, which were obtained in {{client-authorizes}}.

TODO: specify the encoding of the above information.

## Client Requests Network Element Participation

Using its QUIC channel with each of the CIDFI network elements, the
client signals both the long Connection ID and the short Connection ID
length and Connection ID of the primary communication to the network
element.  As QUIC allows changing the Connection ID and to avoid loss
of CIDFI functionality, the client SHOULD additionally signal the next
long and short Connection ID it anticipates using with the server.  The
CIDFI network element MUST NOT use those signaled CIDs for its own
communication with the server.

The client obtains the CIDFI network element's list of reserved QUIC
CIDs (see {{collision}}).

Note the source IP address and source UDP port are not signaled by
design.  This is because NATs ({{?NAPT=RFC3022}}, {{?NAT=RFC2663}}),
multiple NATs on the path, IPv6/IPv4 translation, and similar
technologies complicate accurate signaling of the source IP address
and source UDP port.

After receiving this request, each network element initiates a QUIC
connection to the server (see also {{side-channel-certificate}})
using mutual TLS, which are validated against the certificate fingerprints
which were provided in {{server-supports-cidfi}}.  This QUIC connection
uses the reserved QUIC Connection IDs that were communicated to the



## Metadata Exchanged

There are two types of metadata exchanged, described below.

### Server to Network Elements {#server-to-network}

To each of the network elements, the serverr sends its mapping
of QUIC CIDs to DSCP code points.

To allow the QUIC endpoints to change their QUIC CIDs and preserve the
network element treatment of the new CID, the 'next CID' MUST be
communicated to the network elements prior to switching to the new
CID.  This can be accomplished by sending the new CID well in advance
of changing CIDs.


### Network Element to Server {#network-to-server}

The network element send network performance information to the
server.  This performance information pertains to this connection's
allowed (or available) bandwidth, burst rate, and other information
such as defined in {{I-D.kaippallimalil-tsvwg-media-hdr-wireless}}.

This information is sent whenever it changes significantly, but MUST
NOT be sent more frequently than once every second.


# CID Collision {#collision}

The QUIC client and server can change their QUIC CIDs.  A CID collision
would occur if those CIDs are also used by the network element(s) on the
side-channel, which shares the same 5-tuple.

To avoid CID collision the network elements and server sends a set of
CIDs they will use for their side channel communication through the
client.  The client is the only party that has a communication path to
the CIDFI network element(s) and the server.  This set of reserved
CIDs is purposefully small (~4) so that the QUIC client and server are
able to use almost their entire CID space.

TODO: more details.

# Topology Change {#topology}

When topology changes (such as switching to a backup WAN connection,
or such as switching from Wi-Fi to 5G), the QUIC server will consider
this a connection migration and will issue a PATH_CHALLENGE.

Upon receipt of PATH_CHALLENGE the CIDFI-aware client SHOULD
re-discover its CIDFI network elements {{discovery}}.  If that
set of network elements differs from the previous set, the client
SHOULD continue with normal CIDFI processing.

Another signal that topology has changed is if the QUIC client
receives a QUIC packet with one of the CIDFI-reserved Connection
IDs (see {{collision}}).


# Discussion Points

This section discusses the issues that benefit from wider discussion.

## Overhead of Packet Examination

If CID-to-DSCP mapping was signaled by the server as described in
{{server-to-network}}, the edge router has to examine each packet for
a matching CID for the lifetime of the connection.

It may be helpful to DSCP mark the packet after examination to reduce
their burden of examining the packets; however, this requires
coordination of DSCP meanings in those downstream networks which would
would persist as an operational burden.


## Overhead of Mapping CID to DSCP

Network Elements have to maintain per-5-tuple mapping of QUIC CID to
DSCP, which needs updating whenever sender changes its CID.  This is
awkward.

An alternative is a fixed mapping of QUIC CIDs to their meanings,
as proposed in {{?I-D.zmlk-quic-te}}.  However, this will ossify
the meaning of those QUIC CIDs.  It also requires all networks to
agree on the meaning of those QUIC CIDs.

## Discovery using in-addr.arpa  {#discuss-in-addr}

The in-addr.arpa discovery technique in {{discovery}} incurs load on
arpa servers (not cool) and we might never be able to sunset that
mechanism entirely.  Need other ideas.

## Privacy Side-Channel Certificate {#side-channel-certificate}

When providing its certificate for authenticating the network
element-to-server connection, the server is likely to share
identifying information.  This discloses additional information to the
network operator which may have been encrypted {{?I-D.ietf-tls-esni}}.
While raw public keys {{?RFC7250}} offers some relief, raw public keys
can still be correlated with known servers.

An idea: client performs the side-channel QUIC handshake with server
then hands QUIC security context to the CIDFI network element, which
takes over that QUIC connection on that same 5-tuple.  This means the
CIDFI network element does not perform QUIC handshake with the server
and never knows server's CIDFI certificate.  This incurs a little more
air time traffic while retaining server identity privacy.

## Delay to Start CIDFI

We want to reduce network communications to start CIDFI.



# Security Considerations

TODO Security


# IANA Considerations

## New QUIC Transport Parameter

Register new QUIC transport parameter "CIDFI", remembered for 0-RTT.

## Special-use Domain Name

Register new special-use domain name cidfi.arpa.

--- back



# Probe-based Discovery {#probe}
{:removeInRFC="true"}

This section describes an alternative to {{discovery}} which uses
probe packets rather than DNS SVCB.

Client Sending Operation:
: The client sends a STUN request packet on same
5-tuple as its existing QUIC connection with the server.  This STUN
request contains a new STUN attribute, CIDFI.

: To avoid packet growth while traversing CIDFI-aware nodes, this packet
originated from the client contains is 512 0x00 octets.  This is
enough two full-sized (255-byte) fully-qualified domain name (FQDN)
fields.  In practice, FQDNs have a shorter length (~50 bytes) so 10
FQDNs could reasonably fit into this space.

Network Element Processing:
: Each CIDFI-aware node on the path increments the FQDN counter and
overwrites the 0x00 octets with its FQDN, decrements its IPv4 TTL or
IPv6 Hop Count, and sends the packet along towards its destination.

Server Receipt Operation:
: Upon receipt of the STUN Request containing CIDFI, the server generates
a STUN response containing a copy of the received CIDFI.

Client Receipt Operation:
: The client validates the STUN Transaction-ID.  The FQDNs are handled
in Step 3, below.

The packet format would be a {{!RFC8489}} request packet containing
a new STUN "CIDFI" attribute.  The STUN "CIDFI" attribute in the request would
contain 512 null octets.  The 'counter' field allows network elements to
increment that counter and, if fewer FQDNs are inside the packet, it indicates
the packet was not big enough to contain all the desired FQDNs and a larger
packet should be originated by the client.  However, this code path is unlikely to
get much exercise.

~~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|counter|0 0 0 0| FQDN-1 length |      FQDN-1 ...               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
| FQDN-2 length |                  FQDN-2 ...                   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~
{: artwork-align="center" #fig-cifi-stun-attribute title="CIDFI STUN Attribute"}








# Acknowledgments
{:numbered="false"}

TODO acknowledge.

