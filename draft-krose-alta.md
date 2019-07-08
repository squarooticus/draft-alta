---
title: Asymmetric Loss-Tolerant Authentication
abbrev: ALTA
docname: draft-krose-alta-latest
category: exp

# ipr: trust200902
# area: General
# workgroup: TODO Working Group
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: "K. Rose"
    name: "Kyle Rose"
    organization: "Akamai Technologies, Inc."
    email: krose@krose.org

normative:

informative:
    RFC2119:
    RFC4082:
    timeskew:
        title: Need a reference for how bad time sync is
    STRAUTH:
        target: https://crypto.stanford.edu/~pgolle/papers/auth.pdf
        title: Authenticating Streamed Data in the Presence of Random Packet Loss
        author:
            name: Philippe Golle
        author:
            name: Nagendra Modadugu
        date: 2001
        annotation: ISOC Network and Distributed System Security Symposium

--- abstract

Establishing authenticity of a stream of datagrams in the presence of multiple receivers is naively achieved through the use of per-packet asymmetric digital signatures, but at high computational cost for both senders and receivers. Timed Efficient Stream Loss-Tolerant Authentication (TESLA) instead employs relatively cheap symmetric authentication, achieving asymmetry via time-delayed key disclosure, while adding latency to verification and imposing requirements on time synchronization between receivers and the sender to prevent forgery. This document introduces Asymmetric Loss-Tolerant Authentication (ALTA), which employs an acyclic graph of message authentication codes (MACs) transmitted alongside data payloads, with redundancy to enable authentication of all received payloads in the presence of certain patterns of loss, along with regularly paced digital signatures. ALTA requires no time synchronization and enables authentication of payloads as soon as sufficient authentication material has been received.

--- middle

# Introduction

Authenticity of streaming data may be inexpensively established via symmetric message authentication codes (MACs) using keys pre-shared exclusively between two parties, as the receiver knows it did not originate the data and that only one other party has access to the key. In the presence of multiple receivers, however, this is not possible because all receivers must have access to the same key, giving any one of them the ability to forge messages. Consequently, authentication must be made asymmetric, such that only the sender has the ability to produce messages that correct receivers will verify as authentic.

Naively, a sender may sign individual datagrams using an asymmetric digital signature algorithm, such as RSA or Ed25519, but this carries high computational cost for both the sender and receivers. In the case of streaming video delivery, while the sender's computational load may be dominated by CPU-intensive video encoding, the receiver is often a device with hardware dedicated to efficient video decoding and with limited general purpose computing hardware and/or battery available for high-rate digital signature authentication.

Timed Efficient Stream Loss-Tolerant Authentication (TESLA) {{RFC4082}} addresses this problem through the use of symmetric authentication by delaying the release of keying material to a deadline at which any packets protected by said key that are subsequently received must be discarded by a receiver. While this reintroduces asymmetry between sender and receiver, it requires the sender and each receiver to (loosely) synchronize clocks and imposes authentication latency relative to RTT and to a pre-declared upper bound on clock skew.

Clock synchronization is not as trivial as it appears: internet-connected hosts often have significant clock skew relative to stratum 0 NTP servers {{timeskew}}, and anyway enterprises serving valuable assets do not regard NTP as a reliable interdomain security protocol. Together with the need to avoid attacks that delay packets required for synchronization, this implies the need for an interactive unicast authenticated clock synchronization protocol, which is complicated by the need to maintain clock synchronization across both the stream publisher and multiple geographically-distributed nodes in a content delivery network (CDN).

This document introduces Asymmetric Loss-Tolerant Authentication (ALTA), which eschews time synchronization for an application of digital signatures to an acyclic graph of symmetric message authentication codes with redundancy sufficient to tolerate certain patterns of loss, and with digital signature authentication load greatly reduced relative to the naive approach. This algorithm is based on research by Golle and Modadugu, as published in {{STRAUTH}}.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# Protocol Overview

ALTA is intended for streaming datagram use cases in which the receiving application has a deadline for the utility of received data and can tolerate a degree of random packet loss. It combines a segment of application data with a variable-length authentication tag into an ALTA payload to be sent as a unit in a single datagram, with the authentication tags constructed in such a way that a receiver will be able to authenticate nearly all such ALTA payloads received by the deadline under certain patterns of random packet loss.

An authentication tag is a combination of zero or more symmetric message authentication codes (MACs) and either zero or one digital signature. Each MAC is of another ALTA payload in the stream, while the digital signature is of the containing ALTA payload with the signature field itself replaced by all zeroes.

The MACs included in a given authentication tag are determined by a scheme, as defined in section 3 of {{STRAUTH}}. Conceptually, a scheme is a mostly backward-looking directed acyclic graph of ALTA payloads such that the MAC of a given payload is contained in two or more other payloads in the stream, enabling the loss of one of these to be tolerated without losing the ability to authenticate the given payload.

For purposes of illustration, a simple example scheme is one in which the ith ALTA payload's authentication tag contains MACs for the (i-1)th and (i-2)th payload:

~~~ figure
                                              .
                                              .
                                              .
+-------------+------+                        |
| payload i+1 | MACs |                        |
|             |      |                        |
|             |   i <---------------------+   |
|             | i-1 <-----------------+   |   |
+---------+---+------+                |   |   |
          |                    +----------+---------+
          +------------------->| MAC of payload i+1 |
                               +--------------------+
+-------------+------+                |   |
| payload i   | MACs |                |   |
|             |      |                |   |
|             | i-1 <-----------------+   |
|             | i-2 <-------------+   |   |
+---------+---+------+            |   |   |
          |                    +--------+---------+
          +------------------->| MAC of payload i |
                               +------------------+
+-------------+------+            |   |
| payload i-1 | MACs |            |   |
|             |      |            |   |
|             | i-2 <-------------+   |
|             | i-3 <--------+    |   |
+---------+---+------+       |    |   |
          |                  | +----+-+-------------+
          +------------------->| MAC of payload i-1 |
                             | +--------------------+
+-------------+------+       |    |
| payload i-2 | MACs |       |    |
|             |      |       |    |
|             | i-3 <--------+    |
|             | i-4 <----+   |    |
+---------+---+------+   |   |    |
          |              |   | ++-+-----------------+
          +------------------->| MAC of payload i-2 |
                         |   | +--------------------+
                         |   |
                         |   |
                         .   .
                         .   .
                         .   .
~~~

The recommended scheme is more complex and will be covered in detail in the section on scheme construction.

Encoding a scheme relies on ALTA payloads being addressable deterministically by an index even in the presence of reordering or loss. This index may be provided by a sequence number in the application data or by a (truncated but fully-resolvable) payload sequence number added at ALTA payload construction time. Two modes are supported:

* If the index increments by exactly one for each payload in the stream, and if the scheme is known to both sender and receiver, then indices are not required to be encoded for each MAC in an authentication tag as they can be deduced from a given payload's index and from the DAG associated with the scheme. Hereafter this is referred to as *implicit index mode*.
* If the index increments unpredictably, or if the scheme is not known to the receiver, then each MAC in an authentication tag must be paired with the explicit index of the ALTA payload from which the MAC is computed. For compactness, this index will be encoded as an offset relative to the index of the containing payload. Hereafter this is referred to as *explicit index mode*.

Authenticity of a payload is established by a chain of MACs rooted in an ALTA payload whose authentication tag contains a digital signature created by a key in which trust has been established out-of-band. Delivery of application data must be delayed until a payload has been authenticated. Note that a given payload may be authenticated by a digital signature as well as by one or more MAC chains; within authentication deadline constraints, receivers should prefer to authenticate by MAC, minimizing the computational load imposed by digital signature authentication.

The variable length of authentication tags in ALTA has implications for application data segmentation when constant-length datagrams are desired (e.g., to maximize data per UDP packet with a given path MTU while avoiding fragmentation).

# Protocol Details

## ALTA Payload

### Explicit Index Form

An ALTA payload with explicit indices comprises the following elements (defined below) concatenated in-order:

* A header octet
* An optional payload index
* A sequence of (index, MAC) pairs
* An optional digital signature
* Application data

~~~ diagram
 0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+
|MACct|S| rsvd  |
+-+-+-+-+-+-+-+-+
~~~
{: #header-octet title="Header Octet"}

The first octet of the payload contains the count of MACs included in the payload (`MACct`) as well as a flag `S` indicating whether the payload also contains a digital signature. It also contains four reserved bits which MUST be set to 0 by senders and ignored by receivers.

~~~ diagram
 0                   1
 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-
|     index     | MAC...
+-+-+-+-+-+-+-+-+-+-+-+-
~~~
{: #explicit-index-mac title="MAC with explicit index"}

Each MAC encoded in the payload comprises a single-octet index, expressed as an offset in two's complement, followed by the digest, which is of a constant length specified by out-of-band metadata.

### Index

### MAC Computation

### Application Data

## Authentication Tag

### Chained MACs

### Digital Signature

## Scheme Construction

# Security Considerations

TODO Security

# Operational Considerations

As ALTA requires an out-of-band channel for provisioning of metadata, including digital signature keys and cryptographic algorithms, versioning of the protocol to support a future ALTA revision may be performed there and acted upon by the application.

# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

The author wishes to acknowledge the contributions of his colleague, Jake Holland, whose work with interdomain multicast live video delivery drove the need for a robust solution to the streaming authentication problem, and Eric Rescorla, who introduced the author to the paper describing the loss-tolerant symmetric authentication scheme used as the basis for ALTA.
