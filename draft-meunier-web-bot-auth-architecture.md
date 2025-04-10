---
title: "HTTP Message Signatures for automated traffic Architecture"
abbrev: "HTTP Message Signatures for Bots"
category: info

docname: draft-meunier-web-bot-auth-architecture-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
# area: AREA
# workgroup: Web Bot Auth
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
#  group: WG
#  type: Web Bot Auth
#  mail: "web-bot-auth@ietf.org"
#  arch: "https://mailarchive.ietf.org/arch/browse/web-bot-auth/"
  github: "thibmeu/http-message-signatures-directory"
  latest: "https://thibmeu.github.io/http-message-signatures-directory/draft-meunier-web-bot-auth-architecture.html"

author:
 -
    fullname: Thibault Meunier
    organization: Cloudflare
    email: ot-ietf@thibault.uk

normative:
  HTTP-MESSAGE-SIGNATURE: RFC9421
  HTTP: RFC9110
  HTTP-CACHE: RFC9111

informative:


--- abstract

This document describes an architecture for identifying automated
traffic using {{HTTP-MESSAGE-SIGNATURE}}. The goal
is to allow automated HTTP clients to cryptographically sign outbound
requests, allowing HTTP servers to verify their identity with confidence.


--- middle

# Introduction

Automated agents are increasingly used for legitimate
purposes, including AI assistants, indexing, testing, and automated
support tools. These agents often use general-purpose browsers,
making it difficult for websites to authenticate or identify them.
Existing methods, such as IP range allowlisting or User-Agent
strings, offer no integrity guarantees and are hard to maintain.

This document proposes a mechanism in which outbound HTTP requests
are signed using {{HTTP-MESSAGE-SIGNATURE}}. These signatures
can be verified by receiving servers using a public key associated
with the platform provider. This enables trusted interactions between
automated agents and HTTP servers, with improved security and
manageability.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# System

~~~aasvg
  +---------+             +----------------+             +--------+
  |  User   |             | Browser Agent  |             | Origin |
  +----+----+             +--------+-------+             +---+----+
       |                          |                           |
       |  Help me do this!        |                           |
       +------------------------->|                           |
       |                          |  GET /path/to/resource    |
       |                          |  Signature: abc===        |
       |                          +-------------------------->|
       |                          |                           |
       |                          |     <h1>Response</h1>     |
       |                          |<--------------------------+
       |  Here are proposals      |                           |
       +<-------------------------+                           |
       |                          |                           |

~~~

Discovery

HTTP Message signature and required fields

Accept Signature

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
