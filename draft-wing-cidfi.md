---
title: "QUIC CID Flow Indicator (CIDFI)"
abbrev: "QUIC CIDFI"
category: std


docname: draft-wing-cidfi-latest
submissiontype: IETF
v: 3
area: "Network"
workgroup: "Network Working Group"
keyword:
 - user experience
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
  QUIC: RFC9000


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

  IANA-SVCB:
    title: "DNS Service Bindings (SVCB)"
    target: https://www.iana.org/assignments/dns-svcb/dns-svcb.xhtml
    date: 2023-06-13

  IANA-STUN:
    title: STUN Attributes
    target: https://www.iana.org/assignments/stun-parameters/stun-parameters.xhtml
    date: 2023-03-20

--- abstract

Conveying metadata about network conditions and metadata about
individual packets can improve the user experience especially on
wireless networks which suffer bandwidth and delay variability.

This document describes how clients and servers can cooperate with
network elements so their QUIC streams can be augmented with
information about network conditions and packet importance.

--- middle

# Introduction

Senders rely on ramping up their transmission rate until they encounter
packet loss or see {{?ECN=RFC3168}} indicating they should level off or
slow down their transmission rate.  This feedback takes time and contributes
to poor user experience when the sender over- or under-shoots the actual
available bandwidth, especially if the sender changes fidelity of the
content (e.g., improves video quality which consumes more bandwidth which
then gets dropped by the network).

Due to network constraints a network element will need to drop or even
prioritize a packet ahead of other packets.  The decision of which packet
to drop or prioritize is improved if the network element knows the
importance of the packet.  Metadata carried in each packet can influence
that decision to improve the user experience.

This document defines CIDFI (pronounced "sid fye") which is a system
of several protocols that allow communicating about a {{QUIC}} connection
from the network to the server and the server to the network.  The
information exchanged allows the server to know about network conditions
and allows the server to signal packet importance.

{{fig-arch}} provides a sample network diagram of a CIDFI system showing two
bandwidth-constrained networks (or links) depicted by "B" and
CIDFI-aware devices immediately upstream of those links, and another
bandwidth-constrained link between a smartphone handset and its Radio
Access Newwork (RAN).  This diagram shows the same protocol and same mechanism
can operate with or without 5G, and can operate with different administrative
domains such as Wi-Fi, an ISP edge router, and a 5G RAN.

~~~~~ aasvg
                    |                     |          |
+------+   +------+ | +------+            |          |
|CIDFI-|   |CIDFI-| | |CIDFI-|            |          |
|aware |   |aware | | |aware |  +------+  |          |
|QUIC  +-B-+Wi-Fi +-B-+edge  +--+router+------+      |
|client|   |access| | |router|  +------+  |   |      | +--------+
+------+   |point | | +------+            |   |      | | CIDFI- |
           +------+ |                     | +-+----+ | | aware  |
                    |                     | |router+---+ QUIC   |
+---------+         | +------+            | +-+----+ | | server |
| CIDFI-  |         | |CIDFI-|            |   |      | +--------+
| aware   |         | |aware |  +------+  |   |      |
| QUIC    +-----B-----+RAN   +--+router+------+      |
| client  |         | |router|  +------+  |          |
|(handset)|         | +------+            |          |
+---------+         |                     |          |
                    |                     | transit  |  server
   user network     |    ISP network      | network  |  network
~~~~~
{: #fig-arch artwork-align="center" title="Network Diagram" :height=88}

The CIDFI-aware QUIC client establishes a TLS connection with the
CIDFI-aware network elements (Wi-Fi access point, edge router, and RAN
router in the above diagram).  Over this connection it receives
network performance information and it sends mapping of QUIC
Destination CIDs to packet importance.

The design creates new state in the CIDFI-aware network elements,
specifically for mapping from the QUIC Destination CID to the packet
importance and for the TLS-encrypted communication from the client.

In {{network-to-server}} this document describes network-to-server signaling
similar to the use-case described in {{Section 2 of ?I-D.joras-sadcdn}}, with metadata
relaying through the client.

In {{server-to-network}} this document describes server-to-network
metadata signaling similar to the use-cases described in
{{?I-D.reddy-tsvwg-explcit-signal}},
{{?I-D.kaippallimalil-tsvwg-media-hdr-wireless}}, and {{Section 3 of
I-D.joras-sadcdn}}.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

CID:
: QUIC Connection Identifier


# Design Goals

This section highlights the design goals of this specification.

Client Authorization:
: The client authorizes each CIDFI-aware network element to participate in CIDFI
for each QUIC flow.

Same Server:
: Communication about the network metadata arrives over the primary QUIC
connection which ensures it arrives at the same QUIC server even through load
balancers.

Privacy:
: The packet importance is only known by CIDFI-aware network
elements.  The network performance data is protected by TLS.

Integrity:
: The packet importance is mapped to QUIC Destination CIDs which are
integrity protected by QUIC itself and cannot be modified by on-path
network elements.  The network performance data is protected by TLS.

Internet Survival:
: The QUIC communications between the client and server are not changed
so CIDFI is expected to work wherever QUIC works.  The elements involved
are only the QUIC client, QUIC server, and the participating network
elements.



# Network Preparation: DNS SVCB Records

This document defines a new DNS Service Binding "cidfi-aware" in
{{iana-svcb}} and a new Special-Use Domain Name "cifi.arpa" in
{{iana-sudn}}.

The local network is configured to respond to DNS SVCB
{{!I-D.ietf-dnsop-svcb-https}} queries with ServiceMode ({{Section
2.4.3 of !I-D.ietf-dnsop-svcb-https}}) for _cidfi-aware.cidfi.arpa with
the DNS names of that network's and upstream network's CIDFI-aware
network elements.  If upstream networks also support CIDFI (e.g., the
ISP network) those SVCB records are aggregated into the local DNS
server's response by the local network's recursive DNS resolvers.  For
example, a query for _cidfi-aware.cidfi.arpa might return two answers
for the two CIDFI-aware elements on the local network, one belonging
to the local ISP (exmaple.net) and the other belonging to the local
Wi-Fi network (example.com),

~~~~~
  _cidfi-aware.cidfi.arpa. 7200 IN SVCB 0 service-cidfi.example.net.
  _cidfi-aware.cidfi.arpa. 7200 IN SVCB 0 wifi.example.com.
~~~~~

When multihoming, the multihome-capable CPE aggregates all upstream
networks' _cidfi-aware.cidfi.arpa responses into the response sent to
its locally-connected clients.


# Client Operation on Network Attach or Topology Change

On network attach or QUIC-detected topology change (see {{topology}}), the
client learns if the network supports CIDFI ({{discovery}}) and authorzes those
network elements ({{client-authorizes}}).

## Client Learns Local Network Supports CIDFI {#discovery}

> ** Note: For this section, a different approach using probe packets
  is described in {{probe}} but the authors are currently not pursuing
  that technique because of additional traffic and the additional
  delay to establish CIDFI on a new connection.  It is retained for
  discussion.

The client determines if the local network provides CIDFI service by
issuing a query to the local DNS server for
_cidfi-aware.cidfi.arpa. with the SVCB resource record type (64)
{{I-D.ietf-dnsop-svcb-https}}.  If this succeeds, processing skips to
{{client-authorizes}}.

If discovery failed it indicates the local network does not support
CIDFI and processing stops.


## Client Authorizes CIDFI Network Elements {#client-authorizes}

The SVCB response from the previous step in {{discovery}} will contain one or more
CIDFI-aware network elements.

The client authorizes each of the CIDFI-aware network elements using its local
policy.  This policy might prompt the user, allow certain names (e.g.,
allow example.net if the user's ISP is configured as example.net), or
similar.

After authorizing that subset of CIDFI-aware network elements, the
client makes a new HTTPS connection to each of those CIDFI-aware
network elements, validates their certificates, and obtains a CIDFI
nonce from each network element used in {{participation}} to prove
ownership of the UDP 4-tuple.


# Client Operation on Each Server Connection

When a QUIC client connects to a QUIC server:

  1. {{server-supports-cidfi}}, the client learns server supports CIDFI
     and obtains its mapping of transmitted destination CID to metadata.
  2. {{participation}}, the client contacts the network elements learned from {{discovery}} and requests
     their participation.
  3. {{initial-metadata-exchange}}, the client performs initial metadata exchange.
  4. {{ongoing-metadata-exchange}}, As network conditions change or the server's transmitted Destination CID changes,
     the client updates the network element or server.


These steps are described in more detail below.

> Note: the client is also a sender, and can also perform all these
functions in its direction.  This functionality will be expanded in
later versions of this document.  For example, a mobile device
connected to Wi-Fi with 5G backhaul might be running an interactive
audio/video application and want to indicate to its internal Wi-Fi
driver and to the 5G modem its mapping from QUIC Destination CID to
per-packet metadata and the application can benefit from receiving
network performance metrics.



## Client Learns Server Supports CIDFI {#server-supports-cidfi}

On initial connection to a QUIC server, the client includes a new QUIC
transport parameter CIDFI ({{iana-tp}}) which is remembered for 0-RTT.

If the server does not indicate CIDFI support, processing stops.

If the server indicates CIDFI support, then the server creates a
new Server-Initiated, Bidirectional stream which is dedicated to
CIDFI communication.  This stream number is communicated in the
CIDFI transport response during the QUIC handshake.  TODO: specify
how CIDFI stream number is communicated to client.

The QUIC client and QUIC server exchange CIDFI information over
this CIDFI-dedicated stream as described in {{initial-metadata-exchange}}.

At this point the client is aware of the server's (short) Destination
CID.

## Client Proves Ownership of UDP 4-Tuple

To ensure the client exchanges information about its its own UDP 4-tuple
with the CIDFI-network element, the client sends the CIDFI nonce it
obtained from {{client-authorizes}}.  The ability to transmit the nonce
on the same UDP 4-tuple as the QUIC connection indicates ownership of
that IP address and UDP port.  The nonce is encapsulated in a STUN
CIDFI-NONCE message ({{iana-stun}}) using the same UDP 4-tuple as the QUIC connection
to the server.  If there are multiple CIDFI-aware network elements,
the single STUN message contains a nonce from each of them.

The figure below shows the message flow around the steps to obtain
the nonce from the CIDFI-aware router, send the nonce in the same
UDP 4-tuple towards the QUIC server, and provide the mapping to the
CIDFI-aware network element.

~~~~~ aasvg
                                    CIDFI-aware            QUIC
   client                           edge router           server
     |                                    |                  |
     |  Enroll CIDFI router to partipate  |                  |
     +----------------------------------->|                  |
     |  Ok.  nonce=12345                  |                  |
     |<-----------------------------------+                  |
     |                                    |                  |
     :                                    :                  :
     |                                    |                  |
     |  QUIC Initial, transport parameter=CIDFI              |
     +------------------------------------------------------>|
     |  STUN nonce=12345                  |                  |
     +------------------------------------------------------>|
     |                                    |              discarded
     |                                    |                  |
     |                            "I saw my nonce!"          |
     |                                    |                  |
     |  QUIC Initial, transport parameter=CIDFI              |
     |<------------------------------------------------------+
     |  "Map DCID=xyz as high importance" |                  |
     +----------------------------------->|                  |
     |  Ok                                |                  |
     |<-----------------------------------+                  |
~~~~~


### STUN CIDFI-NONCE Attribute

The format of the STUN CIDFI-NONCE attribute is:

~~~~~
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                  Nonce-1 (128 bits)                           |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
:                                                               :
:                  Nonce-2 (128 bits)                           :
:                                                               :
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~
{: artwork-align="center" #fig-stun-cidfi-nonce title="Format of CIDFI-NONCE Attribute"}

The CIDFI nonce is 128 bits and multiple nonces can be transmitted in
a single STUN Binding Indication packet.  The STUN "Message Length"
field indicates how many Nonces are contained within the STUN message.


## Client Requests CIDFI Network Element's Participation {#participation}

Using its HTTPS channel with each of the CIDFI network elements it
previously authorized for CIDFI participation, the client signals the
mapping of the server's transmitted short Destination Connection ID
and its length to the CIDFI-aware network element.  As server support
of the QUIC CIDFI transport parameter is remembered for 0-RTT, the
client can immediately send the nonce.

The primary purpose of a second Connection ID is connection migration
({{Section 9 of QUIC}}).  With CIDFI, additional Connection IDs are
necessary to:
  * maintain CIDFI operation when topology remains the same.
  * use Destination Connection ID to indicate packet importance

To maintain CIDFI operation when topology remains the same, the
CIDFI client signals the CIDFI-aware network elements of that 'next'
Destination CID.  When QUIC detects a topology change, however, that
Destination CID MUST NOT be used by the peer, otherwise it links
the communication on the old topology to the new topology ({{Section 9.5 of QUIC}}).
Thus, an additional Connection ID is purposefully not communicated
from the CIDFI client to its CIDFI-aware network elements, so that
Connection ID can be immediately used by the peer during connection
migration when the topology changes.

Note the source IP address and source UDP port number are not signaled
by design.  This is because NATs ({{?NAPT=RFC3022}},
{{?NAT=RFC2663}}), multiple NATs on the path, IPv6/IPv4 translation,
similar technologies, and QUIC connection migration all complicate
accurate signaling of the source IP address and source UDP port
number.

If the CIDFI-aware router receives the HTTP map request but has not
yet seen the STUN nonce message, it rejects the mapping request. This
causes the client to re-send both the STUN nonce message and the
mapping request(s).

## Initial Metadata Exchange {#initial-metadata-exchange}

There are two types of metadata exchanged, described in the following sub-sections.

### Server to Network Elements {#server-to-network}

The server communicates to network elements via the client which then
communicates with the network element(s).  While this adds
communication delay, it allows the user at the client to authorize
the metadata communication about its own incoming (and outgoing) traffic.

The communication from the client to the server are using a CIDFI-dedicated
QUIC stream over the same QUIC connection as their primary communication.

~~~~~ aasvg
                  CIDFI-aware       CIDFI-aware
   client     Wi-Fi Access Point    edge router           server
     |                  |                 |                  |
     |  QUIC CIDFI stream: "Map DCID=xyz as high importance" |
     |<------------------------------------------------------+
     |  "Map DCID=xyz as                  |                  |
     |  high importance"|                 |                  |
     +----------------->|                 |                  |
     |  "Map DCID=xyz as high importance" |                  |
     +----------------------------------->|                  |
     |  Ok              |                 |                  |
     |<-----------------+                 |                  |
     |  Ok              |                 |                  |
     |<-----------------------------------+                  |
     |  QUIC CIDFI stream: Ok             |                  |
     +------------------------------------------------------>|
~~~~~

To each of the network elements authorized by the client, the client
sends the mappings of the server's transmitted Destination CIDs to
packet metadata (see {#packet-metadata}).

### Network Element to Server {#network-to-server}

The network element send network performance information to the server
which is intended to influence the sender'ss traffic rate (such as
improving or reducing fidelity of the audio or video).  In the figure below
the CIDFI-aware edge router informs the client of reduced bandwidth and the
client informs the server using CIDFI.

~~~~~ aasvg
                  CIDFI-aware       CIDFI-aware
   client     Wi-Fi Access Point    edge router     server
     |                  |                 |            |
     |  "bandwidth now 1Mbps"             |            |
     |<-----------------------------------+            |
     |  QUIC CIDFI stream: "bandwidth now 1Mbps"       |
     +------------------------------------------------>|
     |  QUIC CIDFI stream: Ok             |            |
     |<------------------------------------------------+
     |  Ok              |                 |            |
     +----------------------------------->|            |
~~~~~

The communication from the client to the server are using a CIDFI-dedicated
QUIC stream over the same QUIC connection as their primary communication.

The CIDFI-aware network element can update the client with whenever
the metadata about the connection changes significantly, but MUST NOT
update more frequently than once every second.

The metadata exchanged over this channel is described in {{metadata-exchanged}}.

## Ongoing Metadata Exchange {#ongoing-metadata-exchange}

For the duration of the primary QUIC connection between the QUIC client and
QUIC server, the client relays network element metadata changes to the server, and server's
transmitted QUIC Destination CID to the network elements.


# Interaction with Load Balancers {#load-balancers}

HTTP servers, including QUIC servers, are frequently behind load balancers.

With CIDFI, all the communication to the load-balanaced QUIC server are over the same UDP 4-tuple
as the primary QUIC connection but in a different QUIC stream.  This means
no changes are required to ECMP load balancers or to CID-aware load balancers
when using a CIDFI-aware back-end QUIC server.

Load balancers providing QUIC-to-TCP interworking is incompatible with
CIDFI because TCP lacks QUIC's stream identification.


# Topology Change {#topology}

When topology changes (such as switching to a backup WAN connection,
or such as switching from Wi-Fi to 5G), the QUIC server will consider
this a connection migration and will issue a PATH_CHALLENGE.

If the CIDFI-aware client is otherwise unaware of a topology change
and receives a PATH_CHALLENGE then the CIDFI-aware client SHOULD
re-discover its CIDFI network elements {{discovery}}.  If that
set of network elements differs from the previous set, the client
SHOULD continue with normal CIDFI processing.


# Details of Metadata Exchanged {#metadata-exchanged}

This section describes the metadata that can be exchanged from the
CIDFI-aware network element to the server (generally network
performance information) and from the server to the CIDFI-aware
network element.

> Note: we may want to use {{!CoAP=RFC7252}} for the client->network
communication and over the CIDFI-dedicated QUIC stream between the
QUIC client and QUIC server.



## Server to CIDFI-aware Network Element

Because there is no direct communications from the server to
the network element the communications are relayed through the client.

The communication from server to network element do not occur directly,
but rather through the client.

Two types of mapping metadata are described below: metadata parameters
and DSCP code points.

### Mapping Metadata Parameters to DCIDs {#mapping-parameters}

Several of the metadata parameters from {{Section 4.2 of
I-D.kaippallimalil-tsvwg-media-hdr-wireless}} can be mapped to QUIC
Destination CID:

Importance:
: As described in {{Section 4.2.5 of I-D.kaippallimalil-tsvwg-media-hdr-wireless}}.

Data burst:
: As described in {{Section 4.2.6 of I-D.kaippallimalil-tsvwg-media-hdr-wireless}}.

Delay budget:
: As described in {{Section 4.2.6 of
I-D.kaippallimalil-tsvwg-media-hdr-wireless}}, counting from when the packet
arrives at the CIDFI-aware network element to when it is transmitted.
The client knows its own jitter (playout) buffer length and the client
and server can calculate the one-way delay using timestamps.  With
that information, the client can adjust the server's signaled delay
budget with the client's own knowledge.  TODO: provide details.

Some other metadata parameters from {{Section 4.2 of
I-D.kaippallimalil-tsvwg-media-hdr-wireless}} cannot be successfully
mapped into QUIC Destination ID such as timestamp, MDU
Sequence, and Packet Counter.

Over the CIDFI-dedicated QUIC stream, the server sends mapping
information to the client when then propagates that information
to each of the CIDFI-aware network elements.

For discussion purposes, JSON is shown below to give a flavor
of the data exchanged.  The authors anticipate a more efficient
encoding such as {{!CBOR=RFC8949}} or pick-your-favorite encoding
and protocol:


~~~~~
  {"metadata-parameters":[{"quicversion":1,
    "dcidlength":3,
       "map":[
       {"import":17,"burst":83,"delaybudget":71,"dcids":[551,381]},
       {"import":3,"burst":888,"delaybudget":180,"dcids":[89,983]},
       {"import":7,"burst":37,"delaybudget":55,"dcids":[33]}]}]}
~~~~~


### Mapping DiffServ Code Point (DSCP) to DCIDs {#mapping-dscp}

A mapping from QUIC Destination CID to DiffServ code point
{{!RFC2474}} leverages existing DiffServ handling that may already
exist in the CIDFI network element.  If there are downstream network
elements configured with the same DiffServ code point the CIDFI
network element could mark the packet with that code point as well.

Signaling the DiffServ values for different QUIC Destination CID
increases the edge network's confidence the sender's DiffServ intent
is preserved into the edge network, even if the DSCP bits were
modified en route to the edge network (e.g., {{pathologies}}).

Over the CIDFI-dedicated QUIC stream, the server sends mapping
information to the client when then propagates that information
to each of the CIDFI-aware network elements.

For discussion purposes, JSON is shown below to give a flavor
of the data exchanged.  The authors anticipate a more efficient
encoding such as {{!CBOR=RFC8949}} or pick-your-favorite encoding
and protocol:

~~~~~
  {"dscp":[{"quicversion":1,
    "dcidlength":3,
    "map":[
      {"dscp":10,"dcids":[123,456]},
      {"dscp":46,"dcids":[998,183]}]}]}
~~~~~
{: #fig-dscp-json artwork-align="left" title="Example JSON for DSCP Mapping"}



## CIDFI-aware Network Element to Server

The CIDFI-aware client informs the network element of the client's
received Destination CIDs.  As bandwidth availability to that client
changes, the CIDFI-aware network element updates the client with new
metadata.

For discussion purposes, JSON is shown below to give a flavor of the
data sent from the CIDFI-aware network element to the client.  The
authors anticipate a more efficient encoding such as {{!CBOR=RFC8949}}
or pick-your-favorite encoding and protocol:

~~~~~
  {"dcid":123,
   "bandwidth":"1Mbps"}
~~~~~

The client then sends that information to the server in the CIDFI-dedicated
QUIC stream associated with that same Connection ID.

# Discussion Points

This section discusses the issues that benefit from wider discussion.

## Client versus Server Signaling CID-to-importance Mapping

Need to evaluate number of round trips (and other overhead) of client
signaling CID-to-importance mapping or server signaling CID-to-importance
mapping.


## Overhead of QUIC Packet Examination

If CID-to-importance metadata was signaled by the server as described
in {{server-to-network}}, the CIDFI-aware network elements have to
examine the UDP payload of each packet for a matching Destination CID
for the lifetime of the connection.  This is somewhat assuaged by
the STUN nonce transmitted which may well be an easier signal to
identify.


## Interaction with Wi-Fi Packet Aggregation

Per-packet metadata influences transmission of that packet but may
well conflict with some Wi-Fi optimizations (e.g., {{wifi-aggregation}})
and similar 5G optimizations.

This impact needs further study.


## Overhead of Mapping CID to packet metadata

Network Elements have to maintain a mapping between each UDP 4-tuple
and QUIC CID and its DSCP code point.  This also needs updating
whenever sender changes its CID.  This is awkward.

An alternative is a fixed mapping of QUIC CIDs to their meanings,
as proposed in {{?I-D.zmlk-quic-te}}.  However, this will ossify
the meaning of those QUIC CIDs.  It also requires all networks to
agree on the meaning of those QUIC CIDs.



## Improve CIDFI Initialization time

Find approaches to further reduce network communications to start CIDFI.


## Primary QUIC Channel CID Change {#primary-cid-change}:

Because the CIDFI network element, QUIC server, and QUIC client all
cooperate to share the primary QUIC connection's Destination CID,
when a new CIDFI network element is involved (e.g., due to client
attaching to a different network), a new Destination CID SHOULD
be used for the reasons discussed in {{Section 9.5 of QUIC}}}.

Do we want/need CIDFI network element to change the Destination CID of
its metadata communication in conjunction with the client changing its
Destination CID?



# Security Considerations

Because the sender's QUIC Destination Connection ID is mapped to
packet importance, and the DCID remains the same for many packets, an
attacker could determine which DCIDs are important by causing
interference on the bandwidth-constrained link (by creating other
legitimate traffic or creating radio interference) and observing which
DCIDs are transmitted versus which DCIDs are dropped.  This is a side-
effect of using fixed identifier (DCIDs) rather than encrypting
the packet importance.  This was a design trade-off to reduce the
CPU effort on CIDFI-aware network elements.  A mitigation is using
several DCIDs for every packet importance.

Communications are relayed through the client because only the
client and server knows the identity of the server and can validate
its certificate.

For an attacker to succeed with the nonce challenge against a victim's
UDP 4-tuple the attacker has to send a STUN CIDFI-NONCE packet using the
victim's source IP address.  Such spoofing of a victim's IP address is
generally prevented on local networks (e.g., {{?RFC2827}} for IPv4,
{{?RFC7513}} for IPv6).




# IANA Considerations

## New QUIC Transport Parameter {#iana-tp}

This document requests IANA to register the following new permanent QUIC transport parameter
in the "QUIC Transport Parameters" registry under the "QUIC" registry group available at {{IANA-QUIC}}:

|Value| Parameter Name| Reference|
|TBD1| CIDFI| This-Document|
{: title="New QUIC Transport Parameter"}


## New Well-known URI "cidfi-aware" {#iana-uri}

This document requests IANA to register the new well-known URI "cidfi" in the
"Well-Known URIs" registry available at {{IANA-WKU}}.

## New Special-use Domain Name {#iana-sudn}

Register new special-use domain name cidfi.arpa for DNS SVCB discovery.


## New DNS Service Binding (SVCB) {#iana-svcb}

This document requests IANA to register the new DNS SVCB "_cidfi-aware" in
the "DNS Service Bindings (SVCB)" registry available at {{IANA-SVCB}}.

## New STUN Attribute {#iana-stun}

This document requests IANA to register the new STUN attribute "CIDFI-NONCE"
in the "STUN Attributes" registry available at {{IANA-STUN}}.


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



