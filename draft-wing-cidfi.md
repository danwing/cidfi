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

  wifi-aggregation:
    title: "Ending the Anomaly: Achieving Low Latency and Airtime Fairness in WiFi"
    author:
      -
        name: Toke Høiland-Jørgensen
      -
        name: Michał Kazior
      -
        name: Dave Täht
      -
        name: Per Hurtig
      -
        name: Anna Brunstrom
    target: https://www.usenix.org/conference/atc17/technical-sessions/presentation/hoilan-jorgesen
    date: 2017-05-22

  IANA-QUIC:
    title: QUIC
    target: https://www.iana.org/assignments/quic/quic.xhtml
    date: 2023-07-26

  IANA-WKU:
    title: "Well-known URIs"
    target: https://www.iana.org/assignments/well-known-uris/well-known-uris.xhtml
    date: 2023-06-20

  DSCP-Registry:
    title: Differentiated Services Field Codepoints (DSCP)
    target: https://www.iana.org/assignments/dscp-registry/dscp-registry.xhtml
    date: 2023-07-21



--- abstract

Conveying metadata about network conditions and metadata about
individual packets between the network and server can improve the user
experience especially on wireless networks which incur significant
variability.

This document describes how clients and servers can cooperate so
that their QUIC streams can be augmented with information about network
conditions and packet importance.


--- middle

# Introduction

This document gives an overview of CIDFI (pronounced "sid fye") which is
a system of several protocols that provides both the QUIC {{!RFC9300}} client and
the QUIC server with a list of the local network's CIDFI-aware network elements.
Both the client and the server authorize the use of each of the CIDFI-aware
network elements in a QUIC connection.  After authorization and activation of those CIDFI network
elements, the server sends metadata to
a network element and the network element sends metadata to the
server.  The order and directionality of supplying metadata is service-specific. The metadata includes:

  - Network characteristics from the network element to the server, such as bandwidth
  - Mapping server's transmitted DCIDs on primary QUIC connection to metadata, such as priority


{{fig-arch}} provides a sample network diagram of a CIDFI system showing two
bandwidth-constrained networks (or links) depicted by "B" and
CIDFI-aware devices immediately upstream of those links, and another
bandwidth-constrained link between a smartphone handset and its Radio
Access Newwork (RAN).  This diagram shows the same protocol and same mechanism
can operate with or without 5G, and can operate with different administrative
domains such as Wi-Fi and an edge router.

~~~~~
                           |                     |
+--------+     +--------+  |  +--------+         |
| CIDFI- |     | CIDFI- |  |  | CIDFI- |         |
| aware  |     | aware  |  |  | aware  |         |
| QUIC   +--B--+ Wi-Fi  +--B--+ edge   +-+       |
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
{: #fig-arch artwork-align="center" title="Network Diagram"}

The QUIC-encrypted communication between the CIDFI-aware network
element and the CIDFI-aware server is a side channel which uses the
same UDP 4-tuple as the (primary) QUIC connection between the QUIC
client and the QUIC server.  In the figure below, steps (1) through
(3) are a normal QUIC handshake, and steps (4) through (6) are the
side-channel QUIC handshake.

~~~~~
203.0.113.1/12345
  +--------+         +---------+                  203.0.113.2/443
  | CIDFI- |         | CIDFI-  |                     +--------+
  | aware  |         | aware   |                     | CIDFI- |
  | QUIC   |         | network |                     | aware  |
  | client |         | element |                     | server |
  +---+----+         +----+----+                     +---+----+
      |                   |                              |
      |    1. QUIC Initial (Client Hello, handshake)     |
      |------------------------------------------------->|
      |    2. QUIC Initial (Server Hello, handshake)     |
      |<-------------------------------------------------|
      |    3. QUIC Finish, Data                          |
      |------------------------------------------------->|
      |                   |                              |
      |                   | 4. QUIC Init. (Client Hello) |
      |                   | source = 203.0.113.1/12345   |
      |                   |----------------------------->|
      |                   | 5. QUIC Init. (Svr Hello, handshake)
      |                   | dest = 203.0.113.1/12345     |
      |                   |<-----------------------------|
      |                   | 6. QUIC client cert, Fin, data
      |                   | source = 203.0.113.1/12345   |
      |                   |----------------------------->|
~~~~~
{: artwork-align="center" title="side-channel ladder diagram"}

This document attempts to solve the problems described in
{{?I-D.joras-sadcdn}}, {{?I-D.reddy-tsvwg-explcit-signal}}, and
several metadata parameters of {{?I-D.kaippallimalil-tsvwg-media-hdr-wireless}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

CID:
: QUIC Connection Identifier.  Unless short or long are called out
explicitly, this term covers both the value of the long header
({{Section 17.2 of !RFC9300}}) Connection ID and the length and value
of the short header Connection ID ({{Section 17.3 of RFC9300}}).

Side channel:
: a communication between a network element and a server, or a network element and a client.
The side channel between a netwrok element and a server occurs within the same UDP 4-tuple as the (primary) QUIC connection between client
and server.



# Design Goals

Privacy:
: Only disclose packet importance to necessary network elements.  The
importance of a packet is signaled by its QUIC CIDs and the mapping of a CID to
its importance is communicated over an encrypted channel between the sender
and the network element.

Integrity:
: The QUIC CIDs are integrity protected by QUIC itself, and cannot
be modified by on-path network elements.

Internet Survival:
: Metadata signaling, per-packet signaling,
(in the side channel) of both per-packet metadata (in the server-to-network element direction) and
which maps to both primary channeusing QUIC Destination CID) is expected to surpass the
survival of other mechanisms and to reduce operational burden of non-CIDFI-aware
routers.

Side-Channel to Same Server:
: The side-channel communcation is time-sensitive and needs to reach
the same server.  A design using the same UDP 4-tuple for the side-channel
as for the primary client-server QUIC channel provides best chance to reach
the same server.  See also {{load-balancers}}.

Client Authorization:
: The client needs to authorize network participation in CIDFI.


# Network Preparation: DNS SVCB Records

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

## Client Learns Local Network Supports CIDFI {#discovery}

> ** Note: For this section, a different approach using probe packets is described in {{probe}}
  but the authors are currently not pursuing that technique (additional traffic, additional
  delay to establish CIDFI on a new connection).  It is retained for discussion.

The client determines if the local network provides CIDFI service by:

* Issuing a query to the local DNS server for _cidfi.cidfi.arpa. with
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


## Client Authorizes CIDFI Network Elements {#client-authorizes}

The SVCB response from the previous step in {{discovery}} will contain one or more
CIDFI-aware network elements.  For example a CIDFI-aware network element belonging to the local
Wi-Fi network and another CIDFI-aware network element belonging to the Internet
Service Provider (ISP).  In cases of (active/active or active/standby)
multihoming, multiple ISPs might be provided.

The client authorizes each of the CIDFI-aware network elements using its local
policy.  This policy might prompt the user, allow certain names (e.g.,
*.example.net if the user's ISP is configured to be example.net),
connect to the CIDFI servers and validate certificate or certificate
chains, and so forth.

After authorizing a subset of the CIDFI-aware network elements, the
client makes a new connection (on a new 5-tuple) to those CIDFI-aware
network elements and obtains their certificate fingerprints and
metadata that will be exchanged (see {{metadata-exchanged}}) for later use
when communicating with a CIDFI-aware QUIC server.

TODO: specify the encoding of the above information.


# Operation on Each Connection to a Server

On each connection to a server:

  1. On connection to server, client learns server supports CIDFI ({{server-supports-cidfi}})
  2. Client contacts the network elements learned from {{discovery}} and requests
     their participation.
  3. Network elements connect to the server by using the same UDP 4-tuple
     as the client's existing QUIC session with the server.  This is the metadata
     side channel.
  4. Over that side channel, the QUIC server communicates its mapping from transmitted
     QUIC Destination CIDs to packet metadata.  Over that same side channel the CIDFI network
     element communicates network performance data to the server.

These steps are described below.

> Note: the client is also a sender, and can also perform all these
functions in its direction.  This functionality will be expanded in
later versions of this document.  For example, a mobile device
connected to Wi-Fi with 5G backhaul might be running an interactive
audio/video application needs to indicate to its internal Wi-Fi driver
and to the 5G modem its mapping from QUIC Destination CID to per-packet
metadata and the application needs to receive network performance metrics.



## Client Learns Server Supports CIDFI {#server-supports-cidfi}

On initial connection to a QUIC server, the client includes a new QUIC
transport parameter CIDFI ({{iana-tp}}) which is remembered for 0-RTT.

If the server does not indicate CIDFI support, processing stops.

If the server indicates CIDFI support, then:

 * the CIDFI server sends the TLS SNI it wants to use for the
   incoming side-channel communications from the CIDFI network
   element.  This is necessary because the CIDFI network element does
   not have visibility to the SNI of the primary QUIC connection
   ({{?I-D.ietf-tls-esni}}).  See also {{side-channel-certificate}}
   for an alternate approach that has better privacy properties.

 * the server sends the QUIC Connection IDs it will use for its
   server-to-network element connection, which are reserved for its
   use (that is, cannot be used by the QUIC client for its primary
   communication to the server).

 * the client sends the certificate fingerprints of the authorized
   network element(s) to the server, which were obtained in {{client-authorizes}}.

TODO: specify the encoding of the above information.

## Client Requests Network Element's Participation

Using its QUIC channel with each of the CIDFI network elements it previously authorized for CIDFI participation, the
client signals both the long Connection ID and the short Connection ID
length and Connection ID of the primary communication to the network
element.  As QUIC allows changing the Connection IDs and to avoid loss
of CIDFI functionality, the client SHOULD additionally signal the next
long and short Connection ID it anticipates using with the server.  The
CIDFI network element MUST NOT use those signaled CIDs for its own
communication with the server.

The client obtains the CIDFI network element's list of reserved QUIC
CIDs (see {{collision}}).

Note that source IP address and source UDP port number are not signaled by
design.  This is because NATs ({{?NAPT=RFC3022}}, {{?NAT=RFC2663}}),
multiple NATs on the path, IPv6/IPv4 translation, and similar
technologies complicate accurate signaling of the source IP address
and source UDP port number.

After receiving a request for participation from a client, each network element initiates a QUIC
connection to the server (see also {{side-channel-certificate}})
using mutual TLS, which are validated against the certificate fingerprints
whiich are provided in {{server-supports-cidfi}}.  This QUIC connection
uses the reserved QUIC Connection IDs that were communicated to the client by the server.




## Mechanism for Metadata Exchange

There are two types of metadata exchanged, described in the following sub-sections.

### Server to Network Elements {#server-to-network}

To each of the network elements, the server sends its mapping
of QUIC CIDs to packet metadata (see {#packet-metadata}).

To allow the QUIC endpoints to change their QUIC CIDs and preserve the
network element treatment of the new CID, the same Destination CIDs
communicated to the peer (as described in {{Section 5.1.1 of RFC9000}})
MUST be communicated to the CIDFI network element.

For each network element sending metadata in the same UDP 4-tuple, the
network element's Destination CID is not counted against the primary
QUIC connection's active_connection_id_limit (see {{Section 5.1.1 of RFC9000}}).

The actual metadata exchanged over this channel is described in {{metadata-exchanged}}.

### Network Element to Server {#network-to-server}

The network element send network performance information to the server
which is intended to influence the sender'ss traffic rate (such as
improving or reducing fidelity of the audio or video).

This information is sent whenever it changes significantly, but MUST
NOT be sent more frequently than once every second.

The actual metadata exchanged over this channel is described in {{metadata-exchanged}}.

# Interaction with Load Balancers {#load-balancers}

ECMP load balancers ({{Section 5.2.3 of !RFC9000}} work with this mechanism unchanged, as the
source 3-tuple is the same for the primary QUIC session and the
side-channel QUIC session.  Similarly an ECMP load balancer that
front-ends a CID load balancer will send both the primary and
side-channel QUIC sessions to the same back-end CID load balancer.

With a CID load balancer, the back end server and load balancer MUST
coordinate using {{!I-D.ietf-quic-load-balancers}} or a proprietary
mechanism (e.g., encoding server identity within the short header
destination CID) so the same QUIC server receives the side-channel
communication.

# CID Collision {#collision}

The QUIC client and server can change their QUIC CIDs.  A CID collision
would occur if those CIDs are also used by the network element(s) on the
side-channel which shares the same UDP 4-tuple.

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

If the CIDFI-aware client is otherwise unaware of a topology change
and receives a PATH_CHALLENGE then the CIDFI-aware client SHOULD
re-discover its CIDFI network elements {{discovery}}.  If that
set of network elements differs from the previous set, the client
SHOULD continue with normal CIDFI processing.

Another signal that topology has changed is if the QUIC client
receives a QUIC packet with one of the CIDFI-reserved Connection
IDs (see {{collision}}).


# Details of Metadata Exchanged {#metadata-exchanged}

This section describes the metadata that can be exchanged from the
CIDFI-aware network element to the server (generally network
performance information) and from the server to the CIDFI-aware
network element.

## CIDFI-aware Network Element to Server

## Server to CIDFI-aware Network Element

### Mapping Metadata Parameters to DCIDs {#metadata-parameters}

Several of the metadata parameters from {{Section 4.2 of
I-D.kaippallimalil-tsvwg-media-hdr-wireless}} can be mapped to QUIC
Destination CID.

Importance:
: As described in {{Section 4.2.5 of I-D.kaippallimalil-tsvwg-media-hdr-wireless}}.

Data burst:
: As described in {{Section 4.2.6 of I-D.kaippallimalil-tsvwg-media-hdr-wireless}}.

Delay budget:
: The time, in milliseconds, before this packet has no value to the receiver.  Because
{{I-D.kaippallimalil-tsvwg-media-hdr-wireless}} defines a multi-packet "MDU"
identifier but QUIC Destination ID does not, the delay budget functionality
is diminished with CIDFI.

The other metadata parameters defined in
{{I-D.kaippallimalil-tsvwg-media-hdr-wireless}} -- Timestamp, MDU
Sequence, and Packet Counter -- are not available with CIDFI because they cannot
be mapped to a QUIC Destination CID.

Over the side-channel QUIC connection, the server sends an HTTP PUT
request to the well-known URI ".cidfi/metadata-parameters" with an
application/json body containing JSON indicating the Destination CID
length and the mapping of importance, burst, and delay budget to the
Destination CIDs:

~~~~~
  {"dcidlength":3,
     "map":[
       {"importance":17,"burst":83,"delaybudget":71,"dcids":[551,381]},
       {"importance":3,"burst":888,"delaybudget":180,"dcids":[89,983]},
       {"importance":7,"burst":37,"delaybudget":55,"dcids":[33]}]}
~~~~~

The CIDFI-aware network element responds with 200 Ok.

### Mapping DiffServ Code Point (DSCP) to DCIDs

A mapping from QUIC Destination CID to DiffServ code point
{{!RFC2474}} leverages existing DiffServ handling that may already
exist in the CIDFI network element.  If there are downstream network
elements configured with the same DiffServ code point the CIDFI
network element could mark the packet with that code point as well.

Signaling the DiffServ values for different QUIC Destination CID
increases the edge network's confidence the sender's DiffServ intent
is preserved into the edge network, even if the DSCP bits were
modified en route to the edge network (e.g., {{pathologies}}).

Over the side-channel QUIC connection, the server sends an HTTP
PUT request to the well-known URI ".cidfi/dscp" with an application/json
body containing JSON indicating the Destination CID length and mapping
the DSCP code-point from {{DSCP-Registry}} to the Destination CIDs:

~~~~~
  {"dcidlength":3,
     "map":[
       {"dscp":10,"dcids":[551,381]},
       {"dscp":46,"dcids":[89,983]},
       {"dscp":12,"dcids":[33]}]}
~~~~~
{: #fig-dscp-json artwork-align="left" title="Example JSON for DSCP Mapping"}

The CIDFI-aware network element responds with 200 Ok.

TODO: define ABNF


## CIDFI-aware Network Element to Server

Bandwidth.  Other parameters from {{I-D.joras-sadcdn}}.

TODO: fill in this section.


# Discussion Points

This section discusses the issues that benefit from wider discussion.

## Overhead of Packet Examination

If CID-to-packet metadata was signaled by the server as described in
{{server-to-network}}, the edge router has to examine the UDP payload
of each packet for a matching Destination CID for the lifetime of the
connection.


## Interaction with Wi-Fi Packet Aggregation

Per-packet metadata influences transmission of that packet but may
well conflict with some Wi-Fi optimizations (e.g., {{wifi-aggregation}})
and similar 5G optimizations.

his impact needs further study.


## Overhead of Mapping CID to packet metadata

Network Elements have to maintain a mapping between each UDP 4-tuple
and QUIC CID and its DSCP code point.  This also needs updating
whenever sender changes its CID.  This is awkward.

An alternative is a fixed mapping of QUIC CIDs to their meanings,
as proposed in {{?I-D.zmlk-quic-te}}.  However, this will ossify
the meaning of those QUIC CIDs.  It also requires all networks to
agree on the meaning of those QUIC CIDs.


## Discovery using in-addr.arpa  {#discuss-in-addr}

: The in-addr.arpa discovery technique in {{discovery}} incurs load on
arpa servers (not cool) and we might never be able to sunset that
mechanism entirely.  Need other ideas when SVCB can't be used.


## Privacy of Side-Channel Certificate {#side-channel-certificate}

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


## Improve CIDFI Initialization time

Find approaches to further reduce network communications to start CIDFI.


## Primary QUIC Channel CID Change {#primary-cid-change}:

Because the CIDFI network element, QUIC server, and QUIC client all
cooperate to share the primary QUIC connection's Destination CID,
when a new CIDFI network element is involved (e.g., due to client
attaching to a different network), a new Destination CID SHOULD
be used for the reasons discussed in {{Section 9.5 of !RFC9000}}}.

Do we want/need CIDFI network element to change the Destination CID of
its metadata communication in conjunction with the client changing its
Destination CID?



# Security Considerations

TODO Security


# IANA Considerations

## New QUIC Transport Parameter {#iana-tp}

This document requests IANA to register the following new permanent QUIC transport parameter
in the "QUIC Transport Parameters" registry under the "QUIC" registry group available at {{IANA-QUIC}}:

|Value| Parameter Name| Reference|
|TBD1| CIDFI| This-Document|
{: title="New QUIC Transport Parameter"}


## New Well-known URI "cidfi"

This document requests IANA to register the new well-known URI "cidfi" in the
"Well-Known URIs" registry available at {{IANA-WKU}}.

## New Special-use Domain Name

Register new special-use domain name cidfi.arpa for DNS SVCB discovery.

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

Thanks to Dave Täht, Magnus Westerlund, Christian Huitema, Gorry Fairhurst,
and Tom Herbert for hallway discussions and feedback at TSVWG that encouraged
the authors to consider the approach described in this document.



