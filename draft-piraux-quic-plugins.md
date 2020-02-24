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
      - ins: F. Rochet
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
compiles (JIT) the bytecode of QUIC Plugins and then exectute the native
instructions. The bytecode is a portable reduced instruction set, such
as eBPF or WebAssembly bytecode. The plugin runs thus inside an isolated
environment implemented by a RISC architecture linked to the QUIC implementation.

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

# QUIC Plugins Authenticity

Several possibilities exist under different threat models and offer
different security properties. Two of them are described in the following
sections.

## Central Authorities

This first approach leverages the central authorities commonly
used to secure HTTPS. In this approach, each QUIC Plugin MUST be associated to
some level of trust regarding its origin. A QUIC Plugin MAY be authenticated
using a certificate, itself certified by a central authority.
Consequently, a QUIC implementation supporting QUIC plugins MAY restrict their
exchange and only accept plugins authenticated using the same certificate
used for establishing the QUIC connection.

## Plugin Transparency

This second approach is presented in the {{PQUIC}} research paper, and
suggests going beyond to the restrictive approach of a centralized trust
model.  The centralized trust model obliges the server to update
manually their list of supported plugins. It also prevents the client
from injecting a plugin that the server has not marked to support, e.g.
because the server is unaware of its existence or its list has not been
updated.

The suggested design, called Plugin Transparency, proposes a methodology
to transparently distribute plugins created by independent developers
and verified by freely selected plugin validators. Those validators
endorse verifying some publicly known safety or security property. A
QUIC endpoint can announce a set of conditions to accept a plugin as a
first order logic formula bound to plugin validators. Whenever the other
peer is willing to inject a plugin, it MUST send a (unforgeable) proof
that fulfills the requirements expressed by the logic formula. If the
requirements are met, then the endpoint MAY safely accept the plugin and
MUST update its list of plugin supported. Compared to the Central
authority approach, supported plugins are updated as part of the
protocol design or as a consequence of any change to the default logic
formula bound to plugin acceptance.

# Security Considerations

## Authenticity

The central authority model is ubiquitous to secure modern application
layer protocols such as HTTP. Yet, many flaws exist within the
centralized trust model that, at several occasions, conducted secured
HTTP connections to suffer from the threats which they were expected to
defend, or to sometimes denial access to the content due to poor
resilience of the system to failure (e.g., forgetting certificate
renewal)

Google's Certificte Transparency project {{CT}} is an attempt to address
the structural issues hidden within the central trust assumption and
prevent issue by mistake, rogue certificates and rogue authorities to
weaken the system. Our Plugin Transparency approach bears similarities
to Certificate Transparency. First, our motivations are drawn from the same
conclusions regarding the danger of central trust models. Second, like
{{CT}}, we build over distributing trust assumptions to secure the
system. However, our design offers stronger properties and gets rid of the
requirement to have independant monitoring entities which hold the
resource endowment to continously monitor the {{CT}} log in behalf of
certificate owners. Indeed our design offers the independant developers
to check for spurious plugins in O(log(N)) with N the size of the log
(instead of O(N) in {{CT}}'s design), offers Secure human-readable names
for plugins to the system that unambiguously authentify them and
Non-equivocation from rogue plugin validators. Our design allows to be
more resilient to failure, by offering to optionnaly trust various
validators within the logic formula in case of failure to provide
service. That is, for example, a PQUIC peer may ask to provide a proof
bound to either one of several plugin validators (or any combinaison of
them). The detail of the design, including performance performance
considerations and security proofs are available in {{PQUIC}}.


## Privacy

In the central authority paradigm, there is no Privacy. That is, the
PQUIC server can set arbitrary plugins to the PQUIC stack of any user as
long as they provide a valid signature to them.

In the Plugin Transparency model, Privacy MAY be achieved under careful
treatment. One solution is to remove the notion of supported plugins
within the transport parameters, remove the cache system and use the
default policy to ask for a plugin endorsement by the validators.
Within the default policy, at least one plugin validator MUST be tasked to verify
that the plugin is not leaking distinguishable information to the PQUIC
server, such as an obvious ID or a more subtle fingerprinting extracting
design built-in to the plugin. Moreover, the validator MAY require
open-source availability and public knowledge of the pseudo-identity of its
developer. To avoid leaking information to the nework, the injection of
a set of plugins (while bing encrypted) SHOULD be indistinguishable from
any other set of plugins.


## System Security

We expect the plugin to run within a sandboxed environment with access
control and resource management defined by the host application running
the plugin (the PQUIC implementation in our case), and traps mechanism.
That is, plugins SHOULD be given whitelist permission type to system
resources such as a file descriptor or a directory. The PQUIC
implementation MAY define access policies to all protocol operations. In
opposition to the mobile ecosystem, protocol operations SHOULD NOT ask
for permissions but are given permission as part of a long-lived 
PQUIC version.

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments {:numbered="false"}

TODO acknowledge.
