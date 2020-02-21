---
title: "QUIC Plugins"
abbrev: "QUIC Plugins"
docname: draft-piraux-quic-plugins-00
category: exp

ipr: trust200902
area: General
workgroup: QUIC Working Group
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: "M. Piraux"
    name: "Maxime Piraux"
    organization: "UCLouvain"
    role: editor
    email: maxime.piraux@uclouvain.be

 -
    ins: "Q. De Coninck"
    name: "Quentin De Coninck"
    organization: "UCLouvain"
    email: quentin.deconinck@uclouvain.be

 -
    ins: "F. Michel"
    name: "Francois Michel"
    organization: "UCLouvain"
    email: francois.michel@uclouvain.be

 -
    ins: "F. Rochet"
    name: "Florentin Rochet"
    organization: "UCLouvain"
    email: florentin.rochet@uclouvain.be

 -
    ins: "O. Bonaventure"
    name: "Olivier Bonaventure"
    organization: "UCLouvain"
    email: olivier.bonaventure@uclouvain.be

normative:
  RFC2119:

informative:
  PQUIC:
    author:
      - ins: Q. De Coninck
      - ins: F. Michel
      - ins: M. Piraux
      - ins: T. Given-Wilson
      - ins: A. Legay
      - ins: O. Pereira
      - ins: O. Bonaventure
    title: Pluginizing QUIC
    seriesinfo: Proceedings of the ACM Special Interest Group on Data Communication
    date: August 2019
  I-D.kazuho-httpbis-priority:
  LNBIP2020:
    author:
      - ins: R. Marx
      - ins: T. De Decker
      - ins: P. Quax
      - ins: W. Lamotte
    title: Resource Multiplexing and Prioritization in HTTP/2 over TCP versus HTTP/3 over QUIC
    date: 2020
  CCP:
    author:
      - ins: A. Narayan
      - ins: F. Cangialosi
      - ins: D. Raghavan
      - ins: P. Goyal
      - ins: S. Narayana
      - ins: R. Mittal
      - ins: M. Alizadeh
      - ins: H. Balakrishnan
    title: Restructuring Endpoint Congestion Control
    date: August 2018
  I-D.ietf-quic-transport:
  RFC3449:
  RFC5690:
  I-D.fairhurst-quic-ack-scaling:
  I-D.iyengar-quic-delayed-ack:
  I-D.deconinck-multipath-quic:
  QUIC-FEC:
    author:
      - ins: F. Michel
      - ins: Q. De Coninck
      - ins: O. Bonaventure
    title: "QUIC-FEC: Bringing the benefits of Forward Erasure Correction to QUIC"
    date: May 2019
  TCP-Options-BPF:
    author:
      - ins: VH. Tran
      - ins: O. Bonaventure
    title: "Beyond socket options: making the Linux TCP stack truly extensible"
    date: May 2019
  TODO:
    author:
      - ins: TODO
    title: TODO
    date: TODO


--- abstract

This document proposes a new mechanism for extending the QUIC protocol
dynamically. QUIC Plugins allows modifying the behavior of a QUIC peer on a
per-connection basis. These plugins run inside a sandboxed environment
monitoring their executions. They can be combined to add their functionalities
to a given QUIC connection.

This document is a straw-man proposal. It aims at sparking discussions on the
proposed approach. TODO(mp): Encourage a place (ml ?) to discuss the approach

--- middle

# Introduction

TODO(mp) Introduction

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# What is a plugin ?
TODO(mp): Change the section title, I don't like it

A QUIC Plugin consists of platform-independent bytecode which modify or extend the
behavior of a QUIC implementation. This document consider the behavior of a QUIC
implementation as a set of functions, also named protocol operations.
Adding the functionality of a QUIC Plugin, i.e. adding or replacing a set of
protocol operations, to a QUIC connection is refered to as injecting a QUIC
Plugin. Injecting a plugin is limited to a given connection.

Its bytecode is run inside a sandboxed execution environment. It access to the
state of a QUIC connection through a restricted API.

The scope of a QUIC Plugin is restricted by both the limitations of this
execution environment, e.g. in terms of instruction set and quantity, and by the
surface exposed by this API, e.g. the quantity of state that can be read from or
written to by a plugin. The API also defines a set of protocol operations to
which QUIC Plugins can be injected. For instance, a QUIC implementation might
restrict QUIC Plugins injection to its acknowledgment generation policy, e.g.
the protocol operation deciding whether sending an ACK frame is needed.

A QUIC Plugin can be injected by several means. The application can inject
plugins to tune its underlying QUIC connection. QUIC peers can exchange and
inject plugins over a QUIC connection, as described in {{exchanging-plugins}}.
Users can set a default configuration to inject plugins on their devices.

# Examples of use-cases

There exist several cases in which being able to modify the behavior of a QUIC
peer is beneficial. {{PQUIC}} demonstrates how this approach can be used to
deploy existing QUIC extensions such as Multipath QUIC
{{I-D.deconinck-multipath-quic}} and {{QUIC-FEC}}. This first version of the
document focus on enabling the extension of simpler QUIC mechanisms. Three of
them are documented in this section. Contributions regarding new uses-cases are
welcomed.

## Application-driven stream scheduling

QUIC streams allow the application to multiplex several bytestreams over a
single QUIC connection. Yet, the QUIC specification does not provide a mechanism
for exchanging prioritization information nor for indicating the relative
priority of streams. As described in Section 2.3 of {{I-D.ietf-quic-transport}},
A QUIC implementation SHOULD provide ways in which an application can indicate
the relative priority of streams. A QUIC implementation could allow QUIC
Plugins to extend or override its stream scheduler.

A proposal for stream prioritisation for HTTP/3 already exists
{{I-D.kazuho-httpbis-priority}}. Instead of defining a communication interface
between the application and the QUIC implementation for indicating the priority
and scheduling of QUIC streams, a QUIC Plugin could directly implement the
required behavior. {{LNBIP2020}} evaluates stream scheduling for HTTP/3 and
claims that performances are heavily impacted by the adopted stream scheduling.

For other applications over QUIC with a broad range of requirements, a flexible
approach for defining the stream scheduling policy is key to best fit their
needs. QUIC Plugin offer a flexible way to embed application knowledge inside
the QUIC implementation.

## Pluggable congestion controller

There exists many congestion control algorithms. Each of them has
been designed for a given context, i.e. a range of applications and a range of
Internet paths. For instance, Reno {{TODO}} has been designed for optimizing the
web use-case on common Internet paths. Westwood {{TODO}} is a modification of
Reno to better accommodate Internet paths with a high bandwidth-delay product,
such as satellite links.

Efforts to restructure congestion controllers within a common framework have
been presented in past works such as {{CCP}}. Such a framework eases the
development and maintenance of those algorithms. It also enables rapid
prototyping and A/B testing.

{{TCP-Options-BPF}} proposes a new TCP Option, leveraging the TCP-BPF
framework, to negotiate the congestion controller to use.
The QUIC specification does not specify a similar mechanism. Negotiating the
congestion controller used allows one endpoint to tune the other, provided that
it implements the algorithm. QUIC Plugins could allow the application to directly
plug the required congestion controller and to exchange it with the other peer.
This flexibility allows the application to choose the best congestion controller
for its requirements.

## Tunable acknowledgments policy

The large diversity of Internet paths also counts networks involving a
significant path asymmetry. Those paths often impact the ability of the receiver
to provide feedback to the sender. The performance of transport protocols such
as TCP has been studied in depth and reported in {{RFC3449}}. A method for
controlling the rate of receiver feedback of TCP, i.e. the rate of ACKs, has
been proposed in {{RFC5690}}.

Proposals for controlling the acknowledgment policy of QUIC already exist.
{{I-D.fairhurst-quic-ack-scaling}} proposes a change to the default policy for
asymmetric paths. {{I-D.iyengar-quic-delayed-ack}} describes a QUIC extension
enabling a QUIC endpoint to control the acknowledgment policy of its peer. For
that purpose, they model the acknowledgment policy depending on a few parameters
and introduce a new QUIC frame to signal new values for those parameters.

A QUIC Plugin could allow implementing a new acknowledgment with a fine
granularity. For example, in the case of an application that generates bursty
trafic, such as a real-time video streaming application, a QUIC Plugin allows
to embed application knowledge, i.e. the characteristics of such bursts, inside
the acknowledgment generation policy. Exchanging and injecting this plugin
allows controlling the other peer behavior.

# Injecting/accepting QUIC Plugins

Injecting a QUIC Plugin to a QUIC implementation requires several
modifications. First, an execution environment is required to execute the
plugin, as it is not consisting of executable machine code. This environment
executes the bytecode of QUIC Plugins. This bytecode is portable and has a
limited instruction set, such as eBPF or WebAssembly bytecode. The plugin runs
thus in an isolated environment inside the QUIC implementation.

The QUIC implementation is responsible for interacting with the plugin, i.e.
running its bytecode and providing restricted access to the QUIC connection
state. For example, a QUIC implementation that accepts plugins deciding whether
an acknowledgment has to be sent should execute the plugin everytime this
decision is considered and provide access to relevant state for computing this
decision. A QUIC implementation that accepts plugins implementing a congestion
controller may provide write access to some state, for example the congestion
window of a given path, so that the plugin can modify the state of the
QUIC connection.

# Exchanging QUIC Plugins {#exchanging-plugins}

Injecting QUIC Plugins locally in the underlying QUIC implementation allows the
application to tune it to its needs. But some use-cases requires adapting the
peer behavior. In those cases, being able to exchange plugins helps to fill
that gap.

In order to negotiate the use of QUIC Plugins, new transport parameters could be
used to announce the plugins that should be injected. Each QUIC peer announces
the plugins it supports and the plugins that it would like to inject to the
other peer. A local policy could restrict the type of plugins that can be
exchanged and injected. Once the QUIC handshake completes and the QUIC Plugins
transport parameters have been exchanged, plugins exchange can take place next
to the data transfer, using a new dedidcated stream type akin to the crypto
stream.

# Trusting QUIC Plugins

Each QUIC Plugin MUST be associated to some level of trust regarding its origin.
For instance, a QUIC Plugin MAY be authenticated using a certificate. A QUIC
implementation accepting plugins MAY restrict the exchange of QUIC Plugins and
only accept plugins authenticated using the same certificate used for
establishing the QUIC connection.

A QUIC endpoint MAY forbid the injection of received QUIC Plugins, while still
being able to send QUIC Plugins to its peer.

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
