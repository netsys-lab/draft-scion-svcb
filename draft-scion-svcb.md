---
title: "The scion SVCB Service Parameter and SCION-Aware Happy Eyeballs"
abbrev: "SCION SVCB"
category: 
docname: 
submissiontype: 
ipr: 
area: "Internet"
workgroup: 
keyword:
  - SVCB
  - SCION
  - Happy Eyeballs
  - path-aware networking
author:
  - fullname: Jelte van Bommel
    initials: J.
    surname: van Bommel
    organization: ETH Zurich
    email: jelte.vanbommel@inf.ethz.ch
  - fullname: Tillman Zaeschke
    initials: T.
    surname: Zaeschke
    organization: ETH Zurich
    email: tillman.zaeschke@inf.ethz.ch
  - fullname: Tony John
    initials: T.
    surname: John
    organization: OVGU Magdeburg
    email: tony.john@ovgu.de
normative:
  RFC2119:
  RFC8174:
  RFC9460:
informative:
  RFC8305:
  RFC9540:
  I-D.ietf-happy-happyeyeballs-v3:
  I-D.dekater-scion-dataplane:
  I-D.dekater-scion-controlplane:

--- abstract

This document defines the "scion" SvcParamKey for the SVCB and HTTPS DNS
resource record types {{RFC9460}}. The parameter conveys that a service
endpoint is additionally reachable over the SCION path-aware
internetworking architecture, and carries the SCION addresses at which
it is reachable. It further specifies how a client implementing Happy
Eyeballs Version 3 {{I-D.ietf-happy-happyeyeballs-v3}} incorporates
SCION-reachable endpoints — including multiple concurrently raced SCION
paths — into its candidate connection attempts alongside IPv6 and IPv4,
such that clients without SCION connectivity are entirely unaffected.

--- middle

# Introduction

SCION {{I-D.dekater-scion-dataplane}} {{I-D.dekater-scion-controlplane}}
is a path-aware internetworking architecture in which endpoints learn
multiple inter-domain paths to a destination and select among them.
Hosts deploying SCION are, in practice, dual-connected: they retain
ordinary IPv4/IPv6 connectivity while additionally being reachable over
SCION.

Today there is no standardized way for such a host to advertise its
SCION reachability in the DNS. Deployed practice uses a freeform TXT
record convention of the form "scion=ISD-AS,host" (see
{{txt-coexistence}}), which cannot participate in service binding:
it carries no association with ALPN protocols, ports, or the
SvcPriority machinery of {{RFC9460}}, and it is invisible to the
resolution phase of Happy Eyeballs Version 3
{{I-D.ietf-happy-happyeyeballs-v3}}, which is driven by SVCB/HTTPS
queries.

Meanwhile, Happy Eyeballs Version 3 (HEv3) defines its candidate
sorting and racing exclusively over IPv4 and IPv6 and provides no
extension point for additional network-layer protocols. Its two
accommodating surfaces are (a) the SVCB SvcParamKey registry, which is
the designated extensibility mechanism of the service binding
framework, and (b) the deliberately implementation-defined notion of
connection-attempt success (Section 6.1 of
{{I-D.ietf-happy-happyeyeballs-v3}}).

This document uses exactly those two surfaces:

1. It defines the "scion" SvcParamKey ({{scion-svcparam}}), modeled on
   the "ipv4hint"/"ipv6hint" parameters, carrying one or more SCION
   addresses for the service endpoint.

2. It specifies SCION-aware candidate construction, sorting, and racing
   for HEv3 clients ({{hev3}}), treating SCION as a third address
   family and expanding each SCION endpoint into a bounded set of
   per-path connection candidates.

A client that does not implement SCION ignores the parameter (per the
default SvcParam handling rules of {{RFC9460}}) and behaves exactly as
an unmodified HEv3 client.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all
capitals, as shown here.

ISD:
: Isolation Domain, the top-level grouping of SCION autonomous systems,
  identified by a 16-bit number.

ASN:
: A SCION AS number, a 48-bit number. Its canonical text form is either
  a decimal number (for values in the BGP-compatible range) or three
  colon-separated groups of up to four hexadecimal digits (e.g.
  "2:0:4a"); the "::" zero-compression of IPv6 is not used.

SCION address:
: The tuple of ISD, ASN, and a host address, written
  "ISD-ASN,host" (e.g. "71-2:0:4a,10.44.25.3" or
  "1-ff00:0:110,2001:db8::1").

Native SCION stack:
: A host configuration in which applications can open SCION
  connections directly (a SCION daemon and underlay connectivity are
  available), as opposed to reaching SCION through an IP-to-SCION
  translation gateway.

# The "scion" SvcParamKey {#scion-svcparam}

The "scion" SvcParamKey conveys that the service endpoint described by
a ServiceMode SVCB or HTTPS record is additionally reachable over
SCION, and enumerates the SCION addresses at which the endpoint is
reachable.

## Wire Format

The SvcParamValue is a non-empty sequence of one or more fixed-length
24-octet SCION address blocks:

~~~
+--------------------+------------------------+
| Field              | Length                 |
+--------------------+------------------------+
| ISD                | 2 octets, network order|
| ASN                | 6 octets, network order|
| Host address       | 16 octets              |
+--------------------+------------------------+
~~~

The host address field always contains an IPv6 address. An IPv4 host
address is carried as an IPv4-mapped IPv6 address (::ffff:a.b.c.d).

A SvcParamValue whose length is zero or is not a multiple of 24 octets
renders the RR malformed and it MUST be entirely ignored.

## Presentation Format {#presentation-format}

The presentation value is a comma-separated list ({{RFC9460}},
Appendix A.1) of SCION addresses in their canonical text form. Because
the SCION address text form itself contains a comma separating the
ISD-ASN from the host address, that inner comma MUST be escaped, in
the same manner that "alpn" values escape embedded commas:

~~~
example.com. 300 IN SVCB 1 . alpn=h3 port=443 (
                           scion=71-2:0:4a\,10.44.25.3 )
~~~

Zone-file implementations MUST accept both the decimal and hexadecimal
ASN text forms and MUST emit the form that was parsed, without
normalization, when round-tripping presentation data. Comparison of
ASNs for equality is performed on the wire-format value.

When converting a wire-format value to presentation form -- for
example when printing a record received on the wire -- no parsed text
form exists to preserve. In that case an implementation MUST render
ASNs of 2^32 and above in the hexadecimal group form and smaller ASNs
in decimal, matching SCION's canonical ASN text convention.

## Semantics

The "scion" SvcParamKey is only defined for ServiceMode records. In
AliasMode records recipients MUST ignore it, per Section 2.4.2 of
{{RFC9460}}.

Presence of the parameter indicates that the alternative endpoint
identified by the record's TargetName is reachable over SCION at each
of the listed SCION addresses. All other connection parameters of the
record apply to the SCION connection exactly as they apply to an IP
connection: in particular "port" designates the UDP or TCP port at the
SCION host address, "alpn"/"no-default-alpn" constrain the application
protocols offered on SCION connections, and "ech" applies to TLS
handshakes carried over SCION.

Like the "ipv4hint" and "ipv6hint" parameters, the listed addresses
are reachability information for the TargetName, not a security
assertion; see {{security}}.

The "scion" SvcParamKey SHOULD NOT be included in the "mandatory"
parameter. Marking it mandatory would cause clients without SCION
support to reject the entire record, defeating the incremental
deployment property that motivates this design. A zone operator MAY
mark it mandatory only in deployments where every intended client is
known to require SCION reachability information.

## Coexistence with the TXT Convention {#txt-coexistence}

Existing SCION deployments publish reachability using a TXT record of
the form:

~~~
example.com. 300 IN TXT "scion=71-2:0:4a,10.44.25.3"
~~~

This convention is consumed by IP-to-SCION translation gateways and
their DNS components, which synthesize AAAA answers inside a
SCION-mapped IPv6 prefix from it. This document does not deprecate the
TXT convention. During transition, zones SHOULD publish both the
"scion" SvcParam and the TXT record for a name, and when both are
published for the same name they SHOULD list the same set of SCION
addresses. They MAY differ where the two records deliberately serve
different access methods — for example, a TXT record carrying the
identity at which a translation gateway forwards to the service while
the SvcParam carries the address of a native SCION listener. Operators
publishing differing sets accept that gateway-mediated and native
clients reach the service at different SCION addresses.

# The "scion-policy" SvcParamKey {#scion-policy}

In SCION the client selects the end-to-end path. Absent other
information it does so with generic heuristics, while the service
operator often knows the service's actual traffic profile: a game
server is best reached over the lowest-latency path, a video origin
over a high-capacity one. The "scion-policy" SvcParamKey (number
65281, Private Use until IANA assignment) lets a service advertise
that knowledge as an advisory hint, in the same spirit as "alpn": it
influences how a client establishes its connection and MAY be ignored
entirely.

## Wire Format

The SvcParamValue is a sequence of policy items, each encoded as a
1-octet item type, a 1-octet value length, and that many octets of
value:

| Item             | Type | Length | Value                |
|------------------|------|--------|----------------------|
| prefer-latency   | 0x00 | 0      | (empty)              |
| prefer-bandwidth | 0x01 | 0      | (empty)              |
| prefer-hops      | 0x02 | 0      | (empty)              |
| max-latency      | 0x40 | 4      | uint32, microseconds |
| min-bandwidth    | 0x41 | 4      | uint32, Kbit/s       |

Recipients MUST skip unrecognized item types using the length octet.
An item whose length overruns the SvcParamValue, a defined item with
a length other than the one given above, and an empty SvcParamValue
each render the RR malformed; it MUST be entirely ignored.

The order of prefer items is the preference order. Senders MUST NOT
emit the same preference metric or the same threshold type twice;
recipients that nevertheless receive duplicates use the first
occurrence.

The units match SCION path-metadata conventions (latency in
microseconds, bandwidth in Kbit/s), so advertised values compare
directly against beaconed path properties.

## Presentation Format {#policy-presentation}

The presentation value is a comma-separated list of items in wire
order. A bare token names a preference metric: "latency" (0x00), "bw"
(0x01) or "hops" (0x02). A key=value pair expresses a threshold:
"maxlat" with a decimal value carrying a REQUIRED "us", "ms" or "s"
suffix, and "minbw" with a decimal value carrying a REQUIRED "K", "M"
or "G" suffix (bits per second). No item contains a comma, so unlike
"scion" no escaping is needed.

~~~
gameserver.example. 300 IN SVCB 1 . alpn=h3 (
    scion=71-2:0:4a\,10.44.25.3 scion-policy=latency,maxlat=50ms )
~~~

Implementations converting presentation to wire form emit items in
the order written. Converting wire to presentation form, threshold
values are rendered with the largest exact unit (50000 microseconds
as "50ms", 25000 Kbit/s as "25M"); round-tripping is therefore
value-exact but not necessarily token-identical.

## Semantics

The parameter is advisory: a client MAY ignore it entirely, and it
never expands or restricts reachability. It is only defined for
ServiceMode records that also carry the "scion" SvcParamKey;
recipients MUST ignore it in AliasMode records and in records without
"scion".

For the "prefer-bandwidth" metric and the "min-bandwidth" threshold,
"bandwidth" means the path's bottleneck bandwidth: the minimum of the
advertised link bandwidths of the path's constituent hops, not an
average or the bandwidth of any single hop. A path's usable capacity
is bounded by its narrowest link.

A client that honors the parameter SHOULD order candidate SCION paths
by the first preferred metric, breaking ties with subsequent metrics,
and SHOULD exclude paths that violate a threshold item. Threshold
items filter the candidate path set before prefer items rank it: a
client first discards paths that violate a threshold, subject to the
fail-open rule below, and only then orders the surviving paths by the
preference metrics. When this filtered, ranked set is larger than the
K limit of {{candidate-construction}}, prefer-item ranking is applied
to the full filtered set before it is truncated to K, not after;
truncating first could discard paths the policy would otherwise have
preferred. If every available path violates the thresholds the client
MUST ignore the thresholds: a policy can never make a destination
less reachable than it would be without the parameter. How a client
obtains metric values for candidate paths (control-plane metadata,
its own measurements) is out of scope for this document.

If multiple SVCB/HTTPS records for the same name carry
"scion-policy", the consistency guidance of {{scion-svcparam}}
applies unchanged.

# Use in Happy Eyeballs Version 3 {#hev3}

This section extends the algorithm of
{{I-D.ietf-happy-happyeyeballs-v3}} for clients with SCION support. All
timer values of that document (Resolution Delay, Connection Attempt
Delay, and their bounds) apply unchanged. A client without SCION
support ignores this section entirely.

## Candidate Construction {#candidate-construction}

During hostname resolution (Section 4 of
{{I-D.ietf-happy-happyeyeballs-v3}}), a SCION-capable client obtains
"scion" SvcParamValues from the SVCB/HTTPS answers it already queries.
No additional DNS queries are introduced.

For each ServiceMode record carrying a "scion" parameter, and for each
SCION address listed, the client queries its local SCION path service
for paths to the address's ISD-ASN. The client selects up to K paths
(RECOMMENDED default: K=3), ranked by the client's path selection
policy; in the absence of a more specific policy, clients SHOULD rank
by path metadata latency where available and by path length (number of
hops) otherwise. Each selected path yields one connection candidate:
the tuple (SCION address, path, port, ALPN set).

A client whose host lacks a native SCION stack but that can reach
SCION through an IP-to-SCION translation gateway MAY instead
synthesize a single candidate per SCION address, dialing the
gateway-mapped IPv6 address as an ordinary IPv6 candidate. Such a
client MUST NOT also construct native per-path candidates for the same
address.

A client whose native SCION stack offers only QUIC-based transports
MUST skip "scion" addresses on records whose effective ALPN set
(Section 7.1.2 of {{RFC9460}}) contains no QUIC-capable protocol
identifier: such a record advertises no application protocol the
client could run over a native SCION path, and constructing a
candidate for it would only manufacture an attempt destined to fail.
Operators SHOULD publish a QUIC-capable ALPN identifier (e.g., "h3")
alongside "scion" so that QUIC-only SCION clients can make use of the
parameter at all. This restriction bounds native per-path candidates
only: a client reaching SCION through an IP-to-SCION translation
gateway MAY still construct a gateway candidate for such a record,
because that leg is dialed as an ordinary IP candidate and is
genuinely TCP-capable, independent of the transports the client's
native SCION stack supports.

## Expansion Delay {#expansion-delay}

Section 4.2 of {{I-D.ietf-happy-happyeyeballs-v3}} defines a
Resolution Delay that briefly withholds a client's less-preferred
address family, giving a preferred family's resolution a bounded
chance to finish first. This document defines an OPTIONAL Expansion
Delay that applies the same idea one stage later, at candidate
construction ({{candidate-construction}}) rather than at name
resolution.

A client MAY delay launching its non-SCION connection attempts while
native SCION path lookup ({{candidate-construction}}) for the
hostname is still in flight, by at most an Expansion Delay. Path
lookup completion -- whether it yields one or more SCION candidates or
none at all -- releases any withheld non-SCION attempts immediately,
without waiting out the remainder of the Expansion Delay.

Support for Expansion Delay is OPTIONAL. A client that implements it
MUST default to disabled, behaving exactly as a client that does not
implement this mechanism at all. When an operator or application
enables Expansion Delay, 150 milliseconds is RECOMMENDED.

SCION path lookup is typically satisfied by one local RPC to the
client's path service and, under normal conditions, completes well
inside 150 ms. Withholding non-SCION attempts for a small, bounded
interval trades a modest and capped increase in worst-case
first-attempt latency for path-aware-first racing in deployments where
the operator judges that trade worthwhile. Because the delay is capped
and defaults to disabled, a client that does not enable it -- or whose
path lookup is slow or fails -- fares no worse than under plain HEv3
racing.

## Sorting

SCION is treated as a third address family in the sorting rules of
Section 5.3 of {{I-D.ietf-happy-happyeyeballs-v3}}: the "Preferred
Address Family Count" mechanism generalizes from two families (IPv6,
IPv4) to three (SCION, IPv6, IPv4).

When a native SCION stack is present, clients SHOULD order the first
SCION candidate before the first IP candidate, and thereafter
interleave families per the Preferred Address Family Count. Candidates
of the same SCION address on different paths are ordered by the path
ranking of {{candidate-construction}} and are separated by the
Connection Attempt Delay like any other successive candidates; a
failure of the leading path therefore costs one stagger interval
rather than a connection timeout.

Rationale for SCION-first ordering: the client possesses strictly more
information about the SCION candidates (explicit path metadata) than
about IP candidates, and SCION connection attempts exercise the
path-aware machinery this parameter exists to enable. Operators and
applications MAY configure IP-first ordering; the mechanism is
identical.

A client that honors "scion-policy" ({{scion-policy}}) applies it
when ordering SCION candidates: the preference metrics replace the
default latency-first ordering within the SCION address family, and
threshold items filter the path set (subject to the fail-open rule of
that section) before candidates are interleaved with IPv6 and IPv4.
The interleaving itself is unchanged.

## Racing and Success

Connection attempts are launched and staggered exactly per Section 6
of {{I-D.ietf-happy-happyeyeballs-v3}}. For SCION candidates whose
ALPN set selects a QUIC-based protocol, the success condition (in the
sense of Section 6.1 of that document, which explicitly permits
higher-level state checks) is the completion of the QUIC handshake
over the SCION path of that candidate.

The first candidate to succeed — SCION or IP — causes cancellation of
all other in-flight and pending attempts. An attempt that fails before
its successor's timer fires promotes the next candidate immediately.

A client MUST be prepared for a SCION candidate to fail due to path
expiry or revocation between path lookup and connection attempt, and
MUST treat this as an ordinary candidate failure.

# Security Considerations {#security}

TODO

# IANA Considerations

TODO

--- back

# Example Zone {#example-zone}

The following zone fragment illustrates a service published for
HTTP/3 and HTTP/2, dual-reachable over IP and SCION, together with the
coexisting legacy TXT record:

~~~
$ORIGIN scion.
@    3600 IN SOA  ns.scion. admin.scion. 2026071000 7200 3600 1209600 3600

web       IN AAAA 2001:db8:660::215
web       IN A    198.51.100.215
web       IN HTTPS 1 . alpn=h3,h2 port=443 scion=1-150\,10.150.0.81 scion-policy=bw,latency
web       IN TXT  "scion=1-150,10.20.3.215"

games     IN HTTPS 1 . alpn=h3 scion=71-2:0:4a\,10.44.25.3 scion-policy=latency
games     IN TXT  "scion=71-2:0:4a,10.44.25.3"
~~~

Note that "web" illustrates the divergence permitted by
{{txt-coexistence}}: its HTTPS SvcParam carries the native SCION
listener's address while its TXT record carries the identity at which a
translation gateway forwards to the same service; "games" publishes one
identical set in both records.

# Test Vectors {#test-vectors}

Each vector gives the presentation form of one SCION address and the
exact 24-octet wire encoding of a "scion" SvcParamValue carrying it,
shown as ISD (2 octets) | ASN (6 octets) | host (16 octets); the
whitespace is illustrative only. The reference implementation executes
these vectors as its test suite.

~~~
Presentation: 1-150,10.20.3.215
Wire:         0001 000000000096 00000000000000000000ffff0a1403d7

Presentation: 1-ff00:0:110,2001:db8::1
Wire:         0001 ff0000000110 20010db8000000000000000000000001

Presentation: 71-2:0:4a,10.44.25.3
Wire:         0047 00020000004a 00000000000000000000ffff0a2c1903
~~~

A value carrying both of the first two addresses is the 48-octet
concatenation of their encodings, with the presentation form:

~~~
scion=1-150\,10.20.3.215,1-ff00:0:110\,2001:db8::1
~~~

Rendering wire values follows the canonical-ASN rule of
{{presentation-format}}: at the threshold, the ASN ffffffff
(2^32 - 1) renders as "4294967295" while 000100000000 (2^32) renders
as "1:0:0".

Vectors for the "scion-policy" SvcParamValue (concatenated
type|length|value items; whitespace illustrative):

~~~
Presentation: scion-policy=bw,latency
Wire:         0100 0000

Presentation: scion-policy=latency,maxlat=50ms,minbw=25M
Wire:         0000 4004 0000c350 4104 000061a8
~~~

# Acknowledgments

