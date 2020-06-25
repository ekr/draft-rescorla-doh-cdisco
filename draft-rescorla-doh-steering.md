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
provider network offers a DNS over HTTPS {{!RFC8484}} server.


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
resolver on its network by configuring its own resolvers to serve a
CNAME record at a well known domain name which cannot be otherwise
registered. The current test deployment uses "doh.test" (see
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

1. If the query succeeds, then look up the CNAME record value in the list
   of preconfigured resolvers. If an exact match is found, then use the
   resolver address for the matching preconfigured resolver.
   Otherwise fall back to the ordinary DoH resolver selection logic.

1. If the query fails, then no associated resolver is present;
   fall back to the ordinary DoH resolver selection logic.

As noted above, this mechanism was designed for ease of implementation.
The next few sections describe the reasoning for some of the design
choices.

Considering that many applications do not act as a DNS client and instead
use platform functions such as getaddrinfo, the domain of the associated
resolver SHOULD also have an A record, so the call to getaddrinfo does
not fail.

## Why DNS?

There have been a number of discussions of using non-DNS mechanisms
resolver information, for instance as in Section 4 of
{{?I-D.pauly-add-resolver-discovery}}. While arguably more
architecturally correct in terms of layering, they have a number of
deployment drawbacks:

- They require the client to have much tighter integration with the
  operating system in order to query the data. By contrast, with
  this mechanism, the client need only be able to do name resolution via
  the system resolver, which it generally already is able to do via
  standard APIs.

- They require new types of configuration which ISPs may not already
  be set up to do. By contrast, configuring DNS records is generally
  well understood.

- They rely on intermediate devices (e.g., NATs) being aware of the
  configuration information and passing it onto clients. These
  devices already do this with DNS information.

For these reasons, DNS seems to be the easiest solution to deploy
quickly.


## Why a CNAME?

Most other proposed designs (e.g., {{?I-D.pp-add-resinfo}} and
{{?I-D.pp-add-stub-upgrade}}, and
{{I-D.pauly-add-resolver-discovery}}) use new RRtypes. While this may
be the right answer eventually, it is less convenient for immediate
deployment, for several reasons:

1. It is somewhat more difficult (though not impossible) to look up
new RRTypes on the client and provision them on the ISP resolver.

1. Some consumer-grade middleboxes (e.g., WiFI routers) may block
unknown RRTypes. The data here is quite old and limited, but still
not particularly promising.

The choice to use a CNAME does have one major drawback: it does
not let one provide the URL template but only the name of the resolver.
This is not a problem for systems using this technique in practice because
they will connect to resolvers on a preconfigured list and thus use the
CNAME record value as a lookup key for that list.

\[\[OPEN ISSUE: We are working to measure the rate of new RRType interference
and may revise this approach depending on the results of that.]]

# Security Considerations

Because the initial request for steering information is done over
insecure DNS, a local attacker or malicious local resolver can
substitute their own response. However, because this mechanism only
selects between preconfigured trusted resolvers, an attacker can only
steer you to a different resolver out of that list, which by
definition is also trusted. If the server which is steered to is not
publicly available, this mechanism can be used as a DoS
attack. Clients should test the selected server before committing to
it and otherwise fall back to the ordinary DoH selection logic.

Any local steering mechanism has potential privacy impacts: suppose
that a user uses their mobile device on ISP A, which steers to their
own resolver, and ISP B which does not.  In that case, the user's
browsing history will be spread over both ISP A's resolver and one of
the public trusted resolvers, which has an impact on the user's
privacy. This has to be balanced against the improvement obtained by a
local resolver and the level of metadata leakage that currently occurs
to the ISP.





# IANA Considerations

This document has no IANA actions.



--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
