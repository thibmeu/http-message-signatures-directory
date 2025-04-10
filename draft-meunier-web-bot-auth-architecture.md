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
  +---------+             +-----------------+             +--------+
  |  User   |             | Automated Agent |             | Origin |
  +----+----+             +--------+--------+             +---+----+
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

Our system has three actors:

User: wants to perform some actions on the web.
Automated Agent: orchestrated user agent, such as Chromium. Can interact with the web, implements web standards. For every request, it constructs a valid HTTP request with {{HTTP-MESSAGE-SIGNATURE}} signature.
Origin: server hosting a resource. The user wants to access it through the browser.

## Generating Message signature
RFC 9421 defines components to be signed. 
The following parameters will need to be signed:
* @authority
* @signature-params(created, expires, keyid)

The private key is available to the automated agent at request time. Algorithms should be registered with IANA as part of HTTP Message Signatures Algorithm registry.

The creation of the signature is defined in RFC 9421, Section 3.1.

## Requesting a Message signature
RFC 9421 defines the Accept-Signature field which can be used to request a Message Signature from a client by a server. Servers MAY choose to request signatures from clients that did not initially provide them. If requesting, servers are expected to request the same parameters as those defined by the Generating Message Signature section.

## Validating Message signature
Upon receiving an HTTP request, the origin website has to verify the signature. The algorithm is provided in RFC 9421, Section 3.2.
Similar to regular User-Agent check, this happens at the HTTP layer, once headers are received.

If the key ID is not known, the origin MAY look at Operating-Agent header as described in ((MESSAGESIG-DIRECTOIRY-DRAFT))

If any of the steps from Section 3.2 fail, the signature validation fails.

## Anti-replay

Origins may want to to prevent signatures from being spoofed or used multiple times by bad actors and thus require a nonce to be added to the signature-params. This nonce would have to be unique for the validity window of the signature, as defined by created and expires attributes. Because the nonce is controlled by the client the origin needs to maintain a list of all nonces that it has seen that are still in the validity window of the signature. In addition, for platform providers offering different services, ai models, or others, the tag attribute of signature-params may be used.

## Discovery

This section describes the discovery mechanism for the automated agent directory.

The reference for discovery is a FQDN. It SHOULD provide a directory hosted on the well known registered in ((MESSAGESIG-DIRECTOIRY-DRAFT))

We add one field to the directory defined in the other draft:
"purpose": Ideally matches some IANA registry for preferences

TODO: replace the key with a JWK

Example

~~~json
{
  keys: [{
    alg:"ed25519",
    key: "-----BEGIN PUBLIC KEY-----...",
    not-before: 1743578485, // optional
    not-after: 1745000000, // optional
  }],
  operating_agent: "my.company.agent.test", // optional. SHOULD match the operating agent domain
  purpose: "search" // Ideally matches some IANA registry for preferences
}
~~~


### Out-of-band communication between client and origin
A service submitting their key to an origin, or the origin manually adding a service to their trusted list

### Public list

Could be a GitHub repository like the public suffix list. The issue is the gating of such repositories, and therefore its governance.

### Operating-Agent header

This is defined in the sibling draft.
This allows for backward compatibility with existing header agent filtering, and an upgrade to cryptographically secured protocol.


# Security Considerations

TODO Security


# IANA Considerations

This document has no IANA actions.


--- back

# Acknowledgments
{:numbered="false"}

TODO acknowledge.

# Implementations

This draft has a couple of implementations

Clients:
* Chrome MV3
* Cloudflare Workers

Servers:
* Caddy plugin
* Cloudflare Workers

A demontstration server has been deployed to https://http-message-signatures-example.research.cloudflare.com/
It uses RFC9421 ed25519 test signing and verifying keys.