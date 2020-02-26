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
    seriesinfo: Proceedings of SIGCOMM'19
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
    seriesinfo: Proc. SIGCOMM 2018
    date: August 2018
  I-D.ietf-quic-transport:
  RFC1122:
  RFC2581:
  RFC3449:
  RFC5690:
  RFC6962:
  RFC7540:
  I-D.fairhurst-quic-ack-scaling:
  I-D.iyengar-quic-delayed-ack:
  I-D.deconinck-multipath-quic:
  I-D.peon-httpbis-h2-priority-one-less:
  QUIC-FEC:
    author:
      - ins: F. Michel
      - ins: Q. De Coninck
      - ins: O. Bonaventure
    title: "QUIC-FEC: Bringing the benefits of Forward Erasure
    Correction to QUIC"
    seriesinfo: IFIP Networking 2019
    date: May 2019
  TCP-Options-BPF:
    author:
      - ins: VH. Tran
      - ins: O. Bonaventure
    title: "Beyond socket options: making the Linux TCP stack truly
    extensible"
    seriesinfo: IFIP Networking 2019
    date: May 2019
  eBPF:
    author:
      - ins: TODO
    title: TODO
    date: TODO
  Reno:
    author:
      - ins: TODO
    title: TODO
    date: TODO
  Westwood:
    author:
      - ins: TODO
    title: TODO
    date: TODO
  WebAssembly:
    author:
      - ins: TODO
    title: TODO
    date: TODO
  ICNP:
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

# Introduction {#intro}

Internet hosts rely on protocols to efficiently exchange
information. Most protocols are designed assuming a layer model.
A protocol provides a service to a user at the above layer. It
interacts with this user through specific commands and responses that
are often represented as an Application Programming Interface (API).
Protocol implementations exchange messages with other implementations
by leveraging the service provided by the underlying layer. For
example, a TCP implementation interacts through the socket API and
exchanges TCP segments with remote hosts through the underlying IP
protocol. Protocol designers usually define transport
and application protocols in two parts:

 - a series of messages that are encoded using a specific format.
 - a Finite State Machine that defines how hosts react to commands
   from the layer above or to the reception of a specific messages.

This model has been used to represent a wide range of Internet
protocols. However, there are some limits to the flexibility of
these protocols as illustrated by recent discussions.

A first example is the transmission of acknowledgments in reliable
transport protocols. There is a trade-off between the feedback provided
by the acknowledgments and the resources (bandwidth and CPU)
required to generate and process them. Various heuristics have
been proposed in TCP to generate these ACKs
{{RFC1122}},{{RFC2581}},{{RFC3449}},{{RFC5690}}. These heuristics are deployed
independently on receivers, but for some of them the senders need to
adapt by at least taking into account the fact that some
acknowledgments were delayed while measuring the round-trip-time.
A similar discussion has started for QUIC. Given the flexibility of
QUIC, researchers have proposed to define an acknowledgment
strategy as a set of parameters that are exchanged in a new QUIC frame
over each connection
{{I-D.fairhurst-quic-ack-scaling}},{{I-D.iyengar-quic-delayed-ack}}. This brings
more flexibility than in TCP where the limited size of the header
made it impossible to exchange such information, but it also affects
the round-trip-time estimation {{I-D.ietf-quic-transport}}.

A second example in the application layer
is the support for stream priorities in HTTP/2 {{RFC7540}}. Since
HTTP/2 provides parallel streams, some application developers have
expressed their need to be able to prioritize some streams over
others. The HTTP/2 protocol defines such priorities, but they are
not widely used and some have proposed to deprecate them
{{I-D.peon-httpbis-h2-priority-one-less}}. In spite of this, a
proposal for stream
prioritization for HTTP/3 already exists {{I-D.kazuho-httpbis-priority}}.
{{LNBIP2020}} evaluates stream scheduling for HTTP/3 and
claims that performances are heavily impacted by the adopted stream
scheduling.

These examples illustrate the difficulty of precisely expressing
complex behaviors in a few parameters that are exchanged inside
packets. In this document, we leverage the recent results in
extending operating system kernels {{eBPF}} or web-browsers {{WebAssembly}}
with virtual machines that execute bytecodes to propose a new
approach to define complex behaviors inside protocols. We first
describe the general architecture of the proposed approach in
{{archi}}. We then provide a few examples showing how such an
approach could be applied to the next version of the QUIC protocol in
{{pquic}}.

# More extensible protocols {#archi}

In a traditional layered protocol model, a protocol implementation
is often represented as a black-box that uses the service provided
by the underlying layer to offer an enhanced through, typically
through an Application Programming Interface (API), to the
upper layer. This is illustrated in {{fig-api}}.

~~~~~~~~~~~~~~~~~~~~~~~~~~~
 Layer N+1
       --- +----------- API -----------+
Management |                           |
Protocol   |                           |
<--------->|  Protocol implementation  |
           |                           |
           +---------------------------+
                 ||         /\
                 \/         ||
-----------------------------------
Layer N-1

~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #fig-api title="A protocol implementation exposes an API to the upper layer"}

On benefit of this approach is that if two implementations expose the
same API, it possible to replace one by the other without changing anything
in the upper layers. In pratice, the API with the upper layer is not
the only API that is exposed by a protocol implementation. For management
and configuration purposes, many protocol implementations also expose
a set of configuration variables that can be accessed and for some of
them be modified by network management protocols such as SNMP or IPFIX.

These management approaches expose abstract configuration parameters and
statistics about the key operations performed by a protocol implementation.
They have been useful in configuring and operating a
wide range of protocol implementations. As different implementations expose
the same abstraction, it becomes possible for operators to configure
and manage different implementations by using the same tool in a unified
manner.

Experience with successful Internet protocols shows that once a protocol
gets (widely) deployed, it attracts new use cases and proposals to
extend and improve it. Most of the key Internet protocols, including IP, TCP,
HTTP, DNS, BGP, OSPF, IS-IS, ... have been improved over the years. Today,
the developer of an implementation of any of these protocols need to
consult dozens of RFCs to find the complete specification of the protocol.

Despite the importance of extensions to those key Internet protocols, we still
do not design them under the assumption that they will evolve over decades and
that their implementations should be made agile. In this document, we propose
a new organisation for protocol implementations. Instead of trying to
pack as many features as possible inside a protocol implementation that
is considered as a blackbox, we consider a protocol implementation as
an open system which can be safely extended to support new features in
a safe and agile manner. Our vision is that such an implementation
exposes an internal API which can be exploited by extensions that we call
Plugins in this document. This is illustrated in {{fig-plugins}}.

~~~~~~~~~~~~~~~~~~~~~~~~~~~
 Layer N+1
       --- +----------- API -----------+
Management |  ( )       ( )       (x)<------ Plugin_A
Protocol   |                           |
<--------->|  Protocol implementation  |
           |       ( )       (x)<--------- Plugin_B
           +---------------------------+
                 ||         /\
                 \/         ||
-----------------------------------


~~~~~~~~~~~~~~~~~~~~~~~~~~~
{: #fig-plugins title="A pluginized protocol implementation exposes an internal API that enables plugins to dynamically extend its operation"}

There are different ways of implementing the idea of dynamically
extending protocols by using plugins. We list here some requirements
that any solution should support:

REQ1:
: Different implementations should expose the same plugin API, e.g. as different
implementations expose the same SNMP MIB.

REQ2:
: It should be possible to dynamically attach a plugin to one instance of an
implementation. For instance, for a protocol that supports connections, it
should be possible to associate different plugins to different connections.

REQ3:
: It should be possible to execute the same plugin on different and
interoperable implementations of the same protocol.

REQ4:
: It should be possible for a protocol implementation to restrict the operations
that a given plugin can execute.

One possible way to realise this new architecture is to include a virtual
machine inside each protocol implementation and
expose a small plugin API accessible through the virtual machine.
Several efficient virtual machines have been proposed and used in related
environments {{eBPF}} {{WebAssembly}}. They provide a sandbox that controls the
operations that the plugin can execute and the memory that it can access.
Since the same virtual machine can be installed on different platforms, it
becomes possible to execute the same plugin on different implementations of
a given protocol that expose the same plugins API.

The idea of extending protocols through plugins can be applied to
different Internet protocols. In this document, we focus adding plugins
to QUIC since it is a recent and flexible protocol that includes
useful security features. A similar approach has been applied to OSPF
and BGP {{ICNP}} and partially to TCP {{TCP-Options-BPF}}.

# Pluginizing QUIC {#pquic}

QUIC is a recently proposed transport protocol {{I-D.ietf-quic-transport}} that
combines the service offered by TCP and TLS. QUIC encrypts all the application
data and most of the packet headers, and thus prevents most of the interferences
from middleboxes. QUIC encode control and application data using a flexible
framing mechanism. In this section, we describe how we combine the approach
proposed in {{archi}} to those features to propose Pluginized QUIC.

We break down a QUIC implementation execution flow into a generic subroutines.
These are specified functions called protocol operations. These protocol
operations implement a given part of the QUIC protocol, for example the
acknowledgment generation or the computation of the round-trip-time. Some are
generic and depend on a parameter, for instance parsing a QUIC frame is a
generic operation that depends on the type of QUIC frame. This version of the
document does not elaborate exhaustively on the protocol operations that compose
a Pluginized QUIC implementation. The next versions of this document will
work on defining a set of protocol operations.

# What is a QUIC plugin ?
TODO(mp): Change the section title, I don't like it

A QUIC Plugin consists of platform-independent bytecode which modify or extend
the behavior of a QUIC implementation. Adding the functionality of a QUIC Plugin
consists in adding or replacing a set of protocol operations implemented by this
bytecode. This action of adding a QUIC plugin to a QUIC connection is referred
to as injecting a QUIC Plugin. Injecting a plugin is limited to a given
QUIC connection.

Its bytecode is run inside a sandboxed execution environment. It accesses to the
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


For other applications over QUIC with a broad range of requirements, a flexible
approach for defining the stream scheduling policy is key to best fit their
needs. QUIC Plugin offer a flexible way to embed application knowledge inside
the QUIC implementation.

## Pluggable congestion controller

There exists many congestion control algorithms. Each of them has
been designed for a given context, i.e. a range of applications and a range of
Internet paths. For instance, Reno {{Reno}} has been designed for optimizing the
web use-case on common Internet paths. Westwood {{Westwood}} is a modification of
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
traffic, such as a real-time video streaming application, a QUIC Plugin allows
embedding application knowledge, i.e. the characteristics of such bursts, inside
the acknowledgment generation policy. Exchanging and injecting this plugin
allows controlling the other peer behavior.

# Injecting/accepting QUIC Plugins

Injecting a QUIC Plugin to a QUIC implementation requires several
modifications. First, an execution environment is required to execute the
plugin, as it does not consist of executable machine code. This environment
recompiles the bytecode of QUIC Plugins and then execute the native
instructions. The bytecode is a portable reduced instruction set, such
as eBPF or WebAssembly bytecode. The plugin runs thus inside an isolated
environment inside the QUIC implementation.

The QUIC implementation is responsible for interacting with the plugin, i.e.
running its bytecode and providing restricted access to the QUIC connection
state. For example, a QUIC implementation that accepts plugins deciding whether
an acknowledgment has to be sent should execute the plugin every time this
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
to the data transfer, using a new dedicated stream type akin to the crypto
stream.

# QUIC Plugins Authenticity

Several possibilities exist under different threat models and offer
different security properties. Two of them are described in the following
sections.

## Based on Central Authorities

This first approach leverages the central authorities commonly
used to secure HTTPS. In this approach, each QUIC Plugin MUST be associated to
some level of trust regarding its origin. A QUIC Plugin MAY be authenticated
using a certificate, itself certified by a central authority.
Consequently, a QUIC implementation supporting QUIC plugins MAY restrict their
exchange and only accept plugins authenticated using the same certificate
used for establishing the QUIC connection.

## Based on Plugin Transparency

This second approach is presented in the {{PQUIC}} research paper, and
suggests going beyond the restrictive approach of a centralized trust
model. The centralized trust model obliges the server to update
manually their list of supported plugins. It also prevents the client
from injecting a plugin that the server has not marked as supported, e.g.
because the server is unaware of its existence, or because its list has not been
updated.

The suggested design, called Plugin Transparency, proposes a methodology
to transparently distribute plugins created by independent developers
and verified by freely selected plugin validators. Those validators
endorse verifying some publicly known safety or security properties. A
QUIC endpoint can announce a set of conditions to accept a plugin as a
first order logic formula bound to plugin validators. Whenever the other
peer is willing to inject a plugin, it MUST send a (unforgeable) proof
fulfilling the requirements expressed by this logic formula. If the
requirements are met, then the endpoint MAY safely accept the plugin and
MUST update its list of plugin supported. Compared to the central
authority approach, supported plugins are updated as part of the
protocol design or as a consequence of any change to the default logic
formula bound to plugin acceptance.

# Security Considerations

## Central Authorities VS Plugin Transparency

The central authority model is ubiquitous to secure modern application layer
protocols such as HTTPS. Yet, flaws such as forged certificates exist within the
centralized trust model that, at several occasions, conducted HTTPS connections
to suffer from the threats which they were expected to defend. Moreover, the
poor resilience of the system can cause a denial of access to content, for
instance when an expired certificate is not renewed.

Certificate Transparency {{RFC6962}} is an attempt to address the structural
issues hidden within the central trust assumption and prevent mistakes, rogue
certificates and rogue authorities from weakening the system. Plugin
Transparency bears similarities to Certificate Transparency (CT). First, its
motivations are drawn from the same conclusions regarding the danger of central
trust models. Second, similar to CT, it is based on distributing trust
assumptions to secure the system. However, Plugin Transparency offers stronger
properties and eliminates the independent monitoring entities which hold the
resource endowment to continuously monitor the CT log in behalf of certificate
owners. Indeed, our design offers the independent developers checking for
spurious plugins in O(log(N)) with N the size of the log (instead of O(N) in
CT's design). Our design also offers secure human-readable plugins names that
unambiguously authenticate them and non-equivocation from rogue plugin
validators. Our design is more resilient to failure by offering several
validators that can be trusted within the logic formula. For example, a PQUIC
peer may request a proof bound to any combination of plugin validators. The
detail of Plugin Transparency, including performance considerations and security
proofs are available in {{PQUIC}}.

## Privacy

In the central authority paradigm, there is no privacy. That is, the PQUIC
server can set arbitrary plugins to the PQUIC client of any user as long as they
provide a valid signature to them.

In the Plugin Transparency model, privacy may be achieved under careful
treatment. One solution is to remove the list of supported plugins from the
transport parameters, to remove the cache system and use the default policy to
ask for a plugin endorsement by the validators. Within the default policy, at
least one plugin validator MUST be tasked to verify that the plugin is not
leaking distinguishable information to the PQUIC server, such as an obvious ID
or a more subtle fingerprinting mechanism built-in to the plugin. Moreover, the
validator MAY require source code availability and public knowledge of the
pseudo-identity of its developer. To avoid leaking information to the network,
the injection of a set of plugins (while being encrypted) SHOULD be
indistinguishable from any other set of plugins.

One other solution to have privacy while supporting the cache system and 0-RTT
injection of plugins is to advertize a set of plugins common to most PQUIC
users. One method to achieve it would be to bind PQUIC users to a special plugin
validator which counts at each epoch the number of PQUIC user reporting to have
the plugin in its cache. When a sufficient number of users have it, the plugin
validator adds this plugin to its Merkle Tree, which would allow PQUIC endpoints
to inject it to their peers. Similar to the previous solution, injecting a set
of plugins SHOULD be indistinguishable from any other set of plugins to an
on-path attacker.

## System Security

We expect the plugin to run within a sandboxed environment with access control
and resource management defined by the QUIC implementation running the plugin,
and traps mechanism. The application using QUIC MUST define whitelist policies
for plugins to access the system resources such as a file descriptor or a
directory. Only the application is able to modify its policies.

The user of the application can also set such policies, then the resulting
access authorization depends on the intersection of both set of policies.

# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
