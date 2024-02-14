---
title: "Discovery of CIDFI-aware Network Elements"
abbrev: "CIDFI NE Discovery"
category: std


docname: draft-wing-cidfi-discovery-latest
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


  IANA-WKU:
    title: "Well-known URIs"
    target: https://www.iana.org/assignments/well-known-uris/well-known-uris.xhtml
    date: 2023-06-20

  IANA-SVCB:
    title: "DNS Service Bindings (SVCB)"
    target: https://www.iana.org/assignments/dns-svcb/dns-svcb.xhtml
    date: 2023-06-13

  IANA-PVD:
    title: Provisioning Domains (PvDs)
    target: https://www.iana.org/assignments/pvds/pvds.xhtml#additional-information-pvd-keys
    date: 2020-08-13


--- abstract

Host-to-network signaling and network-to-host signaling can improve
the user experience to adapt to network's constraints and share expected application needs, and thus to provide
differentiated service to a flow and to packets within a flow. The differentiated service may be provided at the network (e.g., packet prioritization), the server (e.g., adaptive transmission), or both.
Such an approach is called CIDFI, for Collaborative and Interoperable Dissemination of Flow Indicators.

A key componenet in a CIDFI system is to discover whether a network is CIDFI-capable, and if so discover
a set of CIDFI-aware Network Elements (CNEs) that will be involved in the Host-to-network signaling and network-to-host signaling.
This document discusses a set of discovery methods.

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

{{?I-D.wing-cidfi}} defines CIDFI (pronounced "sid fye") which is a system
of several protocols that allow communicating about a connection from the network to the server and the server to the network.
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

This document focuses on the discovery step. On initial network attach topology change,
the client learns if the network supports CIDFI ({{discovery}}) and
authorizes discovered network elements ({{client-authorizes}}) as a function of a local policy.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

The document makes use of the following terms:

CNE:
: CIDFI-aware Network Element, a network element that
supports this CIDFI specification.  This is typically a router.

Differentiated service:
: Refers to a differentiated processing that can be provided to a flow (or specific packets within a flow) by a network, client, or server.
: Examples of differentiated service are: prioritization, adaptive transmission, or traffic steering.


# Network Configuration to Support CIDFI

The network is configured to advertise its support for CIDFI.

For this step, four mechanisms are described in this document: DNS
SVCB records {{!RFC9460}}, IPv6 Provisioning Domains (PvD) {{!RFC8801}}, DHCP {{!RFC2131}}{{!RFC8415}}, and 3GPP PCO.
These are described in the following sub-sections.

> Standardizing all or some of these mechanisms is for further discussion.


## DNS SVCB Records

This document defines a new DNS Service Binding parameter "cidfi-aware" in
{{iana-svcb}} and a new Special-Use Domain Name "cidfi.arpa" in
{{iana-sudn}}.

The local network is configured to respond to DNS SVCB
{{!RFC9460}} queries with ServiceMode ({{Section
2.4.3 of !RFC9460}}) for "_cidfi-aware.cidfi.arpa" with
the DNS names of that network's and upstream network's CIDFI-aware
network elements (CNEs).  If upstream networks also support CIDFI (e.g., the
ISP network) those SVCB records are aggregated into the local DNS
server's response by the local network's recursive DNS resolvers.  For
example, a query for "_cidfi-aware.cidfi.arpa" might return two answers
for the two CNEs on the local network, one belonging
to the local ISP (example.net) and the other belonging to the local
Wi-Fi network (example.com).

~~~~~
_cidfi-aware.cidfi.arpa. 7200 IN SVCB 0 service-cidfi.example.net. (
    alpn=h3 cidfipathauth=/path-auth-query{?cidfi}
    cidfimetadata=/cidfi-metadata
    )
_cidfi-aware.cidfi.arpa. 7200 IN SVCB 0 wifi.example.com. (
    alpn=h3 cidfipathauth=/path-auth-query{?cidfi}
    cidfimetadata=/cidfi-metadata
    )
~~~~~
{: #svcb-ex artwork-align="center" title="Example of SVCB Records"}

When multihoming, the multihome-capable CPE aggregates all upstream
networks' "_cidfi-aware.cidfi.arpa" responses into the response sent to
its locally-connected clients.


## Provisioning Domains

The CIDFI networks are configured to set the H-flag so clients can
request PvD Additional Information ({{Section 4.1 of !RFC8801}}).

The "application/pvd+json" returned looks like what is depicted in {{pvd-ex}} when there are two
CIDFI-aware network elements, service-cidfi and wi-fi.

~~~~~
{
   "cidfi":[
      {
         "cidfinode":"service-cidfi.example.net",
         "min-ttl":3,
         "cidfipathauth":"/path-auth-query{?cidfi}",
         "cidfimetadata":"/cidfi-metadata"
      },
      {
         "cidfinode":"wi-fi.example.net",
         "min-ttl":2,
         "cidfipathauth":"/path-auth-query{?cidfi}",
         "cidfimetadata":"/cidfi-metadata"
      }
   ]
}
~~~~~
{: #pvd-ex artwork-align="center" title="Example of PvD Information"}

Multiple CIDFI-aware network elements on a network path will require
propagating the Provisioning Domain Additional Information.  For
example, a CIDFI-aware Wi-Fi access point connected to a CIDFI-aware
5G network will require the information for both CIDFI networks be available
to the client, in a single Provisioning Domain Additional Information
request.  This means the Wi-Fi access point has to obtain that information
so the Wi-Fi access point can provide both the 5G network's information
and the Wi-Fi access point's information.

## DHCP or 3GPP PCO

The network is configured to respond to DHCPv6, DHCPv4 sub-option,
or 3GPP PCO (Protocol Configuration Option) Information Element.

# Client Learns Local Network Supports CIDFI {#discovery}

For this step, four mechanisms are identified: DNS SVCB records, IPv6
PvD, DHCP, or 3GPP PCO.  These are described
in the following sub-sections.

In all cases below,

- if the discovery succeeds (i.e., the client concludes that the local
  and/or ISP network support CIDFI) client processing proceeds to
  {{client-authorizes}}.

- if the discovery failed (i.e., the client concludes that the local
  network and ISP do not support CIDFI) client processing stops.

## Client Learns Using DNS SVCB

The client determines if the local network provides CIDFI service by
issuing a query to the local DNS server for
"_cidfi-aware.cidfi.arpa." with the SVCB resource record type (64)
{{!RFC9460}}.

## Client Learns Using Provisioning Domain

The client determines if the local network supports CIDFI by
querying https://\<PvD-ID\>/.well-known/pvd as described in {{Section
4.1 of !RFC8801}}.

## Client Learns Using DHCP or 3GPP PCO

The client determines that a local network is CIDFI-capable if the
client receives an explicit signal from the network, e.g., via a
dedicated DHCP option or a 3GPP PCO (Protocol Configuration Option)
Information Element. An example of explicit signal would be a DHCPv6
option or DHCPv4 sub-option that that is returned as part of
{{?RFC7839}}.

# Client Authorizes CIDFI-aware Network Elements {#client-authorizes}

The client authorizes each of the CNEs using
a local policy.  This policy is implementation-specific.  An
implementation example might have the users authorize their ISP's CIDFI server
(e.g., allow "cidfi.example.net" if a user's ISP is configured with
"example.net").  Similarly, if none of the CNEs are recognized by the client, the client
might silently avoid using CIDFI on that network.


# Security Considerations

TBC.

# IANA Considerations

## New Well-known URI "cidfi-aware" {#iana-uri}

This document requests IANA to register the new well-known URI "cidfi" in the
"Well-Known URIs" registry available at {{IANA-WKU}}.

## New Special-use Domain Name {#iana-sudn}

This document requests IANA to register new a special-use domain name cidfi.arpa for DNS SVCB discovery.

## New DNS Service Binding (SVCB) {#iana-svcb}

This document requests IANA to register the new DNS SVCB "_cidfi-aware" in
the "DNS Service Bindings (SVCB)" registry available at {{IANA-SVCB}}.

The document also requests IANA to register the following service parameter
in the "Service Parameter Keys (SvcParamKeys)" registry {{IANA-SVCB}}:

Number:
: TBD

Name:
: min-ttl

Meaning:
:The minimum IPv4 TTL or IPv6 Hop Limit to use for a connection.

Reference:
: This-Document

## New Provisioning Domain Additional Information Key {#iana-pvd}

This document requests IANA to register a new JSON key in the
Provisioning Domains Additional Information registry at {{IANA-PVD}}:

~~~~~
JSON key: cidfi
Description: CID Flow Indicator
Type: array of cidfi details
Example: ["cidfinode": "service.example.net", "cidfipathauth":
          "/authpath", "cidfimetadata": "/meta"]
~~~~~

Additionally, this document requests creating a new registry, entitled "CIDFI JSON Keys" under
the Provisioning Domains Additional Information registry group {{IANA-PVD}}.
The policy for assigning new entries in this registry is Expert Review {{Section 4.5 of !RFC8126}}.
The structure of this registry is identical to the Provisioning Domains Additional Information registry group.
The initial content of this registry is provided below:

~~~~~
JSON key: cidfinode
Description: FQDN of CIDFI node
Type: string
Example: service.example.net

JSON key: min-ttl
Description: The minimum TTL or Hop Limit to reach a CNE
Type: Unsigned integer
Example: 5

JSON key: cidfipathauth
Description: authentication and authorization path for CIDFI
type: string
Example: "/authpath"

JSON key: cidfimetadata
Description: metadata path for CIDFI
type: string
example: "/metadata"
~~~~~


--- back

# Acknowledgments
{:numbered="false"}

Thanks to Dave Täht, Magnus Westerlund, Christian Huitema, Gorry Fairhurst,
and Tom Herbert for hallway discussions and feedback at TSVWG that encouraged
the authors to consider the approach described in this document.  Thanks to
Ben Schwartz for suggesting PvD as an alternative discovery mechanism.

