---
title: "WebTransport over WebSocket"
abbrev: "WebTransport-WS"
category: info

docname: draft-richter-webtransport-websocket-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date: {DATE}
consensus: true
v: 3
# area: art
# workgroup: webtrans
keyword:
 - Internet-Draft
#venue:
#  group: WG
#  type: Working Group
#  mail: WG@example.com
#  arch: https://example.com/WG
#  github: martenrichter/draft-ietf-webtransport-websocket
#  latest: https://example.com/LATEST

author:
 -
    fullname: Marten Richter
    organization: Technische Universität Berlin
    email: marten.richter@tu-berlin.de

normative:
  OVERVIEW: I-D.ietf-webtrans-overview
  WEBTRANSPORT-H3: I-D.ietf-webtrans-http3
  WEBTRANSPORT-H2: I-D.ietf-webtrans-http2
  HTTP: I-D.ietf-httpbis-semantics
  WEBSOCKET: RFC6455
  WEBSOCKET-H2: RFC8441

informative:
  DATAGRAM: RFC9221


--- abstract

WebTransport {{OVERVIEW}}, a protocol framework within the Web security model, empowers Web clients to initiate secure multiplexed transport for low-level client-server interactions with remote servers.
This document outlines a protocol, based on WebSocket {{WEBSOCKET}}, offering WebTransport capabilities similar to the HTTP/2 variant {{WEBTRANSPORT-H2}}. It serves as an alternative when UDP-based protocols are inaccessible, and the client environment exclusively supports WebSocket {{WEBSOCKET}}.
***DISCLAIMER: So far this document was not submitted to any IETF WG. Currently, it just describes the protocol used for WebSocket connections of the @fails-components/webtransport package.***

--- middle

# Introduction

WebTransport {{OVERVIEW}} is designed to facilitate communication for Web clients over HTTP/3 {{?HTTP3=I-D.ietf-quic-http}}, leveraging QUIC {{?QUIC=RFC9000}} semantics with streams or datagrams {{DATAGRAM}}. In cases where UDP-based traffic is restricted, HTTP/2 protocol {{WEBTRANSPORT-H2}} serves as an alternative built solely on HTTP semantics.

Both {{WEBTRANSPORT-H2}} and {{WEBTRANSPORT-H3}} variants require a native WebClient implementation due to the usual unavailability of plain UDP and TCP/IP socket access for scripts within WebClients

This document defines a protocol implementable on the WebClient using available scripting languages, without altering the WebClient's native code.
It uses the widespread WebSocket protocol as the base without modification.
However, a direct implementation in a WebClient is possible.

The protocol utilizes capsule semantics derived from {{WEBTRANSPORT-H2}} and translates them into WebSocket frames. By relying on WebSockets, also intermediates such as proxies
unaware of WebTransports can apply application layer processing.

An implementation should support both WebSocket over http/1 and http/2.
The server should incorporate WebTransport flow control constraints and capsule processing into its WebSocket parser code. Therefore, using unmodified existing WebSocket code is not recommended.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

The document follows the terminology defined in {{Section 1.2 of OVERVIEW}}.

# Protocol Overview

WebTransport servers are identified by an HTTPS URI per {{Section 4.2.2 of HTTP}}.

The protocol uses {{WEBTRANSPORT-H2}} semantics with the following modifications.

## Connection and version negotiation
The WebSocket connection is established according to {{Section 4 of WEBSOCKET}} or {{WEBSOCKET-H2}}.

When a WebSocket connection is established, both the client and server select the WebTransport-Websocket protocol by setting |Sec-WebSocket-Protocol| {{Section 1.9 of WEBSOCKET}} to the supported versions. The protocol names follow the scheme "webtransport_VERSIONAME", where VERSIONNAME identifies the particular protocol version. For this protocol VERSIONAME would be "kDraft1" and the |Sec-WebSocket-Protocol| field would include "webtransport_kDraft1".
The protocol negotiation follows the procedures as described in {{Section 4.1 of WEBSOCKET}} and {{Section 4.2.2 of WEBSOCKET}}.
No protocol extensions MUST BE negotiated.

## Data framing
The protocol uses the data frames as defined in {{Section 5 of WEBSOCKET}}.
PING and PONG frame handling is not changed {{Section 5.5 of WEBSOCKET}}.

The CLOSE frame {{Section 5.5.1 of WEBSOCKET}} replaces the mechanism invoked after CONNECT stream closure in {{Section 2 of WEBTRANSPORT-H2}}.
The body MUST include a UTF-8 encoded reason string when transmitted by a protocol-aware client or server.
The reason string has the form "CODE:REASONSTRING", where CODE is a text representation of an unsigned 32-bit decimal integer in the range of between 0x00000000 and
0xffffffff {{Section 4.3 of WEBTRANSPORT-H3}}. REASONSTRING is the actual reason transmitted
through WebTransport.
A close FRAME without a reason may be sent by protocol-unaware WebClients or proxies. In such instances, the CODE and REASONSTRING are reconstructed using the WebSocket Close Code Number as specified in the WebSocketCloseCode Registry outlined in {{Section 11.7 of WEBSOCKET}}.

Data Frames containing Text are reserved for future use and MUST NOT be sent.
Binary Data Frames transport CAPSULE content defined in {{WEBTRANSPORT-H2}} and {{DATAGRAM}}. For details, refer to the next section {{capsule-frames}}. Their length is limited by WebTransport flow control, and a violation SHOULD lead to connection termination.
CONTINUATION frames are processed per {{WEBSOCKET}} specifications. Given the streaming nature of the content, partial DATA frames or CONTINUATION frames should be promptly forwarded to corresponding streams reducing latency.

## Capsule frames
This protocol adopts the mechanisms and intrinsic elements outlined in {{WEBTRANSPORT-H2}}, which itself is constructed upon the CAPSULE protocol originating from {{DATAGRAM}}.

A CAPSULE has the form in {{DATAGRAM}}:
~~~
Capsule {
  Capsule Type (i),
  Capsule Length (i),
  Capsule Value (..),
}
~~~
where Capsule Type and Length are variable-length integers.
The Capsule Value represents the payload of the capsule, and its semantics are determined by the payload type

In the context of WebTransport over WebSockets, CAPSULEs are substituted by binary DATA FRAMES of WebSockets, following the format:
~~~
WebSocketDataFrameCapsule {
  FrameHeader (..),
  PayloadData (..)
}
~~~
FrameHeader contains the first two bytes of the FRAME, and if present the extended payload length and masking key as defined in {{Section 5.2 of WEBSOCKET}}.
PayloadData is defined as:
~~~
PayloadData {
  Capsule Type (i),
  Capsule Value (..)
}
~~~
with the variable length integer Capsule Type and Capsule Value as in the CAPSULE protocol.

Capsule length can be calculated from the Payload Length as set in {{Section 5.2 of WEBSOCKET}}:
```
  Capsule Length = Payload Length - sizeof(Capsule Type),
```
as no Extension Data is allowed.

## Replacement for SETTINGS

{{Section 3.1 of WEBTRANSPORT-H2}} requires sending an SETTINGS_WEBTRANSPORT_MAX_SESSIONS settings parameter. This is not required here, as the protocol type is negotiated using the
subprotocol mechanism of WebSockets and SETTINGS_WEBTRANSPORT_MAX_SESSIONS equal to 1
is assumed per WebSocket connection(HTTP1)/stream(HTTP2).
Subsections of {{Section 3.4 of WEBTRANSPORT-H2}} require sending initial SETTINGS for
flow control. As SETTINGS are not accessible for the WebSocket protocol using the existing WebSocket interfaces, a replacement is required.

Therefore client and server MUST send the initial flow control values using CAPSULES
immediately before ANY other capsules such as WT_STREAM or DATAGRAM capsules have been sent.

# Security Considerations

The security considerations of {{Section 10 of WEBSOCKET}} also apply here.
The last paragraph of {{Section 8 of WEBTRANSPORT-H2}} is equally applicable to this protocol.

# IANA Considerations

## WebSocket Subprotocol Name Registry

All possible subprotocol names following the format "webtransport_VERSION," where VERSION is an alphanumeric string denoting the subprotocol version of this protocol, are added to the registry as domains for this protocol and its successors.

## WebTransport WebSocket Protocol Version Registry

This specification establishes a new IANA registry for WebTransort Protocol Version names, intended for use with the WebSocket WebTransport Protocol, in alignment with the principles outlined in {{!RFC5226}}.

As part of this registry, IANA manages the following information (similar to {{WEBSOCKET}} versions):

   Version String
      The version string name as part of the subprotocol defined in {{websocket-subprotocol-name-registry}} and {{connection-and-version-negotiation}}.  The value must only include alphanumeric characters.

   Reference
      The RFC requesting a new version number or a draft name with
      version number (see below).

   Status
      Either "Interim" or "Standard".  See below for a description.

  A version string can be either "Interim" or "Standard".

  A "Standard" version string is part of an RFC and identifies a major, stable version of the WebTransport-WebSocket protocol. The "IETF Review" IANA registration policy {{!RFC5226}} applies to "Standard" version string.

  An Internet-Draft documents an "Interim" version string. Internet-Drafts helps implementors to identify and interoperate with the WebTransport-WebSocket protocol,
  as this current draft. The "Expert Review" IANA registration policy {{!RFC5226}} applies to the "Interim" version names. The initial Designated Experts need to be determined.

--- back

# Acknowledgments
{:numbered="false"}

Parts of the text were rephrased using ChatGPT. Portions of this document are based upon a modification of text parts from the underlying standards.
