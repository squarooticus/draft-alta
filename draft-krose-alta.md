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



--- abstract

Establishing authenticity of a stream of datagrams in the presence of multiple receivers is naively achieved through the use of asymmetric digital signatures, but at high computational cost. Timed Efficient Stream Loss-Tolerant Authentication (TESLA) instead employs relatively cheap symmetric authentication, achieving asymmetry via time-delayed key disclosure, adding latency to verification and imposing requirements on time synchronization between clients and the publisher to prevent injection. This document introduces Asymmetric Loss-Tolerant Authentication (ALTA), which employs a graph of message authentication codes (MACs) transmitted alongside data payloads, with redundancy to enable authentication of all received payloads in the presence of certain patterns of loss, along with regularly paced digital signatures. ALTA requires no time synchronization and enables authentication of payloads as soon as sufficient authentication material has been received.

--- middle

# Introduction

TODO Introduction


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
