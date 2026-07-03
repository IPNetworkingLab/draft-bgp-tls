---
title: "BGP over TLS/TCP"
abbrev: bgp-tls
docname: draft-wirtgen-bgp-tls-latest
category: std

submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
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
    organization: Unaffiliated
    email: thomas.wirtgen@gmail.com
 -
    name: Olivier Bonaventure
    organization: UCLouvain & WELRI
    email: olivier.bonaventure@uclouvain.be
 -
    name: AravindBabu MahendraBabu
    organization: Cisco Systems
    email: aramahen@cisco.com
 -
    name: Chennakesava Reddy Gaddam
    organization: Cisco Systems
    email: chgaddam@cisco.com

normative:
  RFC2119:
  RFC8174:
  RFC4271:
  RFC5925:
  RFC7301:
  RFC8446:
  I-D.piraux-tcp-ao-tls:
  I-D.hbq-bgp-tls-auth:

informative:
  I-D.retana-idr-bgp-quic:
  RFC4272:
  RFC5082:
  RFC9000:

--- abstract

This document specifies the use of TLS over TCP to support BGP. The
Border Gateway Protocol (BGP) relies on TCP to establish sessions
between routers. While the TCP Authentication Option (TCP-AO) provides
transport-layer integrity protection against spoofing and reset attacks,
it does not provide confidentiality, cryptographic peer identity, or
scalable key management. This document specifies a method for
establishing a secure BGP session by running BGP over a TLS 1.3
session. The underlying TCP transport MUST be protected using TCP-AO.
An "Implicit TLS" model on TCP port 179 is specified as the preferred
mechanism.

--- middle

# Introduction

The Border Gateway Protocol (BGP) {{RFC4271}} relies on TCP to
establish BGP sessions between routers. A recent draft
{{I-D.retana-idr-bgp-quic}} has proposed replacing TCP with the QUIC
protocol {{RFC9000}}. QUIC provides several advantages over TCP,
including security, support for multiple streams, and datagram
transport.

From a security viewpoint, an important benefit of QUIC compared to TCP
is that QUIC, by design, prevents the injection attacks that are
possible when TCP is used by BGP {{RFC4272}}. Several techniques exist
to counter such attacks {{RFC5082}} and {{RFC5925}}.

TCP-AO {{RFC5925}} authenticates the packets exchanged over a BGP
session and enhances transport-layer integrity by protecting TCP
segments against spoofing and reset attacks. However, TCP-AO does not
provide encryption, cryptographic identity, or scalable key management.
TCP-AO SHOULD be used to protect BGP transport traffic.

TLS {{RFC8446}} introduces authenticated peer identities,
confidentiality of routing messages, and cryptographic agility aligned
with modern compliance requirements. The widespread deployment of TLS
creates an interest in using Mutual TLS (mTLS) to secure BGP sessions.
TLS complements TCP-AO: TCP-AO authenticates the entire TCP segment,
covering both the TCP header and payload, to provide integrity and
peer authentication at the transport layer. mTLS additionally encrypts
and authenticates the application data (the TCP payload), and
authenticates both BGP endpoints.

This document describes how to establish a secure BGP session using
mTLS. The underlying TCP transport MUST be protected using TCP-AO
{{RFC5925}} with pre-shared key authentication or deriving TCP-AO
keys from the TLS handshake as described in {{I-D.piraux-tcp-ao-tls}}.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

This document uses network byte order (that is, big-endian). Fields are
placed starting from the high-order bits of each byte.

# Summary of Operation

A BGP over TLS/TCP-AO session is established in two phases:

1. A TCP connection is established on port 179. The integrity of the
TCP segments is protected by using TCP-AO.

2. A TLS session is established over this TCP connection.

With mandatory TCP-AO as the underlying transport protection, TCP port
179 continues to provide authenticated transport establishment. This
avoids the need for a new port while preserving the existing BGP
operational model.

The key benefits of this approach are as follows:

* The existing BGP TCP port 179 transport is reused.

* TCP-AO protects the integrity of the TCP segment.

* Firewall, ACL, and operational deployment changes are avoided.

During the establishment of the TLS session, the router that initiates
the connection MUST use the "botls" token in the Application-Layer
Protocol Negotiation (ALPN) extension {{RFC7301}}. Other ALPN tokens
MUST NOT be included in the TLS handshake.

Once the TLS handshake is complete, the BGP session is initiated as
defined in {{RFC4271}}, and the protocol operates in the same manner as
a classic BGP-over-TCP session, except that the session is encrypted
and authenticated by the TLS layer.

# Transport

## Overview

This document specifies the use of TCP port 179 rather than a dedicated
port. The use of a dedicated port is purely an operational and
deployment choice.

## TLS Authentication for BGP

{{I-D.hbq-bgp-tls-auth}} discusses authentication considerations for
running BGP over TLS protocols and defines a PKI framework to provide
for authenticating BGP peering sessions.

## Initiating the TLS Procedure

This document specifies the Implicit TLS model for BGP session
establishment. Both peers MUST be pre-configured to enable TLS
on port 179. After the TCP-AO connection is established, the
active peer (TLS client) MUST transmit a TLS ClientHello as its
first application-layer bytes; the passive peer (TLS server)
MUST expect a TLS ClientHello before sending any data. BGP
exchanges MUST NOT commence until the TLS handshake has
completed successfully. No plaintext BGP bytes appear on the
wire; use of TLS is assumed entirely by configuration.

~~~
Peer A (active = TLS client)         Peer B (passive = TLS server, :179)
  |--- TCP SYN + AO(MAC) (port 179) ------>|
  |<-- TCP SYN-ACK + AO(MAC) --------------|
  |--- TCP ACK + AO(MAC) ----------------->|
  |                                        |  <- TCP up (TCP-AO), no BGP yet
  |--- TLS ClientHello ------------------->|  <- TLS 1.3 begins
  |<-- ServerHello, {EncryptedExtensions}, |
  |    {CertificateRequest}, {Certificate},|
  |    {CertificateVerify}, {Finished} ----|  <- server's whole flight;
  |                                        |     server is already "done"
  |--- {Certificate}, {CertificateVerify}, |
  |    {Finished} ------------------------>|  <- client authenticates and
  |                                        |     sends the LAST Finished
  |                                        |  <- Encrypted, mutually-
  |                                        |     authenticated channel up
  |--- BGP OPEN (inside TLS) ------------->|  <- BGP exchanges start here
  |<-- BGP OPEN (inside TLS) --------------|
  |--- BGP KEEPALIVE --------------------->|
  |<-- BGP KEEPALIVE ----------------------|
  |                                        |  <- ESTABLISHED
~~~
{: #fig-implicit-tls title="Implicit TLS Establishment on Port 179"}

This model is simple, requires no protocol changes to BGP, and incurs
no negotiation overhead. In this model, both endpoints MUST be
explicitly configured for TLS.

Implementations MUST support TLS 1.3 {{RFC8446}} with mutual TLS
authentication. The connection MUST NOT proceed for any lower TLS
version.

## Connection Establishment Failures

An active BGP peer MUST connect over a TLS-protected TCP-AO
connection and never over plaintext TCP. Any failure within the TLS
layer is abstracted from the BGP Finite State Machine (FSM), which
observes only a generic connection failure event. TLS error alerts
are defined in Section 6.2 of {{RFC8446}}. The active BGP peer should
continue attempting the TLS establishment. After the configured number
of failed attempts, it may proceed based on the local policy decisions
described in [Operational Considerations](#operational-considerations),
using TCP-AO authentication only.

~~~
TCP connected (TCP-AO authenticated)
  -> TLS layer
       -> TLS succeeded? -> BGP socket ready -> send OPEN
       -> TLS failed?    -> TcpConnectionFails
                            -> Local Policy decision
~~~
{: #fig-tls-abstraction title="Abstraction of TLS Failures from the BGP FSM"}

### TLS ClientHello received by a non-TLS peer

A non-TLS peer reads the TLS ClientHello (0x16 0x03 0x01 ...)
as a BGP message header. As per Section 6.1 of {{RFC4271}}, the
16-octet Marker MUST be all 0xFF; because the record begins with 0x16,
the check fails immediately. As per Section 4.5 of {{RFC4271}}, the
BGP connection is closed immediately after the NOTIFICATION message
is sent.

~~~
Peer A (active = TLS client)          Peer B (plain BGP, non-TLS)
  |-- TCP handshake ------------------->|  <- TCP up
  |-- TLS ClientHello ----------------->|
  |   Content Type: 16                  |  Marker byte 0x16 != 0xFF
  |   Version:      03 01               |  -> Connection Not Synchronized
  |<- BGP NOTIFICATION -----------------|
  |<- TCP FIN --------------------------|
~~~
{: #fig-clienthello-malformed title="ClientHello Interpreted as a Malformed BGP Header"}

After receiving TcpConnectionFails (Event 18), Peer A SHOULD terminate
the connection and MAY initiate a new connection attempt based on local
policy.

### TLS Handshake Timeout

If a non-TLS peer accepts the TCP connection but neither sends a BGP
NOTIFICATION message nor resets the connection after receiving a TLS
ClientHello, the TLS-capable peer that initiated the TLS handshake may
eventually experience a TLS connection timeout and transition, via
TcpConnectionFails (Event 18) or ConnectRetryTimer expiry, back to the
Idle state.

## ALPN for BGP over TLS

Application-Layer Protocol Negotiation {{RFC7301}} allows a TLS client
to declare the application protocol it intends to use within the TLS
session. For BGP over TLS, the ALPN identifier is the octet sequence
0x62 0x6F 0x74 0x6C 0x73 ("botls"), defined by this document and
registered as specified in {{iana-considerations}}.

An active BGP peer initiating the TLS exchange MUST include the following
extension in its ClientHello:

~~~
TLS ClientHello extensions:
  application_layer_protocol_negotiation (0x0010):
    ProtocolNameList:
      list length: 00 06
      name length: 05
      "botls":     62 6F 74 6C 73
~~~
{: #fig-alpn-clienthello title="ALPN Extension in ClientHello"}

The passive BGP peer MUST indicate the selected protocol in the
EncryptedExtensions message:

~~~
application_layer_protocol_negotiation (0x0010):
  ProtocolNameList:
    list length: 00 06
    name length: 05
    "botls":     62 6F 74 6C 73
~~~
{: #fig-alpn-encrypted title="ALPN Extension in EncryptedExtensions"}

If the responding BGP peer does not recognize "botls", it MUST send a
fatal TLS alert:

~~~
TLS Alert (fatal):
  Level       : 02   (fatal)
  Description : 78   (no_application_protocol = 120)
~~~
{: #fig-tls-alert title="Fatal Alert on Unrecognized ALPN Identifier"}

ALPN provides an early, explicit signal that the remote endpoint is not
a BGP-over-TLS endpoint, which is preferable to completing the
handshake and failing later on a malformed OPEN message.

# Connection and Session Management

Since BGP operates as a peer-to-peer protocol rather than a strict
client/server protocol, either peer MAY initiate the TCP connection.
The peer that sends the TLS ClientHello is designated the active peer
and acts as the TLS client; the peer that receives it is the passive
peer and acts as the TLS server. Therefore, each BGP speaker
implementing TLS MUST be capable of operating in both roles.

## Connection Collision Detection

When both peers simultaneously initiate a TCP connection, two TCP
connections may form. Section 6.8 of {{RFC4271}} resolves this as
follows:

* The peer with the higher BGP Identifier (Router-ID) retains its
  outgoing connection.

* The peer with the lower BGP Identifier drops its outgoing connection
  and uses the incoming connection.

If a connection collision is detected in the OpenSent or OpenConfirm
state, the procedures defined in Section 6.8 of {{RFC4271}} apply. The
BGP speaker detecting the collision SHALL send a NOTIFICATION message
with the Cease Error Code, terminate the corresponding TCP connection,
and transition the FSM to the Idle state.

Since the BGP Identifier is not known until the BGP OPEN message is
received, both competing TCP/TLS sessions may complete full TLS
establishment prior to collision resolution. Once the BGP Identifier
comparison is performed, the speaker determined to have the
lower-precedence connection, as defined by {{RFC4271}}, MUST terminate
the corresponding TCP/TLS connection.

As a result, the use of TLS does not modify the collision resolution
algorithm itself but may increase the connection establishment cost and
delay associated with collision handling, since one fully established
TCP/TLS session may subsequently be discarded.

~~~
Peer A (lower Router-ID - loses)         Peer B (higher Router-ID - wins)
  |                                         |
  | <- both TCP+TLS connections up <-       |
  | <- both BGP OPENs exchanged    <-       |
  |                                         |
  |  ** Collision: Peer A drops its         |
  |     outgoing connection **              |
  |                                         |
  |--- BGP NOTIFICATION (Cease) ----------->|  (over outgoing conn, encrypted)
  |--- TLS close_notify ------------------->|
  |--- TCP FIN ---------------------------->|
  |                                         |
  |<===== Established on Peer B's conn ====>|
~~~
{: #fig-collision title="Teardown Sequence on the Losing Connection"}

## TLS Session Continuity and Certificate Expiry Handling

If a new TCP/TLS connection is established, full TLS certificate
validation procedures MUST be performed during the new TLS handshake.

Certificate validity and peer authentication are performed during TLS
session establishment. Expiration or revocation of a certificate after
successful establishment of an active TCP/TLS-protected BGP session
SHOULD NOT, by itself, trigger termination of the existing session.

Implementations are not required to perform periodic TLS
re-authentication or certificate revalidation for active BGP sessions.
Existing TLS sessions MAY continue operating until normal BGP or
transport-layer termination occurs.

# Operational Considerations

An implementation MAY support locally configurable transport security
policy modes for BGP over TCP-AO and TLS deployments.

The following operational modes are RECOMMENDED:

tls-required:
: TLS is mandatory. Session establishment MUST fail if TLS negotiation
  or peer validation fails.

tls-preferred:
: TLS establishment SHOULD be attempted. If TLS negotiation or
  validation fails, the implementation MAY continue operation using
  TCP-AO-protected BGP transport based on local policy.

ao-only:
: The BGP session operates using TCP-AO-protected transport without TLS
  establishment.

These modes allow operators to deploy BGP over TLS incrementally while
preserving interoperability with existing TCP-AO-protected deployments.

Implementations supporting the "tls-preferred" mode SHOULD provide
configurable timeout behavior for TLS establishment prior to fallback
to TCP-AO-protected BGP transport.

# Security Considerations

This document improves the security of BGP sessions, as the information
exchanged over the session is protected using TLS. This document
mandates the use of TCP-AO, which protects the TLS stack against
payload injection attacks.

It is RECOMMENDED that an opportunistic TCP-AO approach be used, as
described in {{I-D.piraux-tcp-ao-tls}}. A router attempts to connect
using TCP-AO with a default key; once the TLS handshake completes, it
derives a new TCP-AO key from the TLS key.

# IANA Considerations

## Registration of the BOTLS ALPN Identification String

This document requests registration of a new entry in the "TLS
Application-Layer Protocol Negotiation (ALPN) Protocol IDs" registry.
The "botls" string identifies BGP over TLS:

~~~
Protocol                : BOTLS (BGP over TLS)
Identification Sequence : 0x62 0x6F 0x74 0x6C 0x73 ("botls")
Reference               : This document
~~~
{: #fig-botls-alpn title="BOTLS ALPN Registration"}

> **Editor's Note:** Earlier revisions of this draft requested
> allocation of a dedicated TCP port (TBD1) for "botls" from the
> "Service Name and Transport Protocol Port Number Registry". That
> request has been WITHDRAWN in favor of reusing TCP port 179 with the
> Implicit TLS model described in this document. Accordingly, no new
> transport port is requested.

# Acknowledgments
{:numbered="false"}

The authors thank Dmitry Safonov for the TCP-AO implementation in Linux.

# Contributors
{:numbered="false"}

   Serge Krier
   Cisco Systems
   De Kleetlaan 6a
   1831 Diegem
   Belgium
   Email: sekrier@cisco.com

# Change log
{:numbered="false"}



