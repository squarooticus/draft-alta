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

Establishing authenticity of a stream of datagrams in the presence of multiple receivers is naively achieved through the use of asymmetric digital signatures, but at high computational cost. Timed Efficient Stream Loss-Tolerant Authentication (TESLA) instead employs relatively cheap symmetric authentication, achieving asymmetry via time-delayed key disclosure, adding latency to verification and imposing requirements on time synchronization between receivers and the sender to prevent forgery. This document introduces Asymmetric Loss-Tolerant Authentication (ALTA), which employs an acyclic graph of message authentication codes (MACs) transmitted alongside data payloads, with redundancy to enable authentication of all received payloads in the presence of certain patterns of loss, along with regularly paced digital signatures. ALTA requires no time synchronization and enables authentication of payloads as soon as sufficient authentication material has been received.

--- middle

# Introduction

Authenticity of a stream of data may be inexpensively established via symmetric message authentication codes (MACs) using keys pre-shared exclusively between two parties, as the receiver knows it did not originate the data and that only one other party has access to the key. In the presence of multiple receivers, however, this is not possible because all receivers must have access to the same key, giving any one of them the ability to forge messages.

Timed Efficient Stream Loss-Tolerant Authentication (TESLA) {{RFC4082}} solves this problem by delaying the release of keying material to a deadline at which any packets protected by said key that are subsequently received must be discarded by a receiver. While this reintroduces asymmetry between sender and receiver, it requires the sender and each receiver to (loosely) synchronize clocks and imposes authentication latency relative to RTT and to a pre-declared upper bound on skew.

Clock synchronization is not as trivial as it appears: internet-connected hosts often have significant clock skew relative to stratum 0 NTP servers {{timeskew}}, and anyway enterprises serving valuable assets do not regard NTP as a reliable interdomain security protocol. Together with the need to avoid attacks that delay packets required for synchronization, this implies the need for an interactive unicast authenticated clock synchronization protocol, which is complicated by the need to maintain clock synchronization across both the stream publisher and multiple geographically-distributed nodes in a content delivery network (CDN).

This document introduces Asymmetric Loss-Tolerant Authentication (ALTA), which eschews time synchronization for a direct application of digital signatures to an acyclic graph of message authentication codes with redundancy sufficient to tolerate certain patterns of loss. This algorithm is based on research by Golle and Modadugu published in {{STRAUTH}}.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

The author wishes to acknowledge the contributions of his colleague, Jake Holland, whose work with interdomain multicast live video delivery drove the need for a robust solution to the streaming authentication problem, and Eric Rescorla, who introduced the author to the paper describing the loss-tolerant symmetric authentication scheme used as the basis for ALTA.
