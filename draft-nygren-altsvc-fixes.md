---
title: "Alt-Svc Fixes and Feature Candidates"
abbrev: "Alt-Svc Fixes"
docname: draft-nygren-altsvc-fixes-latest
category: info

ipr: trust200902
area: General
workgroup: TODO Working Group
keyword: Internet-Draft

stand_alone: yes
smart_quotes: no
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: E. Nygren
    name: Erik Nygren
    organization: Akamai
    email: erik+ietf@nygren.org


normative:

informative:

  SVCB:
    I-D.ietf-dnsop-svcb-https


--- abstract

HTTP Alternative Services has become the primary mechanism for HTTP/3 upgrade,
but overlaps with and disagrees with other developing standards, such as the
HTTPS resource record in DNS. This document explores a set of potential fixes
and/or additional features for Alt-Svc. It is used to record and share thoughts,
and is not expected to progress on its own.

--- middle

# Overview

Alt-Svc {{?AltSvc=RFC7838}} was published in April of 2016.  Since then, it has
become the primary mechanism to upgrade connections to HTTP/3, at least until
HTTPS RRs {{SVCB}} are standardized and widely
supported.

This brainstorms a set of potential fixes and feature candidates for an Alt-Svc
BIS.  In the spirit of brainstorming, some of the things in this list may be bad
ideas. One of the points of this document is to judge interest in each of these
items to determine what potential authors may be interested in including in a
draft.

A number of these items were deferred out of the SVCB/HTTPS draft.  Others are
based on regrets from implementation experience with AltSvc.  A few are
potentially valuable features.

It is not yet defined whether this should be a new header or can be done via
extensions to Alt-Svc as it exists today.

A number of major clients have yet to implement Alt-Svc to other hostnames
fully, and some of this is due to concerns that they have with the current
specification.

It may make sense to split this list into batches, but another option would be
to try and get all of these done at once even if as separate but cooperating
drafts.

# Potential Scope Items

## Incorporating errata and Editorial improvements

(Hopefully this is non-controversial.)

Includes:

- Incorporate errata
- Reference RFC 8336 “ORIGIN Frames” (regarding disabling connection coalescing)
- Incorporate {{?I-D.pardue-httpbis-dont-be-clear}}
- ...

## Fix ALPN handling

The ALPN semantics in {{?AltSvc}} are ambiguous, and problematic in some
interpretations. We should update {{?AltSvc}} to give it well-defined semantics
that match the HTTPS RRs.  For example, specify that the ALPN {{?ALPN=RFC7301}}
negotiated via the TLS handshake does not need to be the same as the ALPN
indicated in the AltSvc.

(From [HTTPS RR #246](https://github.com/MikeBishop/dns-alt-svc/issues/246) and other discussion
threads during HTTPS RR.  David Benjamin also has strong opinions on this
topic.)

One option would be to pull in the text we landed on for the HTTPS/SVCB draft,
see {{Section 6.1 of SVCB}}.  See also {{?I-D.thomson-tls-snip}}.

## Address concerns about Alt-Svc lifetime bounding

Some people have expressed concerns that Alt-Svc allows a compromised Origin to
hold onto clients forever by continuing to offer updated Alt-Svc entries.  There
may be ways to reduce the vulnerability exposure here, such as by periodic
reconfirmation with the "real" origin or something it controls.  This is of
particular concern when an Alt-Svc record has a much longer lifetime than an
HTTPS RR.

For example, if the Alt-Svc records were signed with a key published in DNS.
Records remain valid so long as the key that signed them is still claimed by the
domain.  An unsigned record has a very short lifetime bound.

## Support ECH

The HTTPS RR {{SVCB}} is currently the only way to retrieve keys for Encrypted
Client Hello {{?ECH=I-D.ietf-tls-esni}}.  To maintain security, it puts Alt-Svc
out-of-scope, since Alt-Svc cannot deliver ECH keys.

Two options (and there may be more) include:

- Add an ech= parameter to Alt-Svc
- Defining some better integration between Alt-Svc and HTTPS RRs.  For example,
  allow an AltSvc server name to be treated as an “AliasMode” reference to an
  HTTPS record.

## Better Interactions with HTTPS Record

Interaction between Alt-Svc and the HTTPS RR was a difficult problem in the
SVCB/HTTPS draft, as dnsop is not the ideal place to update Alt-Svc.

Providing a way for Alt-Svc to act as AliasMode references to HTTPS SvcMode
records seems like one clean way for interaction in that it avoids needing to
duplicate SVCB in Alt-Svc.  We would still need to address time-bounding and
trust considerations.

### Relative Priority

Alt-Svc and HTTPS records have different interaction and trust models.  Alt-Svc
is provided in-connection, and might be targeted to the specific client instead
of simply based on the client IP address. It is also received over an
authenticated channel, so that the Alt-Svc entry is known to come from the
origin server.

An HTTPS record is published in the DNS, and can be targeted only to the extend
that any DNS record might be targeted based on the client subnet.  In the
absence of DNSSEC signing and verification, information from the DNS must be
considered untrusted.

Alt-Svc records are at best cached from the last interaction with the server,
while HTTPS records can be retrieved when a client is preparing to make a
connection to the server.

When the client is preparing to make a connection, the Alt-Svc records are more
trusted and more tailored, while the HTTPS records are more current.  Should a
client not do HTTPS queries when it has a usable Alt-Svc record?  Should the
HTTPS records override Alt-Svc?  Or should the two mechanisms interact in some
way?

{{Section 8.3 of SVCB}} resolves the question this way:

> Clients that implement support for both Alt-Svc and HTTPS records SHOULD
> retrieve any HTTPS records for the Alt-Svc alt-authority, and ensure that
> their connection attempts are consistent with both the Alt-Svc parameters and
> any received HTTPS SvcParams. If present, the HTTPS record's TargetName and
> port override the alt-authority.

That is, if Alt-Svc provides an alternative host, it changes the hostname being
resolved away from the origin stated in the URL.  Once those HTTPS records are
retrieved, the capabilities advertised in Alt-Svc filter the endpoints
advertised in the DNS.  However, only the endpoints advertised in DNS are
actually used.

### Handling Negotiation Variability

During the development of SVCB/HTTPS, David Benjamin observed that Alt-Svc
preempts the negotiation of available protocols in TLS.  In ALPN, the client
offers its supported protocols, and the server selects one.  {{?AltSvc}} says
the following:

> If the connection to the alternative service does not negotiate the expected
> protocol (for example, ALPN fails to negotiate h2, or an Upgrade request to
> h2c is not accepted), the connection to the alternative service MUST be
> considered to have failed.

This means that, even if the client and the server both support a more preferred
protocol than that advertised in the Alt-Svc entry, the client MUST consider its
negotiation to have failed.  (There is a possible alternate reading, which is
that the client can rely on the server supporting the advertised protocol;
however, if the server does not support the advertised protocol, this can only
be detected if that's the only protocol the client offered.)

In contrast, SVCB says that the list of advertised ALPNs is used to select a
transport layer -- TLS, DTLS, or QUIC. The client offers all of its
supported protocols and performs negotiation with the server only within TLS.

This difference is motivated in part by the status of the DNS record as an
untrusted channel.  The set of ALPN tokens supported by the origin and its
alternatives is more trusted when obtained directly from the origin over an
authenticated connection.

## HTTP/3 Frame Definition

The ALTSVC frame has not been defined for HTTP/3.  Perhaps it should be
{{?I-D.bishop-httpbis-altsvc-quic}}. Alternatively, if the frame has not been
widely adopted, should it be deprecated from HTTP/2 instead?

## Accept-Alt-Svc Request Header

There is significant variation in client support for the Alt-Svc specification,
including some clients which only implement a subset of the specification.
Having an Accept-Alt-Svc request header that lists a set of supported Alt-Svc
features allows for extension of Alt-Svc but also allows for deprecation.

If we don’t deprecate the frames, we’d also need a SETTINGS equivalent.

There are potential client fingerprinting concerns here, so we’ll want to not go
too far with this.

## Improve/Replace Alt-Used Header

There is limited implementation support for Alt-Used out of privacy concerns.
It also only sends a subset of the Alt-Svc record being used, and there are
unclear interactions between Alt-Used and HTTPS RRs.

Daniel Stenberg points out:

> Alt-Used (RFC 7838 section 5) is a request header that only sends host name +
> port number, with no hint if that port number is TCP or UDP (or ALPN name),
> which makes at least one large HTTP/3 deployment trigger its Alt-Svc loop
> detection when only switching protocols to h3.

It is proposed that we replace or redefine Alt-Used and also define how it
interacts with SVCB.  Note that hostile origins have many knobs for getting this
information (e.g., encoding in hostnames, ports, or IPv6 addresses) so a goal
would be to allow non-hostile origins to get information on which Alt-Svc or
SVCB record is being used in a way that doesn’t make things worse from a privacy
perspective.

Some options include:

- Just send the whole Alt-Svc or SVCB binding used in the header
- Have a param encoding an N-bit or N-character value for the record-id.  This
  value would be sent as Alt-Used.  How many bits to use is an open question.
    - Allowing for a dynamic length where the client chooses how many bits or
      characters to include based on privacy budget is one attractive but
      complicated option.  Server implementers would put the most important info
      into earlier bits/characters.

The goal here is to allow for virtual hosting of alternative services, allowing
the server to know which alternative service was used (eg, for load feedback,
diagnostics/debugging, loop detection, and other operational purposes), but
without hacks like separate ports or IP addresses that leak information to
passive network observers.

The usefulness of Alt-Used is currently limited by the fact that most servers
simply send ":443" and some clients won't consider any other alternative
offered.

See some discussion and other options [here](https://github.com/MikeBishop/dns-alt-svc/issues/107).


## Path-Scoped Alt-Svc

The largest-scope, most disruptive, and perhaps most controversial item would be
to allow Alt-Svc to be scoped to URL paths with a way to indicate that
transitions to use the Alt-Svc should be done synchronously.

This is desired for use-cases of large content libraries where an Origin would
like to have clients use different endpoints for different objects while sharing
a single Origin.  This would also likely need negotiation.

This use-case is similar to that served by {{?I-D.reschke-http-oob-encoding}},
which is one possible solution.  In that model, the origin retains control of
the entire namespace while delegating delivery of particular objects to other
endpoints.

Extending Alt-Svc is another approach which might allow more flexibility.  For example:

- Client indicates via a request header (eg, Accept-*) or a SETTING that it supports this feature
- Server's Alt-Svc indicates that the path="/movies/MurderOnTheExampleExpress/"
  should be accessed by a particular alternative service
- Server returns a new 3xx response header response header indicating that the
  Alt-Svc should be used synchronously to fetch the response

## Persist and Caching Concerns

{{?AltSvc}} defines the `persist` parameter.

> Alternative services that are intended to be longer lived (such as those that
> are not specific to the client access network) can carry the "persist"
> parameter with a value "1" as a hint that the service is potentially useful
> beyond a network configuration change.
>
> When alternative services are used to send a client to the most optimal
> server, a change in network configuration can result in cached values becoming
> suboptimal.  Therefore, clients SHOULD remove from cache all alternative
> services that lack the "persist" flag with the value "1" when they detect such
> a change, when information about network state is available.

For some clients (e.g. cURL), detecting network changes is very tricky. Certain
clients default to behaving like persist=1 for all alternatives.

The RFC text today seems to imply that servers factor in client network
properties when deciding what to advertise. That is not true for all
deployments. The recommendation that a client invalidate Alt-Svc cache entries
based on their own network state changes can seem mistaken today. The situation
can potentially get worse with protocol evolution (connection migration,
multipath, etc).

This feature was designed to address the "mistaken mapping" scenario, where
either DNS mapping or anycast landed you at one POP but the server knows another
one is closer to you:  You're in Seattle, your VPN endpoint and DNS server is in
Sacramento, and so DNS resolution gives you a CDN node in California.  The
California endpoint gives you the unicast IP of the Seattle endpoint as a
friendly shove.

Similarly, it's one of the best work-around options for off-net DNS (via DoH,
ODoH, etc.) if there is a CDN endpoint very close to the user's network, but
doesn't work so well if the Alt-Svc record keeps being used after a network
change.

When you're no longer in the same network environment, we can trust mapping
again.  What is the signal this redirect is no longer useful if clients can't
reliably detect network changes?

## Radical Simplification

If we are able to reach wide deployment and use of the HTTPS record, it may
supersede many use cases for Alt-Svc.  We should reassess the needs from the new
baseline to see whether Alt-Svc can be radically simplified.  (Chrome never
fully implemented Alt-Svc redirection anyway.)

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

Martin Thomson, Lucas Pardue, and Mike Bishop reviewed and commented on [an
early version of this
draft](https://docs.google.com/document/d/1QNaXduqohACK93qLPpxkPJ2rHQMgWqUPL-DkZS11htQ/edit?ts=60a5dc92#).
