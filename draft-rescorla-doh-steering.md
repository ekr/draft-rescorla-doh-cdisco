---
title: "Steering DNS over HTTPS Resolution"
abbrev: "DoH Steering"
docname: draft-rescorla-doh-steering-latest
category: info

ipr: trust200902
area: Internet
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: E. Rescorla
    name: Eric Rescorla
    organization: Mozilla
    email: ekr@rtfm.com

normative:
  RFC2119:

informative:
    DOHTRR:
        title: Trusted Recursive Resolver
        target: https://wiki.mozilla.org/Trusted_Recursive_Resolver
        author:
            - ins: Mozilla


--- abstract

This note describes a simple mechanism for determining whether a
provider network has a DNS over HTTPS {{!RFC8484}} server on it.


--- middle

# Introduction

Some applications perform their own name resolution rather than using
the system resolver, typically using an encrypted protocol such as DoH
{{RFC8484}} These applications have the choice of using either the
same recursive resolver configured into the system or of using a
resolver chosen out of a preconfigured list of trusted resolvers as in
{{DOHTRR}}.

If all of the trusted resolvers are publicly available, then there
are a number of mechanisms for choosing between them, for instance
randomly or based on performance. {{?I-D.arkko-abcd-distributed-resolver-selection}}
describes a number of potential mechanisms. However, if the
list of trusted resolvers includes Internet Service Providers
and the client is on a network associated with such a provider,
then it may be desirable to preferentially select the resolver
associated with that provider. This provides the benefits both
of using a DNS resolver with a known policy and using a resolver
that has high quality local information about the local network
topology.

This document describes a mechanism to address this situation.
This mechanism is being actively tested in the Firefox browser
with \[REDACTED]'s resolvers.


# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 {{RFC2119}} {{!RFC8174}}
when, and only when, they appear in all capitals, as shown here.

# DoH Resolver Discovery

The basic mechanism described in this document is straightforward and has
been chosen for ease of implementation rather than architectural correctness.

~~~~
                                +--------------+
Provision CNAME                 |              |
doh.test -> resolver.example -> | ISP Resolver |
                                |              |
                                +--------------+
                                    ^      |
                                    |      |
                          doh.test? |      | resolver.example
                                    |      |
                                    |      v
                                +--------------+
                                |              |
                                |    Client    |
                                |              |
                                +--------------+
~~~~


A network provider can publish the fact that it has an associated DoH
resolver on its network by configuring its own resolvers to server a
CNAME record at a well known domain name which cannot be otherwise
registered. In our current test deployment we use "doh.test" (see
{{!RFC2606}} for the definition of .test). This CNAME points to the
domain name of the associated DoH resolver ("resolver.example" in the
diagram above).

\[\[OPEN ISSUE: doh.test is probably the wrong domain. We may pick
something else later.]]

A client which wishes to test for the presence of a DoH resolver on
the network takes the following steps:

1. Do any testing for whether DoH should be disabled, such as looking
   for canary domains {{!I-D.grover-add-policy-detection}} or checking for
   local enterprise configuration.

1. Do a CNAME query for "doh.test" using either the system resolver
   or by talking directly to the recursive resolver IP address configured
   into the system.

1. If the query succeeds, then look up the CNAME in the list of
   preconfigured resolvers. If a match is found then use the
   resolver address for the matching preconfigured resolver. Otherwise
   fall back to the ordinary DoH resolver selection logic.

1. If the query fails then no associated resolver is present;
   fall back to the ordinary DoH resolver selection logic.

As noted above, this mechanism was designed for ease of implementation.
The next few sections describe the reasoning for some of the design
choices.

## Why DNS?

There have been a number of discussions of using non-DNS mechanisms
resolver information, for instance as in Section 4 of
{{?I-D.pauly-add-resolver-discovery}}. While arguably more
architecturally correct in terms of layering, they have a number of
deployment drawbacks:

- They require the client to have much tighter integration with the
  operating system in order to query the data. By contrast, with
  this mechanism, the client need only be able to do name resolution,
  which it generally already is able to do.

- They require new types of configuration which ISPs may not already
  be set up to do. By contrast, configuring DNS records is generally
  well understood.

- They rely on intermediate devices (e.g., NATs) being aware of the
  configuration information and passing it onto clients. These
  devices already do this with DNS information.

For this reason we believe that DNS is the easiest solution to deploy
quickly.


## Why a CNAME?

Most other proposed designs (e.g., {{?I-D.pp-add-resinfo}},
{{?I-D.pp-add-stub-upgrade}}, and
{{I-D.pauly-add-resolver-discovery}}) use new RRtypes. While this may
be the right answer eventually, it is less convenient for immediate
deployment for several reasons:

1. It is somewhat more difficult (though not impossible) to look up
new RRTypes on the client and provision them on the ISP resolver.

1. We have concerns about how frequently consumer-grade middleboxes
(e.g., WiFI routers) block unknown RRTypes. The data here is quite
old and limited, but still not particularly promising.

The choice to use a CNAME does have one major drawback: it does
not let us provide the URL template but only the name of the resolver.
This is not a problem for our system in practice because we will
only connect to resolvers on our preconfigured list and thus
we can use the CNAME as our lookup key for that list. We are working
to measure the rate of new RRType interference and may revise
this approach depending on the results of that.


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
