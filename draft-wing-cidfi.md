---
title: "CID Flow Indicator (CIDFI)"
abbrev: "CIDFI"
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
 - DTLS Connection Identifier
 - DTLS-SRTP
 - QUIC Connection Identifier
 - QUIC

venue:
  group: "ART Area Working Group"
  type: "Working Group"
  mail: "art@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/art/"
  github: "danwing/cidfi"
  latest: "https://danwing.github.io/cidfi/draft-wing-cidfi.html"

stand_alone: yes
pi: [toc, sortrefs, symrefs, strict, comments, docmapping]

author:
 -
    fullname: Dan Wing
    organization: Cloud Software Group, Inc.
    abbrev: Cloud Software Group
    country: United States of America
    email: ["danwing@gmail.com"]
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
  DTLS-CID: RFC9146

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
network elements so their QUIC and DTLS streams can be augmented with
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
of several protocols that allow communicating about a {{QUIC}}
connection or a DTLS connection {{DTLS-CID}} from the network to the
server and the server to the network.  The information exchanged
allows the server to know about network conditions and allows the
server to signal packet importance.

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
|client+-B-+Wi-Fi +-B-+edge  +--+router+------+      |
+------+   |access| | |router|  +------+  |   |      | +--------+
           |point | | +------+            |   |      | | CIDFI- |
           +------+ |                     | +-+----+ | | aware  |
                    |                     | |router+---+ QUIC or|
+---------+         | +------+            | +-+----+ | | DTLS   |
| CIDFI-  |         | |CIDFI-|            |   |      | | server |
| aware   |         | |aware |  +------+  |   |      | +--------+
| client  +-----B-----+RAN   +--+router+------+      |
|(handset)|         | |router|  +------+  |          |
+---------+         | +------+            |          |
                    |                     |          |
                    |                     | transit  |  server
   user network     |    ISP network      | network  |  network
~~~~~
{: #fig-arch artwork-align="center" title="Network Diagram" :height=88}

The CIDFI-aware client establishes a TLS connection with the
CIDFI-aware network elements (Wi-Fi access point, edge router, and RAN
router in the above diagram).  Over this connection it receives
network performance information and it sends mapping of (QUIC or DTLS)
Destination CIDs to packet importance.

The design creates new state in the CIDFI-aware network elements for
mapping from the QUIC Destination CID or DTLS Destination CID to the
packet importance, bandwidth information for that connection towards
the client, and for the TLS-encrypted communication with the client.

In {{network-to-server}} this document describes network-to-server signaling
similar to the use-case described in {{Section 2 of ?I-D.joras-sadcdn}}, with metadata
relaying through the client.

In {{server-to-network}} this document describes server-to-network
metadata signaling similar to the use-cases described in {{Section 3
of I-D.joras-sadcdn}}.  The server-to-network metadata signaling can
also benefit {{?I-D.ietf-avtcore-rtp-over-quic}} and {{DTLS-CID}}.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

CID:
: Connection Identifier used by {{QUIC}} or by {{DTLS-CID}}.

CNE:
: CIDFI-aware Network Element, a network element that
supports this CIDFI specification.  This is usually a router.

# Design Goals

This section highlights the design goals of this specification.

Client Authorization:
: The client authorizes each CIDFI-aware network element (CNE) to participate in CIDFI
for each QUIC flow or each DTLS flow.

Same Server:
: Communication about the network metadata arrives over the primary QUIC or
DTLS connection which ensures it arrives at the same server even through
local network translators (NAT) or server-side load balancers.

Privacy:
: The packet importance is only known by CIDFI-aware network
elements (CNE).  The network performance data is protected by TLS.

Integrity:
: The packet importance is mapped to Destination CIDs which are
integrity protected by QUIC or DTLS itself and cannot be modified by on-path
network elements.  The communication between client, server, and
network element are protected by TLS.

Internet Survival:
: The QUIC communications and DTLS communications between the client
and server are not changed so CIDFI is expected to work wherever QUIC
(or DTLS) work.  The elements involved are only the QUIC (or DTLS)
client and server and with the participating CIDFI network elements.



# Network Preparation: DNS SVCB Records

This document defines a new DNS Service Binding "cidfi-aware" in
{{iana-svcb}} and a new Special-Use Domain Name "cifi.arpa" in
{{iana-sudn}}.

The local network is configured to respond to DNS SVCB
{{!I-D.ietf-dnsop-svcb-https}} queries with ServiceMode ({{Section
2.4.3 of !I-D.ietf-dnsop-svcb-https}}) for _cidfi-aware.cidfi.arpa with
the DNS names of that network's and upstream network's CIDFI-aware
network elements (CNEs).  If upstream networks also support CIDFI (e.g., the
ISP network) those SVCB records are aggregated into the local DNS
server's response by the local network's recursive DNS resolvers.  For
example, a query for _cidfi-aware.cidfi.arpa might return two answers
for the two CNE on the local network, one belonging
to the local ISP (exmaple.net) and the other belonging to the local
Wi-Fi network (example.com),

~~~~~
_cidfi-aware.cidfi.arpa. 7200 IN SVCB 0 service-cidfi.example.net. (
    alpn=h3 cidfipathauth=/path-auth-query {?cidfi}
    cidfimetadata=/cidfi-metadata
    )
_cidfi-aware.cidfi.arpa. 7200 IN SVCB 0 wifi.example.com. (
    alpn=h3 cidfipathauth=/path-auth-query {?cidfi}
    cidfimetadata=/cidfi-metadata
    )
~~~~~

When multihoming, the multihome-capable CPE aggregates all upstream
networks' _cidfi-aware.cidfi.arpa responses into the response sent to
its locally-connected clients.


# Client Operation on Network Attach or Topology Change {#attach}

On initial network attach topology change (see {{topology}}),
the client learns if the network supports CIDFI ({{discovery}}) and
authorizes those network elements ({{client-authorizes}}).

## Client Learns Local Network Supports CIDFI {#discovery}

The client determines if the local network provides CIDFI service by
issuing a query to the local DNS server for
_cidfi-aware.cidfi.arpa. with the SVCB resource record type (64)
{{I-D.ietf-dnsop-svcb-https}}.  If this succeeds, processing skips to
{{client-authorizes}}.

If discovery failed it indicates the local network does not support
CIDFI and processing stops.


## Client Authorizes CIDFI Network Elements {#client-authorizes}

The SVCB response from the previous step in {{discovery}} will contain one or more
CNE.

The client authorizes each of the CNE using
its local policy.  This policy is implementation specific.  An example
implementation might have the user authorize their ISP's CIDFI server
(e.g., allow cidfi.example.net if the user's ISP is configured as
example.net).  Similarly, if none of the CNE are recognized the client
might silently avoid using CIDFI on that network.

After authorizing that subset of CNE, the
client makes a new HTTPS connection to each of those CNE
and performs PKIX validation of their certificates.
The client MAY have to authenticate itself to the CIDFI network
element.

The client then obtains the CIDFI nonce and CIDFI HMAC
secret from each network element used later in {{ownership}} to prove
the client owns its UDP 4-tuple.

For discussion purposes, JSON is shown below to give a flavor of the
data the client retrieves from the CIDFI network element.  The authors
anticipate using a more efficient encoding such as {{!CBOR=RFC8949}}.


~~~~~
  {"cidfi-path-authentication":[
    {"nonce":"ddqwohxGZysgy0BySNh7sNHV5IH9RbE7rqXmg9wb9Npo",
     "hmac-secret":"jLNsCvuU59mt3F4/ePD9jbZ932TfsLSOP2Nx3XnUqc8v"}]}
~~~~~


# Client Operation on Each Connection to a QUIC Server

When a QUIC client or {DTLS-CID} client connects to a QUIC or {DTLS-CID} server, the client:

  1. learns the server supports CIDFI
     and obtains its mapping of transmitted destinations CID to metadata.
  2. proves ownership of its UDP 4-tuple to
     the on-path network elements.
  3. performs initial metadata exchange
     with the CIDFI network element and server, and server and network element.
  4. continually updates the server and the CIDFI network element whenever
     new information is received from the other party.

These steps are described in more detail below.

> Note: the client is also a sender, and can also perform all these
functions in its direction.  This functionality will be expanded in
later versions of this document.  For example, a mobile device
connected to Wi-Fi with 5G backhaul might be running an interactive
audio/video application and want to indicate to its internal Wi-Fi
driver and to the 5G modem its mapping from its transmitted QUIC
Destination CID to per-packet metadata and the application can benefit
from receiving network performance metrics.



## Client Learns Server Supports CIDFI {#server-supports-cidfi}

On initial connection to a QUIC server, the client includes a new QUIC
transport parameter CIDFI ({{iana-tp}}) which is remembered for 0-RTT.

If the server does not indicate CIDFI support, CIDFI processing stops.

If the server indicates CIDFI support, then the server creates a
new Server-Initiated, Bidirectional QUIC stream which is dedicated to
CIDFI communication.  This stream number is communicated in the
CIDFI transport response during the QUIC handshake.  TODO: specify
how CIDFI stream number is communicated to client.

The QUIC client and QUIC server exchange CIDFI information over
this CIDFI-dedicated stream as described in {{initial-metadata-exchange}}.


## Client Proves Ownership of its UDP 4-Tuple {#ownership}

To ensure the client messages to the CNE
pertain only to the client's own UDP 4-tuple, the client sends the
CIDFI nonce protected by the HMAC secret it obtained from
{{client-authorizes}} over the QUIC UDP 4-tuple it is using with the
QUIC server.  The ability to transmit that packet on the same UDP
4-tuple as the QUIC connection indicates ownership of that IP address
and UDP port.  The nonce and HMAC are sent in a {{!STUN=RFC8489}} indication (STUN
class of 0b01) containing one or more CIDFI-NONCE attributes
({{iana-stun}}).  If there are multiple CNE
the single STUN indication contains a CIDFI-NONCE attribute from each of
them.  This message is discarded by the QUIC server.


The figure below shows a summarized message flow obtaining
the nonce and HMAC secret from the CNE then later
sending the nonce and HMAC in the same UDP 4-tuple towards the QUIC server:

~~~~~ aasvg
 QUIC                            CIDFI-aware            QUIC
client                           edge router           server
  |                                    |                  |
  |  HTTPS: Enroll CIDFI router to partipate              |
  +----------------------------------->|                  |
  |  HTTPS: Ok.  nonce=12345           |                  |
  |<-----------------------------------+                  |
  |                                    |                  |
  :                                    :                  :
  |                                    |                  |
  |  QUIC Initial, transport parameter=CIDFI              |
  +------------------------------------------------------>|
  |  STUN Indication, nonce=12345, hmac=e8FEc             |
  +------------------------------------------------------>|
  |                                    |              discarded
  |                                    |                  |
  |                    "I saw my nonce, HMAC is valid"    |
  |                                    |                  |
  |  HTTPS: "Map DCID=xyz as high importance"             |
  +----------------------------------->|                  |
  |  QUIC Initial, transport parameter=CIDFI              |
  |<------------------------------------------------------+
  |  HTTPS: Ok                         |                  |
  |<-----------------------------------+                  |
~~~~~
{: artwork-align="center"}

Because multiple QUIC clients will use the same incoming Destination
CID on their own UDP 4-tuple, the STUN Indication message also allows
the CIDFI network element to distinguish which UDP 4-tuple belongs to
each CIDFI client.

To reduce CIDFI set-up time the client STUN Indication MAY be sent at
the same time as the QUIC Initial packet, which is encouraged
if the client remembers the server supports CIDFI (0-RTT).

To prevent replay attacks, the Nonce is usable only for authenticating
one UDP 4-tuple.  When the connection is migrated ({{Section 9 of
QUIC}}) the CIDFI network element won't apply any CIDFI behavior to
that newly-migrated connection.  The client will have to restart
CIDFI procedures at the beginning ({{attach}}).



### STUN CIDFI-NONCE Attribute

The format of the STUN CIDFI-NONCE attribute is:

~~~~~ aasvg
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                       Nonce (128 bits)                        |
|                                                               |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
|                                                               |
|                                                               |
|                     HMAC-output (256 bits)                    |
|                                                               |
|                                                               |
|                                                               |
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
~~~~~
{: artwork-align="center" #fig-stun-cidfi-nonce title="Format of CIDFI-NONCE Attribute"}

The nonce is 128 bits obtained from the CIDFI network element.  The
HMAC-output field is computed per {{?RFC5869}} using the CIDFI network
element-provided HMAC secret, and the CIDFI network element-provided
Nonce concatenated with the fixed string "cidfi" (without quotes),
shown below with "|" denoting concatenation.

~~~~~
  HMAC-output = HMAC-SHA256( hmac-secret, nonce | "cidfi" )
~~~~~


## Initial Metadata Exchange {#initial-metadata-exchange}

Using its HTTPS channel with each of the CIDFI network elements it
previously authorized for CIDFI participation, the client signals the
mapping of the server's transmitted short Destination Connection ID
and its length to the CNE.  As server support
of the QUIC CIDFI transport parameter is remembered for 0-RTT, the
client can immediately send the nonce.

The primary purpose of a second Connection ID is connection migration
({{Section 9 of QUIC}}).  With CIDFI, additional Connection IDs are
necessary to:
  * maintain CIDFI operation when topology remains the same.
  * use Destination Connection ID to indicate packet importance

To maintain CIDFI operation when topology remains the same, the
CIDFI client signals the CNEs of that 'next'
Destination CID.  When QUIC detects a topology change, however, that
Destination CID MUST NOT be used by the peer, otherwise it links
the communication on the old topology to the new topology ({{Section 9.5 of QUIC}}).
Thus, an additional Connection ID is purposefully not communicated
from the CIDFI client to its CNEs, so that
Connection ID can be immediately used by the peer during connection
migration when the topology changes.

Note the source IP address and source UDP port number are not signaled
by design.  This is because NATs ({{?NAPT=RFC3022}},
{{?NAT=RFC2663}}), multiple NATs on the path, IPv6/IPv4 translation,
similar technologies, and QUIC connection migration all complicate
accurate signaling of the source IP address and source UDP port
number.

If the CNE receives the HTTPS map request but has not
yet seen the STUN nonce message it rejects the mapping request with a
403 and provides a new nonce.  The new nonce avoids the problem of an
attacker seeing the previous nonce and using that nonce on its own UDP
4-tuple.  The client then sends a new STUN message with that new nonce
value and send a new HTTPS mapping request(s).  This interaction is
highlighted in the simplified message flow, below.

~~~~~ aasvg
                                 CIDFI-aware            QUIC
client                           edge router           server
  |                                    |                  |
  |  HTTPS: Enroll CIDFI router to partipate              |
  +----------------------------------->|                  |
  |  HTTPS: Ok.  nonce=12345           |                  |
  |<-----------------------------------+                  |
  |                                    |                  |
  :                                    :                  :
  |                                    |                  |
  |  QUIC Initial, transport parameter=CIDFI              |
  +------------------------------------------------------>|
  |  STUN Indication, nonce=12345, HMAC=8f93e             |
  +--------------------> X (lost)      |                  |
  |                                    |                  |
  |  HTTPS: "Map DCID=xyz as high importance"             |
  +----------------------------------->|                  |
  |  HTTPS: 403, new Nonce=5678        |                  |
  |<-----------------------------------|                  |
  |  STUN Indicate, nonce=5678, HMAC=aaf3c                |
  +------------------------------------------------------>|
  |                                    |              discarded
  |                                    |                  |
  |                    "I saw my nonce, HMAC is valid"    |
  |                                    |                  |
  |  HTTPS: "Map DCID=xyz as high importance"             |
  +----------------------------------->|                  |
  |  Ok                                |                  |
  |<-----------------------------------+                  |
~~~~~
{: artwork-align="center" title="Client re-transmtting lost nonce"}


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
{: artwork-align="center"}

To each of the network elements authorized by the client, the client
sends the mappings of the server's transmitted Destination CIDs to
packet metadata (see {#packet-metadata}).

### Network Element to Server {#network-to-server}

The network element send network performance information to the server
which is intended to influence the sender'ss traffic rate (such as
improving or reducing fidelity of the audio or video).  In the figure below
the CNE informs the client of reduced bandwidth and the
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
{: artwork-align="center"}

The communication from the client to the server are using a CIDFI-dedicated
QUIC stream over the same QUIC connection as their primary communication.

The CNE can update the client with whenever
the metadata about the connection changes significantly, but MUST NOT
update more frequently than once every second.

The metadata exchanged over this channel is described in {{metadata-exchanged}}.

## Ongoing Metadata Exchange {#ongoing-metadata-exchange}

For the duration of the primary QUIC connection between the QUIC client and
QUIC server, the client relays network element metadata changes to the server, and server's
transmitted QUIC Destination CID to the network elements.


# Interaction with Load Balancers {#load-balancers}

HTTPS servers, including QUIC servers, are frequently behind load balancers.

With CIDFI, all the communication to the load-balanaced QUIC server are over the same UDP 4-tuple
as the primary QUIC connection but in a different QUIC stream.  This means
no changes are required to ECMP load balancers or to CID-aware load balancers
when using a CIDFI-aware back-end QUIC server.

Load balancers providing QUIC-to-TCP interworking is incompatible with
CIDFI because TCP lacks QUIC's stream identification.


# Topology Change {#topology}

When topology changes the client will transmit from a new IP address
-- such as switching to a backup WAN connection, or such as switching
from Wi-Fi to 5G.  If using QUIC, QUIC server will consider this a
connection migration and will issue a PATH_CHALLENGE.  If the client
is aware of the topology change (such as attaching to a different
network), the client would also change its QUIC Destination CID ({{Section
9 of QUIC}}).

If the QUIC CIDFI-aware client is otherwise unaware of a topology change
and receives a QUIC PATH_CHALLENGE then the CIDFI-aware client SHOULD
re-discover its CIDFI network elements {{discovery}}.  If that
set of network elements differs from the previous set, the client
SHOULD continue with normal CIDFI processing.

> todo: include discussion of {{DTLS-CID}} client and discussion
of its ICE interaction, if any?


# Details of Metadata Exchanged {#metadata-exchanged}

This section describes the metadata that can be exchanged from the
CNE to the server (generally network
performance information) and from the server to the CNE.


## Server to CIDFI-aware Network Element

Because there is no direct communications from the server to
the network element the communications are relayed through the client.

The communication from server to network element do not occur directly,
but rather through the client.

Two types of mapping metadata are described below: metadata parameters
and DSCP code points.

### Mapping Metadata Parameters to DCIDs {#mapping-parameters}

Several of metadata parameters can be mapped to Destination CID:

Importance:
: Low/Medium/High importance, relative to other CIDs within this
same UDP 4-tuple.

Delay budget:
: Time in milliseconds until this packet is worthless to the receiver.
This is counted from when the packet arrives at the CNE
to when it is transmitted; other delays may occur
before or after that event occurs.  The receiver knows its own jitter
(playout) buffer length and the client and server can calculate the
one-way delay using timestamps.  With that information, the client can
adjust the server's signaled delay budget with the client's own
knowledge.  TODO: provide enough details to create interoperable
implementations.

Over the CIDFI-dedicated QUIC stream, the server sends mapping
information to the client when then propagates that information
to each of the CNEs.

For discussion purposes, JSON is shown below to give a flavor
of the data exchanged.  The authors anticipate a more efficient
encoding such as {{!CBOR=RFC8949}}.

~~~~~
  {"metadata-parameters":[{"quicversion":1,
    "dcidlength":3,
       "map":[
       {"import":17,"burst":83,"delaybudget":71,"dcids":[551,381]},
       {"import":3,"burst":888,"delaybudget":180,"dcids":[89,983]},
       {"import":7,"burst":37,"delaybudget":55,"dcids":[33]}]}]}
~~~~~


### Mapping DiffServ Code Point (DSCP) to DCIDs {#mapping-dscp}

A mapping from Destination CID to DiffServ code point
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
to each of the CNEs.

For discussion purposes, JSON is shown below to give a flavor
of the data exchanged.  The authors anticipate using a more efficient
encoding such as {{!CBOR=RFC8949}}.

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
changes, the CNE updates the client with new
metadata.

For discussion purposes, JSON is shown below to give a flavor of the
data sent from the CNE to the client.  The
authors anticipate using a more efficient encoding such as {{!CBOR=RFC8949}}.


~~~~~
  {"dcid":123,
   "bandwidth":"1Mbps"}
~~~~~

The client then sends that information to the server in the CIDFI-dedicated
QUIC stream associated with that same Connection ID.

# Discussion Points

This section discusses known issues that would benefit from wider discussion.



## Client versus Server Signaling CID-to-importance Mapping

Need to evaluate number of round trips (and other overhead) of client
signaling CID-to-importance mapping or server signaling CID-to-importance
mapping.


## Overhead of QUIC DCID Packet Examination

If CID-to-importance metadata was signaled by the server as described
in {{server-to-network}}, the CNE have to
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

We need clear way to signal which DCIDs can be used for 'this'
network attach and which DCIDs are for a migrated connection.  Probably
belongs in the QUIC transport parameter signaling?



# Security Considerations

Because the sender's QUIC Destination Connection ID is mapped to
packet importance, and the DCID remains the same for many packets, an
attacker could determine which DCIDs are important by causing
interference on the bandwidth-constrained link (by creating other
legitimate traffic or creating radio interference) and observing which
DCIDs are transmitted versus which DCIDs are dropped.  This is a side-
effect of using fixed identifier (DCIDs) rather than encrypting
the packet importance.  This was a design trade-off to reduce the
CPU effort on the CNEs.  A mitigation is using
several DCIDs for every packet importance.

Communications are relayed through the client because only the
client and server knows the identity of the server and can validate
its certificate.  The protocol in this specification does not disclose
the server's identity to the CIDFI network element.


For an attacker to succeed with the nonce challenge against a victim's
UDP 4-tuple the attacker has to send a STUN CIDFI-NONCE packet using
the victim's source IP address and a valid HMAC.  A valid HMAC can
only be obtained by the attacker making its own connection to the
CIDFI server and spoofing the source IP address and UDP port of the
victim.  Such spoofing of a victim's IP address is prevented by the
network using network ingress filtering ({{?RFC2827}}, {{?RFC7513}},
{{?RFC6105}}, and/or {{?RFC6620}}).  In the event network ingress
filtering is not configured or configured improperly, the CIDFI
network element can detect an attack if the client implements CIDFI.
The CIDFI network element receive two HTTPS connections describing the
same DCID (one connection from the attacker, another from the victim).
The CIDFI network element will issue unique Nonces and HMACs to both
the attacker and victim, and the attacker and victim will both send
the STUN indication on that same UDP 4-tuple.  This should never
normally occur and should generate an alarm on the CIDFI network
element.  In this situation, it is recommended both attack and victim
be denied CIDFI access.




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




# Acknowledgments
{:numbered="false"}

Thanks to Dave Täht, Magnus Westerlund, Christian Huitema, Gorry Fairhurst,
and Tom Herbert for hallway discussions and feedback at TSVWG that encouraged
the authors to consider the approach described in this document.


