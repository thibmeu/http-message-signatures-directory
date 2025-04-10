---
title: "HTTP Message Signatures Directory"
abbrev: "HTTP Message Signatures Directory"
category: info

docname: draft-todo-yourname-protocol-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
number:
date:
consensus: true
v: 3
area: AREA
workgroup: WG Working Group
keyword:
 - next generation
 - unicorn
 - sparkling distributed ledger
venue:
  group: WG
  type: HTTPBIS
  mail: "httpbis@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/httpbis/"
  github: thibmeu/http-message-signatures-directory
  latest: https://thibmeu.github.io/http-message-signatures-directory/draft-meunier-httpbis-http-message-signatures-directory.html

author:
 -
    fullname: Thibault Meunier
    organization: Cloudflare
    email: ot-ietf@thibault.uk

normative:
  HTTP-MESSAGE-SIGNATURE: RFC9421
  HTTP: RFC9110

informative:


--- abstract

This document describe a method for clients using using {{HTTP-MESSAGE-SIGNATURE}}
to advertise their signing keys.
It defines a key directory format based on JWKS, as well as new HTTP Method Context
to allow for in-band key discovery.


--- middle

# Introduction

TODO Introduction


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# System

Define a well-known

Use JWKS

Deterministic key ID

Cache

Signature-Agent for discovery

# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
