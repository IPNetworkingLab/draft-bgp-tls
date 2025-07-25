---
title: "BGP over TLS/TCP"
abbrev: bgp-tls
docname: draft-wirtgen-bgp-tls-latest
category: exp

submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number: 03
date:
consensus: true
v: 3
area: "Routing"
workgroup: "IDR"
keyword:
 - tcp
 - tls
 - bgp
 - tcp-ao

venue:
  group: "IDR"
  type: "Working Group"
  mail: "idr@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/idr/"
  github: "IPNetworkingLab/draft-bgp-tls"

author:
 -
    name: Thomas Wirtgen
    organization: UCLouvain 
    email: thomas.wirtgen@uclouvain.be
 -
    name: Olivier Bonaventure
    organization: UCLouvain & WELRI
    email: olivier.bonaventure@uclouvain.be



normative:
  RFC7301:
  RFC4271:
  RFC4272:
  RFC7301:
  RFC5925:
  RFC2385:

  I-D.draft-piraux-tcp-ao-tls:
  #  title: Opportunistic TCP-AO with TLS
  #  author:
  #    -
  #      ins: M. Piraux
  #      name: Maxime Piraux
  #    -
  #      ins: O. Bonaventure
  #      name: Olivier Bonaventure
  #    -
  #      ins: T. Wirtgen
  #      name: Thomas Wirtgen
  #  date: 2023
  #  seriesinfo: Internet draft, draft-bonventure-tcp-ao-tls, work in progress

informative:
  I-D.draft-retana-idr-bgp-quic:
  RFC5082:
  RFC8446:
  RFC9000:
  BGPOST: DOI.10.1145/3696406
  SURVEY:  http://hdl.handle.net/2078.1/292356
    title: Survey on the Configuration of BGP routers
    author:
      -
        ins: T. Wirtgen
        name: Thomas Wirtgen
    date: 2024
    seriesinfo: Technical report, http://hdl.handle.net/2078.1/292356
  IPCERT: 
    title: We've Issued Our First IP Address Certificate
    author:
      -
         ins: A. Gable
         name: Aaron Gable
    date: 2025     
    seriesinfo: Blog https://letsencrypt.org/2025/07/01/issuing-our-first-ip-address-certificate/
    
--- abstract

This document specifies the utilization of TCP/TLS to support BGP.

--- middle

# Introduction

The Border Gateway Protocol (BGP) {{RFC4271}} relies on the TCP protocol
to establish BGP sessions between routers. There are ongoing discussions
within the IETF {{I-D.draft-retana-idr-bgp-quic}} to replace TCP with 
the QUIC protocol {{RFC9000}}. QUIC brings many features compared to
TCP including security, the support of multiple streams or datagrams.

From a security viewpoint, an important benefit of QUIC compared to TCP is
that QUIC by design prevents injection attacks that are possible when
TCP is used by BGP {{RFC4272}}. Several techniques can be used by BGP routers
to counter this attacks {{RFC5082}} {{RFC5925}}. TCP-AO {{RFC5925}}
authenticates the packets exchanged over a BGP session and provides similar
features as QUIC. However, it a recent survey {{SURVEY}} indicates that it remains
less used than TCP over MD5 {{RFC2385}}. 

The widespread deployment of TLS {{RFC8446}} combined with the possibility of
deriving TCP-AO keys from the TLS handshake {{I-D.draft-piraux-tcp-ao-tls}}
creates an interest in using TLS to secure BGP sessions. While TLS is mainly
used to interact with servers that have a certificate bound to a domain name,
it is also possible to use TLS certificates bound to IP addresses {{IPCERT}}. 
Such certificates are very useful to use BGP over TLS/TCP.

This document
describes how BGP can operate over TCP/TLS. Experience in implementing BGP
over TLS/TCP {{BGPOST}} shows that this is less costly than porting a BGP implementation
over QUIC.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses network byte order (that is, big endian) values.
Fields are placed starting from the high-order bits of each byte.

# Summary of operation

A BGP over TLS/TCP session is established in two phases:

 - establish a transport layer connection using TCP
 - establish a TLS session over the TCP connection

The TCP connection SHOULD be established on port TBD1.

During the establishment of the TLS session, the router that initiates the
connection MUST use the "botls" token in the Application Layer Protocol
Negotiation (ALPN) extension {{RFC7301}}. The support for other ALPN MUST
NOT be proposed during the TLS handshake.

Once the TLS handshake is established and finished, the BGP session is
initiated as defined in {{RFC4271}} and the protocol operates in the
same way as a classic BGP over TCP session. The difference is that the
BGP session is now encrypted and authenticated using the TLS layer.
As in {{I-D.draft-retana-idr-bgp-quic}}, the TLS authentication 
parameters used for this connection are out of the scope of this draft.

# Security Considerations

This document improves the security of BGP sessions since the information 
exchanged over the session is now protected by using TLS.

If TLS encounters a payload injection attack, it will generate an alert that immediately
closes the TLS session. The BGP router SHOULD then attempt to reestablish the session.
However, this will cause traffic to be interrupted during the connection re-establishement.

If both BGP peer supports TCP-AO, the TLS stack is protected against payload injection and
this attack can be avoided. When enabled, TCP-AO counters TCP injection
attacks listed in {{RFC5082}}.

Furthermore, if the BGP router supports TCP-AO, we recommend an opportunistic
TCP-AO approach as suggested in {{I-D.draft-piraux-tcp-ao-tls}}. The
router will attempt to connect using TCP-AO with a default key. When the TLS
handshake is finished, the routers will securely derive a new TCP-AO key from the TLS key.

TCP-MD5 {{RFC2385}} MAY be used to protect the TLS session if TCP-AO is not available on the
BGP router.

# IANA Considerations

IANA is requested to assign a TCP port (TBD1) from the "Service Name and Transport
Protocol Port Number Registry" as follows:

- Service Name: botls
- Port Number: TBD1
- Transport Protocol: TCP
- Description: BGP over TLS/TCP
- Assignee: IETF
- Contact: IDR WG
- Registration Data: TBD
- Reference: this document
- Unauthorized Use Reported: idr@ietf.org

It is suggested to use the same port as the one selected for BGP over QUIC
{{I-D.draft-retana-idr-bgp-quic}}.

# Acknowledgments
{:numbered="false"}

The authors thank Dimitri Safonov for the TCP-AO implementation in Linux.
This work has been partially supported by the Walloon Region as part of the 
funding of the FRFS-WEL-T strategic axis.

# Change log
{:numbered="false"}



