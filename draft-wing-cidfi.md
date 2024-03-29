---
title: "Framework for CID Flow Indicator (CIDFI)"
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
  group: "TSV Area"
  type: "Working Group"
  mail: "tsvwg@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/tsvwg/"
  github: "danwing/cidfi"
  latest: "https://danwing.github.io/cidfi/draft-wing-cidfi.html"

stand_alone: yes
pi: [toc, sortrefs, symrefs, strict, comments, docmapping]

author:
 -
    fullname: Dan Wing
    organization: Cloud Software Group Holdings, Inc.
    abbrev: Cloud Software Group
    country: United States of America
    uri: https://www.cloud.com
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

  IANA-PVD:
    title: Provisioning Domains (PvDs)
    target: https://www.iana.org/assignments/pvds/pvds.xhtml#additional-information-pvd-keys
    date: 2020-08-13


--- abstract

Host-to-network signaling and network-to-host signaling can improve
the user experience to adapt to network's constraints and share expected application needs, and thus to provide
differentiated service to a flow and to packets within a flow. The differentiated service may be provided at the network (e.g., packet prioritization), the server (e.g., adaptive transmission), or both.

This document describes how clients can communicate with their nearby
network elements so they can learn network constraints.  Optionally,
with QUIC server support their incoming QUIC packets can be mapped to
metadata about their contents so packet importance can influence both
intentional and reactive management policies.  The framework handles
both directions of a flow.

--- middle

# Introduction

Senders rely on ramping up their transmission rate until they encounter
packet loss or see {{?ECN=RFC3168}} indicating they should level off or
slow down their transmission rate.  This feedback takes time and contributes
to poor user experience when the sender over- or under-shoots the actual
available bandwidth, especially if the sender changes fidelity of the
content (e.g., improves video quality which consumes more bandwidth which
then gets dropped by the network).  This is also called an 'intentional
management policy'.

Due to network constraints a network element will need to drop or even
prioritize a packet ahead of other packets within the same UDP 4-tuple. The decision of which packet
to drop or prioritize is improved if the network element knows the
importance of the packet.  By mapping packet metadata to a network-visible
field in each packet, the network element is better informed and better able
to improve the user experience.

There are also exceptional cases (crisis) where "normal" network
resources cannot be used at maximum and, thus, a network would seek to
reduce or offload some of the traffic during these events -- often
called 'reactive traffic policy'. Network-to-host signals are
useful to put in place adequate traffic distribution policies (e.g.,
prefer the use of alternate paths, offload a network).

{{design-approaches}} depicts examples of approaches to establish channels to convey
and share metadata between hosts, networks, and servers. This document adheres to
the client-centric metadata sharing approach because it preserves privacy and also
takes advantage of clients having a full view on their available network attachments.
Metadata exchanges can occur in one single direction or both directions of a flows.


~~~~~ aasvg
(1)  Proxied Connection
                       .--------------.                   +------+
                      |                |                +-+----+ |
+------+              |   Network(s)   |              +-+----+ +-+
|Client+--------------)----------------(--------------+Server+-+
+---+--+              |                |              +---+--+
    |                  '-------+------'                   |
    |                          |                          |
    +<===User Data+Metadata===>+<===User Data+Metadata===>+
    |   Secure Connection 1    |   Secure Connection 2    |
    |                          |                          |

(2)  Out-of-band Metadata Sharing
                        .--------------.                  +------+
                       |                |               +-+----+ |
+------+               |   Network(s)   |             +-+----+ +-+
|Client+---------------)----------------(-------------+Server+-+
+---+--+               |                |             +---+--+
    |                   '-------+------'                  |
    |                           |                         |
    +<-----End-to-End Secure Connection + User Data------>+<---.
    |                           |                         | GLUE|
    |                           |                         | CXs |
    +<-- Metadata (Optional) -->+<----- Metadata -------->+<---'
    |    Secure Connection 1    |    Secure Connection 2  |
    |                           |                         |

(3)  Client-centric Metadata Sharing
                          .--------------.                  +------+
                         |                |               +-+----+ |
+------+                 |   Network(s)   |             +-+----+ +-+
|Client+-----------------)----------------(-------------+Server+-+
+---+--+                 |                |             +---+--+
    |                     '-------+------'                  |
    |                             |                         |
    +<--------- Metadata -------->+                         |
    |        Secure Connection    |                         |
    |                             |                         |
    +<== End-to-End Secure Connection User Data+Metadata ==>+
    |                             |                         |
~~~~~
{: #design-approaches artwork-align="center" title="Candidate Design Approaches"}

The document is a generic framework that would function in any network deployment. This framework can be leveraged by any transport protocol (see {{extending}}). To illustrate the framework's applicability this document focuses on QUIC transport.

The design supports multiple CIDFI and QUIC implementations on one
host (i.e., by several applications), cellular devices providing IP
connectivity to other devices (see {{Section 3 of ?RFC7649}}), multiple
CIDFI-aware network elements (e.g., Wi-Fi and an ISP network), DOCSIS and
5G networks, and hosts
behind one or more IPv4 NATs or other IP translation technologies.  A comprehensive list of such translation technologies is provided in {{Section 2.2 of ?RFC8512}}.

# Overview

This document defines CIDFI (pronounced "sid fye") which is a system
of several protocols that allow communicating about a {{QUIC}}
connection from the network to the server and the server to the network.
The information exchanged allows the server to know about network conditions
and allows the server to signal packet importance. The following main steps
are involved in CIDFI; some of them are optional:

* CIDFI-awareness discovery between a host and a network.
* Establishment of a secure association with all or a subset of CIDFI-aware
  networks.
* Negotiation of CIDFI support with remote servers.
* CIDFI-aware networks sharing of changes of network conditions.
* CIDFI-aware clients sharing of metadata with CIDFI-aware networks as hints
  to help processing flows.
* CIDFI-aware clients sharing of metadata with CIDFI-aware server to adapt
  to local network conditions.

**CIDFI does not require that all these steps are enabled**. Incremental
deployments may be envisaged (e.g., network and client support, network, client,
and server support). Differentiated service can
be provided to a flow, packets within a flow, or a combination thereof as a function
of the CIDFI support by various involved entities. For example, a CIDFI-aware network
might share signals with clients that would then trigger locally connection migration or relay
the information to the server (if it is CIDFI-aware) to adjust its sending behavior
by avoiding aggressive use of local resources or using alternate paths. {{metadata-exchanged}}
further elaborates on the differentiated service that can be provided by enabling CIDFI.

{{fig-arch}} provides a sample network diagram of a CIDFI system showing two
bandwidth-constrained networks (or links) depicted by "B" and
CIDFI-aware devices immediately upstream of those links, and another
bandwidth-constrained link between a smartphone handset and its Radio
Access Network (RAN).  This diagram shows the same protocol and same mechanism
can operate with or without 5G, and can operate with different administrative
domains such as Wi-Fi, an ISP edge router, and a 5G RAN. Readers may refer to Appendix C
of {{?I-D.ietf-teas-5g-ns-ip-mpls}} for an overview of key 5G building blocks.

For the sake of illustration, {{fig-arch}} simplifies the representation
of the various involved network segments. It also assumes that multiple
server instances are enabled in the server network but the document
does not make any assumption about the internal structure of the service
nor how a flow is processed by or steered to a service instance. However,
CIDFI includes provisions to ensure that the service instance that is
selected to service a client request is the same instance that will
receive CIDFI metadata for that client.

~~~~~ aasvg
                    |                     |          |
+------+   +------+ | +------+            |          |
|CIDFI-|   |CIDFI-| | |CIDFI-|            |          |
|aware |   |aware | | |aware |  +------+  |          |     +--------+
|client+-B-+Wi-Fi +-B-+edge  +--+router+------+      |   +-+------+ |
+------+   |access| | |router|  +------+  |   |      | +-+------+ | |
           |point | | +------+            |   |      | | CIDFI- | | |
           +------+ |                     | +-+----+ | | aware  | +-+
                    |                     | |router+---+ QUIC   +-+
+---------+         | +------+            | +-+----+ | | server |
| CIDFI-  |         | |CIDFI-|            |   |      | +--------+
| aware   |         | |aware |  +------+  |   |      |
| client  +-----B-----+RAN   +--+router+------+      |
|(handset)|         | |router|  +------+  |          |
+----+----+         | +------+            |          |
     |              |                     |          |
+----+----+         |                     |          |
| CIDFI-  |         |                     |          |
| aware   |         |                     |          |
|  app    |         |                     |          |
+---------+         |                     |          |
                    |                     | Transit  |  Server
   User Network     |    ISP Network      | Network  |  Network
~~~~~
{: #fig-arch artwork-align="center" title="Network Diagram" :height=88}

The CIDFI-aware client establishes a TLS connection with the
CIDFI-aware network elements (Wi-Fi access point, edge router, and RAN
router in the above diagram).  Over this connection it receives
network performance information (n2h) and it sends mapping of QUIC
Destination CIDs to packet importance (h2n).

The design creates new state in the CIDFI-aware network elements for
mapping from Destination CID to the packet metadata and maintaining
triggers to update the client if the network characteristics change,
and to maintain a TLS channel with the client.

{{network-to-host}} describes network-to-host signaling
similar to the use case described in {{Section 2 of ?I-D.joras-sadcdn}}, with metadata
relaying through the client.

{{host-to-network}} describes host-to-network
metadata signaling similar to the use cases described in {{Section 3
of ?I-D.joras-sadcdn}}.  The host-to-network metadata signaling can
also benefit {{?I-D.ietf-avtcore-rtp-over-quic}}.


CIDFI brings benefits to QUIC as that protocol is of primary interest.
QUIC is quickly replacing HTTPS-over-TCP on many websites and content
delivery networks because of its advantages to both end users and
servers.  CIDFI can bring value to a system comprised solely of a
CIDFI-aware client and the CIDFI-aware network elements.  By adding a
CIDFI-aware server that supports QUIC unreliable datagrams
{{!RFC9221}} and API integration (see {{api-integration}}), each
packet can receive differentiated service from the network.  This is
especially useful during user transitions from a high quality wireless
reception to lower quality reception (e.g., entering a building).
Additionally, CIDFI can be extended to other protocols as discussed in
{{extending}}.

## Operation with Streaming Video

Incremental deployment:
: Streaming video only needs to be transmitted slightly faster than the
video playout rate.  Sending the video significantly faster can waste
bandwidth, most notably if the user abandons the video early.  Worse, as discussed in {{Section 3.10 of ?RFC8517}}, a fast download of a video that won't be viewed completely by the subscriber may lead to quick exhaustion of the user data quota. CIDFI
helps this use-case with its network-to-host signaling which informs
the client of available bandwidth allowing the client to choose
a compatible video stream.  This functionality does not need a CIDFI-
aware server.

Full system deployment:
: With reliable transport such as TCP, the only purpose of
video key frames is the user scrolling forward/backward.  When
video streaming uses unreliable transport ({{?RFC9221}})
it is beneficial to differentiate keyframes from predictive
frames on the network especially when the network performs
reactive policy management.  When the server also supports CIDFI,
key frames can be differentiated which improves user experience
during linear playout.

## Operation with Interactive Audio/Video/Screen sharing

Incremental deployment:
: With interactive sessions CIDFI can help determine the bandwidth
available for the flow so the video (and screen sharing) quality and
size can be constrained to the available bandwidth.  This benefit
can be deployed locally with a CIDFI-aware client and CIDFI-aware
network.

Full system deployment:
: When the remote peer also supports CIDFI, the remote peer can
differentiate packets containing audio, video, or screen sharing.  In
certain use-cases audio is the most important whereas in other
use-cases screen sharing is most important.  With CIDFI, the relative
importance of each packet can be differentiated as that relative
importance changes during a session.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

The document makes use of the following terms:

CID:
: Connection Identifier used by {{QUIC}}.

CNE:
: CIDFI-aware Network Element, a network element that
supports this CIDFI specification.  This is typically a router.

Differentiated service:
: Refers to a differentiated processing that can be provided to a flow (or specific packets within a flow) by a network, client, or server.
: Examples of differentiated service are: prioritization, adaptive transmission, or traffic steering.

## Notations

For discussion purposes, JSON is used in the examples to give a flavor of the
data that the client retrieves from a CNE.  The authors
anticipate using a more efficient encoding such as {{!CBOR=RFC8949}}.

# Design Goals

This section highlights the design goals of this specification.

Client Authorization:
: The client authorizes each CIDFI-aware network element (CNE) to participate in CIDFI
for each QUIC flow.

Same Server Instance:
: When the server also participates in CIDFI, the same QUIC connection is used for CIDFI
communication with that server,
which ensures it arrives at the same server instance even in the presence of
network translators (NAT) or server-side ECMP load balancers or server-side CID-aware
load balancers {{?I-D.ietf-quic-load-balancers}}.

Privacy:
: The host-to-network signaling of the mapping from packet metadata to CID is only sent to CIDFI-aware network
elements (CNEs) and is protected by TLS.  The network-to-host signaling of network metadata is protected by TLS.  For
CIDFI to operate, a CNE never needs the server's identity, and a CNE is never provided decryption keys for
the QUIC communication between the client and server.

Integrity:
: Metadata sharing, including the mapping of packet importance to Destination CIDs, are
integrity protected by QUIC itself and cannot be modified by on-path
network elements.  The communication between client, server, and
network elements is protected by TLS.
: Packet metadata is communicated over a
TLS-encrypted channel from the CIDFI client to its CIDFI-aware network elements,
and mapped to integrity-protected QUIC CIDs.

Internet Survival:
: The QUIC communications between clients
and servers are not changed so CIDFI is expected to work wherever QUIC
works.  The elements involved are only the QUIC
client and server and with the participating CIDFI-aware network elements.
: CIDFI can operate over IPv4, IPv6, IPv4/IPv4 translation (NAT), and IPv6/IPv4
translation (NAT64).

Fast Path Forwarding Support:
: For some differentiated services (e.g., capacity awareness), CIDFI does not require specific
processing by on-path network devices. For others, once a state is programmed (CIDs, for example)
no other forwarding constraint is required at CNEs.

Single Encryption and No Nested Congestion Control:
: CIDFI does not require any tunneling mechanism or any overhead
of multi-layer encryption schemes that would impact CNEs processing. CIDFI uses the
base connection to convey specific signals. Unlike tunneling mechanisms, CIDFI does not
suffer from nested congestion control.

# Network Configuration to Support CIDFI

The network is configured to advertise its support for CIDFI using a variety of means.

Four mechanisms are described in {{?I-D.wing-cidfi-discovery}}: DNS
SVCB records {{!RFC9460}}, IPv6 Provisioning Domains (PvD) {{!RFC8801}}, DHCP {{!RFC2131}}{{!RFC8415}}, and 3GPP PCO.

# Client Operation on Network Attach or Topology Change {#attach}

On initial network attach topology change (see {{topology}}),
the client learns if the network supports CIDFI ({{discovery}}) and
authorizes discovered network elements ({{client-authorizes}}).

## Client Learns Local Network Supports CIDFI {#discovery}

The client uses one on of the four mechanisms defined in {{?I-D.wing-cidfi-discovery}}
to discover CNEs of the networks it is attached to.

If the discovery succeeds (i.e., the client concludes that the local
  and/or ISP network support CIDFI), client processing proceeds to
  {{client-authorizes}}.

If the discovery failed (i.e., the client concludes that the local
  network and ISP do not support CIDFI), client processing stops.

## Client Authorizes CIDFI-aware Network Elements {#client-authorizes}

The response from the previous step in {{discovery}} will contain one or more
CNEs.

The client authorizes each of the CNEs using
a local policy.  This policy is implementation-specific.  An
implementation example might have the users authorize their ISP's CIDFI server
(e.g., allow "cidfi.example.net" if a user's ISP is configured with
"example.net").  Similarly, if none of the CNEs are recognized by the client, the client
might silently avoid using CIDFI on that network.

After authorizing that subset of CNEs, the
client makes a new HTTPS connection to each of those CNEs
and performs PKIX validation of their certificates.
The client MAY have to authenticate itself to the CNE.

The client then obtains the CIDFI nonce and CIDFI HMAC
secret from each CNE used later in {{ownership}} to prove
the client owns its UDP 4-tuple.

~~~~~
{
   "cidfi-path-authentication":[
      {
         "nonce":"ddqwohxGZysgy0BySNh7sNHV5IH9RbE7rqXmg9wb9Npo",
         "hmac-secret":"jLNsCvuU59mt3F4/ePD9jbZ932TfsLSOP2Nx3XnUqc8v"
      }
   ]
}
~~~~~
{: #hmac-ex artwork-align="center" title="Example of CIDFI HMAC and Nonce"}


# Client Operation on Each Connection to a Server

When a QUIC client connects to a QUIC server, the client:

  1. Learns if the server supports CIDFI
     and obtains its mapping of transmitted Destination CIDs to metadata, described
     in {{server-supports-cidfi}}.
  2. Proves ownership of its UDP 4-tuple to
     the on-path CNEs, described in {{ownership}}.
  3. Performs initial metadata exchange
     with the CIDFI network element and server, and server and network element,
     described in {{initial-metadata-exchange}}.
  4. For the duration of the connection, receives network-to-host and performs
     host-to-network updates as network conditions or network requirements change,
     described in {{ongoing}}. Some policies are provided to CNEs to control which network changes can trigger updating clients.


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
transport parameter "enable_cidfi" (TBD1) ({{iana-tp}}) which is remembered for 0-RTT.

If the server does not indicate CIDFI support by means of enable_cidfi transport parameter, the client can still
perform CIDFI -- but does not expect different CIDs to indicate
differentiated behavior.  The client can still signal to its CNE(s) about
the flow, because the client knows some characteristics of the flow it
is receiving.  For example, if the client requested streaming video of
a certain bandwidth from a server or participated in a WebRTC
offer/answer exchange, the client knows some connectivity expectation about the
incoming flow without the server supporting CIDFI.  Processing
continues with the next step.

The QUIC client and server exchange CIDFI information using the new
CIDFI_NEW_CONNECTION_ID_MAPPING frame type as described in
{{initial-metadata-exchange}}.

Processing continues with the next step.

## Client Proves Ownership of its UDP 4-Tuple {#ownership}

> Optimizations to this mechanism are being considered while
  maintaining support for multiple CIDFI and QUIC implementations on
  one host (i.e., by several applications) and support for cellular devices
  providing IP connectivity to other devices (see {{Section 3 of ?RFC7849}}).

To ensure that the client messages to a CNE
pertain only to the client's own UDP 4-tuple, the client sends the
CIDFI nonce protected by the HMAC secret it obtained from
{{client-authorizes}} over the QUIC UDP 4-tuple it is using with the
QUIC server over the path that involves that CNE. The ability to transmit that packet on the same UDP
4-tuple as the QUIC connection indicates ownership of that IP address
and UDP port number.  The nonce and HMAC are sent in a {{!STUN=RFC8489}} indication (STUN
class of 0b01) containing one or more CIDFI-NONCE attributes
({{iana-stun}}).  If there are multiple CNEs
the single STUN indication contains a CIDFI-NONCE attribute from each of
them.  This message is discarded, if received, by the QUIC server.

In order to avoid overloading servers, the client may set the TTL/Hop Limit
to a value that allows to cross the CNE, but then discarded before reaching the server.
For example, the host sets the TTL to "min-ttl" that is returned during CNE discovery.

{{flow-diag-attach}} shows a summarized message flow obtaining
the nonce and HMAC secret from the CNE (steps 1-2) which is performed
on network attach.  The CNE also sends active_cidfi_connection_id_limit
in step 2.


~~~~~ aasvg
 QUIC                            CIDFI-aware            QUIC
client                           edge router           server
  |                                    |                  |
  |  1. HTTPS: Enroll CIDFI router to participate         |
  +----------------------------------->|                  |
  |  2. HTTPS: Ok.  nonce=12345        |                  |
  |     active_cidfi_connection_id_limit                  |
  |<-----------------------------------+                  |
  |                                    |                  |
~~~~~
{: #flow-diag-attach title="Example of Flow Exchange" artwork-align="center"}


Later, when connecting to a new QUIC server, the client
determines if there are on-path CIDFI Network Elements by sending the
nonce and HMAC in the same UDP 4-tuple as the QUIC connect
(step 2).  This is necessary to deal with both IP address spoofing
and with multiple QUIC+CIDFI implementations running on the same
host; each QUIC+CIDFI implementation pair should only be able to
modify treatment of its own flows, not of other flows to other
UDP flows running on that same host.

If a CIDFI Network Element is present on the path it processes the STUN
Indication and sends a response to the client over HTTP using the
HTTP channel established above.  It decrements the IPv4 TTL or IPv6
Hop Limit and forwards the STUN Indication along its normal path,
to accommodate another CIDFI Network Element farther away from the
client.


~~~~~ aasvg
 QUIC                            CIDFI-aware            QUIC
client                           edge router           server
  |                                    |                  |
  |  1. QUIC Initial, transport parameter=enable_cidfi    |
  +------------------------------------------------------>|
  |  2. STUN Indication, nonce=12345, hmac=e8FEc          |
  +------------------------------------------------------>|
  |                                    |                  |
  |                                    |           3. discarded
  |                                    |                  |
  |                 4. "I saw my nonce, HMAC is valid"    |
  |                                    |                  |
  |  5. Valid STUN Indication processed|                  |
  |<-----------------------------------+                  |
  |                                    |                  |
  |  6. HTTPS: "Map DCID=xyz as high importance"          |
  +----------------------------------->|                  |
  |  7. QUIC Initial, transport parameter=enable_cidfi    |
  |<------------------------------------------------------+
  |  8. HTTPS: Ok                      |                  |
  |<-----------------------------------+                  |
~~~~~
{: #flow-diag-connect title="Example of Flow to New Server" artwork-align="center"}

> Note the above message
flow shows an initial QUIC handshake for simplicity (steps 1 and 7)
but because of QUIC connection migration ({{Section 9 of QUIC}}) the
QUIC messages might appear later.

> Also, "Map DCID=xyz as high importance" refers to a CID chosen by
> the client (for traffic destined towards the client) and not
> the DCID used by the client to communicate with the server.


The short header's Destination Connection ID (DCID) can be 0 bytes or
as short as 8 bits, so multiple QUIC clients on the same host or
on different hosts behind a NAT are likely to use the
same incoming Destination CID on their own UDP 4-tuple (Birthday Paradox). The STUN
Indication message allows the CIDFI network element to distinguish
each QUIC client's UDP 4-tuple -- both between hosts and between QUIC+CIDFI
implementations on the same host (implemented within an application).

To reduce CIDFI setup time the client STUN Indication MAY be sent at
the same time as it establishes connection with the QUIC server.

To prevent replay attacks, the Nonce is usable only for authenticating
one UDP 4-tuple.  When the connection is migrated ({{Section 9 of
QUIC}}) the CNE won't apply any CIDFI behavior to
that newly-migrated connection.  The client will have to restart
CIDFI procedures at the beginning ({{attach}}).

After the CIDFI Network Element receives the STUN Indication it
informs the client by sending an HTTP message to the client.  Details TBD.

As the proof of ownership of its UDP 4-tuple is only useful to CIDFI
Network Elements near the client, the client MAY reduce traffic to the
server by modulating the IPv4 TTL or IPv6 Hop Limit of its STUN Indication messages. The client SHOULD set TTL/Hop Limit to "min-ttl". The client MAY use other values (e.g., explicit configuration, inferred from probe messages).

Processing continues with the next step.

### STUN CIDFI-NONCE Attribute

The format of the STUN CIDFI-NONCE attribute is shown in {{fig-stun-cidfi-nonce}}.

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
{: artwork-align="center" #fig-stun-cidfi-nonce title="Format of STUN CIDFI-NONCE Attribute"}

The nonce is 128 bits obtained from the CIDFI network element.  The
HMAC-output field is computed per {{?RFC5869}} using the CIDFI network
element-provided HMAC secret, and the CIDFI network element-provided
Nonce concatenated with the fixed string "cidfi" (without quotes),
shown below with "|" denoting concatenation.

~~~~~
  HMAC-output = HMAC-SHA256( hmac-secret, nonce | "cidfi" )
~~~~~

When there are multiple CIDFI Network Elements on the network,
multiple CIDFI-NONCE attributes are sent in a single STUN Indication
message.

## Initial Metadata Exchange {#initial-metadata-exchange}

If the server indicated support for CIDFI during the QUIC handshake,
the client uses its HTTPS channel with each of the CNEs it
previously authorized for CIDFI participation to map client-chosen
Destination CIDs to metadata for that CID.  As server support
of the QUIC CIDFI transport parameter is remembered for 0-RTT, the
client can immediately send the nonce.

Over the QUIC connection with the server, the client sends QUIC
CIDFI_NEW_CONNECTION_ID_MAPPING frames which map the destination CID
to its metadata (e.g., high priority), not to exceed active_cidfi_connection_id_limit.

As with NEW_CONNECTION_ID, the client
should allocate additional connection IDs retain client privacy during
connection migration ({{Section 9.5 of QUIC}}) and those additional
CIDs should also be communicated via CIDFI_NEW_CONNECTION_ID.  In
anticipation of connection migration those additional connection IDs
are not communicated to the existing network's CNEs, but only to the
new network's CNEs.

Connection IDs which are communicated using NEW_CONNECTION_ID do
not receive per-packet CIDFI treatment.  But their contribution
to bandwidth consumption is considered by the CNE.

Note that the source IP address and source UDP port number are not signaled
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
highlighted in the simplified message flow in {{ex-lost-nonce}}.

~~~~~ aasvg
                                 CIDFI-aware            QUIC
client                           edge router           server
  |                                    |                  |
  |  HTTPS: Enroll CIDFI router to participate            |
  +----------------------------------->|                  |
  |  HTTPS: Ok.  nonce=12345           |                  |
  |<-----------------------------------+                  |
  |                                    |                  |
  :                                    :                  :
  |                                    |                  |
  |  QUIC Initial, transport parameter=enable_cidfi       |
  +------------------------------------------------------>|
  |  STUN Indication, nonce=12345, HMAC=8f93e             |
  +--------------------> X (lost)      |                  |
  |                                    |                  |
  |  HTTPS: "Map DCID=xyz as high importance"             |
  +----------------------------------->|                  |
  |  HTTPS: 403, new Nonce=5678        |                  |
  |<-----------------------------------|                  |
  |  STUN Indication, nonce=5678, HMAC=8f93e              |
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
{: #ex-lost-nonce artwork-align="center" title="Example of a Client Re-transmitting Lost Nonce"}


After the initial metadata is exchanged, processing continues with
ongoing host-to-network and network-to-host updates as described in
{{ongoing}}.

There are two types of metadata exchanged, described in the following sub-sections.

### Host to Network Signaling {#host-to-network}

The server communicates to CNEs via the client which then
communicates with the CNE(s).  While this adds
communication delay, it allows the user at the client to authorize
the metadata communication about its own incoming (and outgoing) traffic.

The communication from the client to the server are using a CIDFI-dedicated
QUIC stream over the same QUIC connection as their primary communication ({{ex-comm}}).

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
{: #ex-comm Title artwork-align="center" title="Example of CIDFI Communication"}

To each of the network elements authorized by the client, the client
sends the mappings of the server's transmitted Destination CIDs to
packet metadata (see {{metadata-exchanged}}).

### Network to Host Signaling {#network-to-host}

The CNE sends network performance information to the server
which is intended to influence the sender's traffic rate (such as
improving or reducing fidelity of the audio or video).  In {{ex-comm-metadata}},
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
{: #ex-comm-metadata Title artwork-align="center" title="Example of CIDFI Communication with Metadata Sharing"}

The communication from the client to the server is using a CIDFI-dedicated
QUIC stream over the same QUIC connection as their primary communication.

The CNE can update the client with whenever
the metadata about the connection changes significantly, but MUST NOT
update more frequently than once every second.

The metadata exchanged over this channel is described in {{metadata-exchanged}}.


# Ongoing Signaling {#ongoing}

Throughout the life of the connection host-to-network and network-to-host
signaling is updated whenever characteristics change. Still, some policies are provided to control when these updates are triggers. Such policies are meant to preserve the connection stability.

Typically, due to environmental changes on wireless networks or other user's
traffic patterns, a particular flow may be able to operate faster or
might need to operate slower.  The relevant CNE SHOULD signal
such conditions to the client ({{network-to-host}}), which can then
relay that information to the server using either CIDFI or via its
application.

For example, a  streaming video client might be retrieving low quality
video because one of their invoked CNEs indicated constrained
bandwidth.  Later, after moving closer to an antenna, more bandwidth is
available which is signaled by the CNE to the client.
The client uses that signal to now request higher-quality video from
the server.

Similarly, the CIDFI client may begin receiving traffic
with different characteristics which might be be signaled to the CNEs.

For example, a client might be participating in an audio-only call
which is modified to audio and video, requiring additional bandwidth
and likely new CIDs to differentiate the video packets from the audio
packets.

# Interaction with Load Balancers {#load-balancers}

QUIC servers are likely to be behind CID-aware load balancers {{?I-D.ietf-quic-load-balancers}}.

With CIDFI, all the communications to the load-balanced QUIC server are over the same UDP 4-tuple
as the primary QUIC connection but in a different QUIC stream.  This means
no changes are required to ECMP load balancers or to CID-aware load balancers
when using a CIDFI-aware back-end QUIC server.

Load balancers providing QUIC-to-TCP interworking are incompatible with
CIDFI because TCP lacks QUIC's stream identification.


# Topology Change {#topology}

When the topology changes the client will transmit from a new IP
address -- such as switching to a backup WAN connection, or such as
switching from Wi-Fi to 5G.  The server will consider
this as a connection migration ({{Section 9 of QUIC}}) and will issue a
PATH_CHALLENGE.  If the client is aware of the topology change (such
as attaching to a different network), the client would also change its
QUIC Destination CID ({{Section 9 of QUIC}}).

When the CIDFI-aware client determines that it is connected to a new
network or has received a QUIC PATH_CHALLENGE, the CIDFI-aware client
MUST re-discover its CNEs ({{discovery}}) and continue with normal CIDFI
processing with any discovered CNEs.  This usually means repeating
the initial metadata exchange ({{initial-metadata-exchange}}) to prove
path ownership.

# Flushing Mapping State

When the server supports CIDFI the metadata mapping creates additional
state in the client, CIDFI Network Elements, and the
server.

Between the QUIC client and server when a mapping is no longer
needed it can be cleaned up with RETIRE_CONNECTION_ID.  If that
connection ID was mapped in one or more CNEs, the client SHOULD
also remove that mapping state from the CNEs.  This allows the
mapping state to be used for other CIDFI implementations on the
same host or by other hosts (belonging to the same subscriber)
or by other subscribers.

As a client can disappear from a network without informing its CNE and
are unlikely to voluntarily clean up CNE state even if they remain
connected to the network, the CNE should retire its CIDFI state after
3 minutes of bi-directional inactivity on that UDP 4-tuple or a more
convenient time such as when it normally flushes its UDP NAT binding
for bi-directional inactivity.


# Details of Metadata Exchanged {#metadata-exchanged}

This section describes the metadata that can be exchanged from a
CNE to a server (generally network
performance information) and from the server to a CNE.


## Server to CIDFI-aware Network Element

Because there is no direct communication from the server to
a CNE, the communication is relayed through the client.

The communications from servers to CNEs do not occur directly,
but rather through the client.

Two types of mapping metadata are described in the following sub-sections: metadata parameters
and DSCP values.

### Mapping Metadata Parameters to DCIDs {#mapping-parameters}

Several of metadata parameters can be mapped to Destination CIDs:

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
knowledge.

> TODO: provide enough details to create interoperable
implementations.

Over the CIDFI-dedicated QUIC stream, the server sends mapping
information to the client when then propagates that information
to each of the CNEs. An example is shown in {{fig-import}}.


~~~~~ json
{
   "metadata-parameters":[
      {
         "quicversion":1,
         "dcidlength":3,
         "map":[
            {
               "import":17,
               "burst":83,
               "delaybudget":71,
               "dcids":[
                  551,
                  381
               ]
            },
            {
               "import":3,
               "burst":888,
               "delaybudget":180,
               "dcids":[
                  89,
                  983
               ]
            },
            {
               "import":7,
               "burst":37,
               "delaybudget":55,
               "dcids":[
                  33
               ]
            }
         ]
      }
   ]
}
~~~~~
{: #fig-import artwork-align="left" title="Example JSON for Flow Importance"}

> Note: {{fig-import}} lists sample attributes and they will be discussed in detail in a separate document.

### Mapping DiffServ Code Point (DSCP) to DCIDs {#mapping-dscp}

A mapping from Destination CID to DiffServ code point
{{!RFC2474}} leverages existing DiffServ handling that may already
exist in the CIDFI network element.  If there are downstream network
elements configured with the same DSCP the CIDFI
network element could mark the packet with that code point as well.

Signaling the DSCP values for different QUIC Destination CIDs
increases the edge network's confidence that the sender's DiffServ intent
is preserved into the edge network, even if the DSCP bits were
modified en route to the edge network (e.g., {{pathologies}}).

Over the CIDFI-dedicated QUIC stream, the server sends the mapping
information to the client when then propagates that information
to each of the CNEs.

An example is shown in {{fig-dscp-json}}.

~~~~~ json
{
   "dscp":[
      {
         "quicversion":1,
         "dcidlength":3,
         "map":[
            {
               "dscp":10,
               "dcids":[
                  123,
                  456
               ]
            },
            {
               "dscp":46,
               "dcids":[
                  998,
                  183
               ]
            }
         ]
      }
   ]
}
~~~~~
{: #fig-dscp-json artwork-align="left" title="Example JSON for DSCP Mapping"}



## CIDFI-aware Network Element to Server

The CIDFI-aware client informs the CNE of the client's
received Destination CIDs.  As bandwidth availability to that client
changes, the CNE updates the client with new
metadata.


~~~~~
{
   "dcid":123,
   "bandwidth":"1Mbps"
}
~~~~~

The client then sends that information to the server in the CIDFI-dedicated
QUIC stream associated with that same Connection ID.

# Privacy Considerations

## Privacy-Aware Metadata Sharing in Network Relationships

If the network operator and the server have a business relationship,
the server can sign or attest the metadata using, e.g., JSON Web Token (JWT) {{?RFC7519}} or CBOR Web Token (CWT) {{?RFC8392}}. The
attested metadata will be sent from the server to the client. The client
will decide whether to convey the attested metadata to the CNE, considering
privacy reasons, as it may reveal the identity of the server to the network.
The client may use any local policy or involve the end-user in the decision-making process regarding
whether to reveal the identity of the server to the network or not.
If the attested metadata is sent to the CNE from the client, the attestation
will be utilized by the CNE, acting as a Relying Party (e.g., {{Section 7.1 of ?RFC9334}}), to determine the
level of trust it wishes to place in the attested metadata. The relying party
may choose to trust or not trust the attestation.

# Discussion Points

This section discusses known issues that would benefit from wider discussion.

## Client versus Server Signaling CID-to-importance Mapping

Need to evaluate number of round trips (and other overhead) of client
signaling CID-to-importance mapping or server signaling CID-to-importance
mapping.

## Overhead of QUIC DCID Packet Examination

If CID-to-importance metadata was signaled by the server as described
in {{host-to-network}}, the CNE have to
examine the UDP payload of each packet for a matching Destination CID
for the lifetime of the connection.  This is somewhat assuaged by
the STUN nonce transmitted which may well be an easier signal to
identify.


## Interaction with Wi-Fi Packet Aggregation

Per-packet metadata influences transmission of that packet but may
well conflict with some Wi-Fi optimizations (e.g., {{wifi-aggregation}})
and similar 5G optimizations.

This impact needs further study.


## Overhead of Mapping CIDs to Packet Metadata

Network Elements have to maintain a mapping between each UDP 4-tuple
and QUIC CID and its DSCP code point.  This also needs updating
whenever sender changes its CID.  This is awkward.

An alternative is a fixed mapping of QUIC CIDs to their meanings,
as proposed in {{?I-D.zmlk-quic-te}}.  However, this will ossify
the meaning of those QUIC CIDs.  It also requires all networks to
agree on the meaning of those QUIC CIDs.



## Improve CIDFI Initialization Time

Find approaches to further reduce network communications to start CIDFI.


## Primary QUIC Channel CID Change {#primary-cid-change}

Because the CIDFI network element, QUIC server, and QUIC client all
cooperate to share the primary QUIC connection's Destination CID,
when a new CIDFI network element is involved (e.g., due to client
attaching to a different network), a new Destination CID SHOULD
be used for the reasons discussed in {{Section 9.5 of QUIC}}).

> We need clear way to signal which DCIDs can be used for 'this'
network attach and which DCIDs are for a migrated connection.  Probably
belongs in the QUIC transport parameter signaling?


# State Maintenance

A CNE can safely remove state after UDP inactivity timeout {{Section
4.3 of !RFC4787}}.  The CIDFI client MUST re-signal its CNE(s) when it
receives a QUIC path validation message, as that indicates a NAT
rebinding occurred.  A CNE's state can also be cleared by signaling from
the CIDFI client, such as when closing the application; however, this
signal cannot be relied upon due to network disconnect, battery
depletion, and suchlike.

> TODO: Probably want keepalives on client->CNE communication. To be assessed.

# API Integration for QUIC Stream and Packet-Level Prioritization {#api-integration}

For each QUIC stream requiring differentiated service, the QUIC stack can
map that stream to a different Destination CID. The application-level code
would require an API to instruct the QUIC stack that a particular stream
needs differentiated service. Similarly, if the application-level code seeks
differentiated service for packets within a stream (e.g., prioritizing P-frames
over I-Frames in a video stream), it would need an API to inform the QUIC stack
that different packets within the QUIC stream require differentiated services
and to map these packets to different Destination CIDs.

**Where packet-level differentiation is not desired, such API enhancements
are not needed**.  In that situation, the CIDFI-aware client and CIDFI-aware
network elements can utilize bandwidth information to optimize their video
streaming usage and their interactive audio/video streams, without the
benefit of packet-level differentiation.

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

Other than what can be inferred from a destination IP address, the server's identity is not disclosed to the CIDFI Network Elements,
thus maintaining the end user's privacy.  Communications are relayed through the client because only the
client knows the identity of the server and can validate
its certificate.

Spoofing Attacks:
: For an attacker to succeed with the nonce challenge against a victim's UDP 4-tuple, an attacker has to send a STUN CIDFI-NONCE packet using the victim's source IP address and a valid HMAC. A valid HMAC can be obtained by the attacker making its own connection to the CIDFI-aware server and spoofing the source IP address and UDP port number of the victim.
: If the client does not support CIDFI, the attacker can influence the packet treatment of the victim's UDP 4-tuple.
: If the client implements CIDFI, a CIDFI network element can identify an IP address spoofing attack. Concretely, the CNE will receive two HTTPS connections describing the same DCID; one connection from the attacker and another one from the victim. The CNE will then issue unique Nonces and HMACs to both the attacker and victim, and both the attacker and victim should send the STUN Indication on that same UDP 4-tuple. Such an event should trigger an alarm on the CNE. In this scenario, it is recommended that both the attacker and the victim be denied CIDFI access.
: The spoofing of a victim's IP address is prevented by the network using network ingress filtering ({{!RFC2827}}, {{!RFC7513}}, {{?RFC6105}}, and/or {{!RFC6620}}).

On-Path Attacks:
: An on-path attacker can observe the victim's Discovery Packet, block it, and then forward the packet within the attacker's 5-tuple. Subsequently, the on-path attacker can 'steal' the victim's CIDFI control from the victim's UDP 4-tuple, causing the victim's CIDFI signaling for that UDP 4-tuple to influence the attacker's UDP 4-tuple.
: Although the on-path attacker can't directly observe the encrypted CIDFI signaling, this attack effectively disables the victim's CIDFI treatment, making it accessible to the attacker. The attacker can send NEW_CONNECTION_ID frames to the server with the victim's (observed) Destination CID, effectively claiming the victim's CIDFI signaling for themselves. An on-path attacker can do a lot more damage by blocking or rate-limiting the victim's traffic.

# IANA Considerations

## New QUIC Transport Parameter {#iana-tp}

This document requests IANA to register the following new permanent QUIC transport parameter
in the "QUIC Transport Parameters" registry under the "QUIC" registry group available at {{IANA-QUIC}}:

|Value| Parameter Name| Reference|
|TBD1| enable_cidfi| This-Document|
{: title="New QUIC Transport Parameter"}

## New QUIC Frame Type {#sec-iana-new-frame}

This document requests IANA to register a new value in in the "QUIC Frame Types" registry under the "QUIC" registry group available at {{IANA-QUIC}}:

Value:
: TBDF2

Frame Name:
: CIDFI_NEW_CONNECTION_ID_MAPPING

Status:
: permanent

Specification:
: This-Document

## New Well-known URI "cidfi-aware" {#iana-uri}

This document requests IANA to register the new well-known URI "cidfi" in the
"Well-Known URIs" registry available at {{IANA-WKU}}.

## New STUN Attribute {#iana-stun}

This document requests IANA to register the new STUN attribute "CIDFI-NONCE"
in the "STUN Attributes" registry available at {{IANA-STUN}}.

--- back


# Extending CIDFI to Other Protocols {#extending}

CIDFI can be extended to other protocols including TCP, SCTP, RTP, and SRTP,
and bespoke UDP protocols.

An extension to each protocol is described below which retains the
ability of the client to prove its ownership of the 5-tuple to a CNE.

## DTLS

DTLS is used by WebRTC and SIP for establishing interactive real-time audio,
video, and screen sharing, which benefit from knowing network characteristics
(n2h signaling) and benefit from prioritizing audio over video (h2n signaling).
{{?RFC9146}} defines an extension to add a Connection ID (CID) to the
DTLS record layer. DTLS CID can be leveraged by CIDFI to communicate
per-connection information from endpoint to CNE and vice-versa.

## TCP

To prove ownership of the TCP 4-tuple, TCP can utilize a new TCP
option to carry the CNE's nonce and HMAC-output.  This TCP option can be carried
in both the TCP SYN and in some subsequent packets to avoid consuming the entire
TCP option space (40 bytes).  Sub-options can be defined to carry pieces of
the Nonce and HMAC output, with the first piece of the Nonce in the TCP SYN
so the CIDFI network element can be triggered to begin looking for the subsequent
TCP frames containing the rest of the CIDFI nonce and CIDFI HMAC-output.  For example,

1. send TCP SYN + CIDFI option (including Nonce bits 0-63)
2. if received TCP SYNACK does not indicate CIDFI support, stop sending CIDFI option
3. send next TCP packet + CIDFI option (including Nonce bytes 64-128)
4. send next TCP packet + CIDFI option (including HMAC-output bits 0-127)
5. send next TCP packet + CIDFI option (including HMAC-output bytes 128-256)

To shorten this further we might truncate the HMAC output and/or
truncate the Nonce after security evaluation.

## SCTP

If SCTP is sent directly over IP, proof of ownership of the
SCTP 4-tuple can be achieved using an extension to its INIT
packets, similar to what is described above for TCP SYN.

If SCTP is run over UDP, the same proof of ownership of the UDP
4-tuple as described in {{ownership}} can be performed.


## RTP and SRTP

The RTP Synchronization Source (SSRC) is in the clear for {{?RTP=RFC3550}}, {{?SRTP=RFC3711}},
and {{?cryptex=RFC9335}}.  If the SSRC is signaled similarly to CID, RTP could also
benefit from CIDFI.  CIDFI network elements could be told the mapping of SSRC values to
importance and schedule those SSRCs accordingly.  However, SSRC is used in playout (jitter)
buffers and a new SSRC seen by a receiver will cause confusion.  Thus, overloading SSRC
to mean both 'packet importance' for CIDFI and 'synchronization source' will require
engineering work on the RTP receiver to treat all the signaled SSRCs as one source for
purposes of its playout buffer.

RTP over QUIC {{?I-D.ietf-avtcore-rtp-over-quic}} is another approach which exposes
QUIC headers to the network (which have CIDs) and does not overload the RTP SSRC.  The
Media over QUIC (MOQ) working group includes RTP over QUIC as one of its use cases
{{Section 3.1 of ?I-D.ietf-moq-requirements}}.


## Bespoke UDP Application Protocols

To work with CIDFI, other UDP application protocols would have to
prove ownership of their UDP 4-tuple ({{ownership}}) and extend their
protocol to include a connection identifier in the first several bits
of each of their UDP packets.

Alternatively, rather than modifying the application protocol it could be run
over {{QUIC}}.


# Acknowledgments
{:numbered="false"}

Thanks to Dave Täht, Magnus Westerlund, Christian Huitema, Gorry Fairhurst,
and Tom Herbert for hallway discussions and feedback at TSVWG that encouraged
the authors to consider the approach described in this document.  Thanks to
Ben Schwartz for suggesting PvD as an alternative discovery mechanism.

