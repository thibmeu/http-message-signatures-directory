---
title: "Registry and Signature Agent card for Web bot auth"
abbrev: "Registry and Signature Agent card for Web bot auth"
category: info

docname: draft-meunier-webbotauth-registry-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "Web Bot Auth"
keyword:
 - not-yet
venue:
  group: "Web Bot Auth"
  type: "Working Group"
  mail: "web-bot-auth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/web-bot-auth/"
  github: "thibmeu/http-message-signatures-directory"
  latest: "https://thibmeu.github.io/http-message-signatures-directory/draft-meunier-webbotauth-registry.html"

author:
 -
    fullname: Thibault Meunier
    organization: Cloudflare
    email: ot-ietf@thibault.uk

normative:
  DIRECTORY: I-D.draft-meunier-http-message-signatures-directory
  HTTP-MESSAGE-SIGNATURES: RFC9421
  HTTP: RFC9110
  HTTP-CACHE: RFC9111
  HTTP-MORE-STATUS-CODE: RFC6585
  JWK-OKP: RFC8037
  JWK-THUMBPRINT: RFC7638


informative:
  OAUTH-BEARER: RFC6750
  OPENID-CONNECT-DISCOVERY:
    title: OpenID Connect Discovery 1.0
    target: https://openid.net/specs/openid-connect-discovery-1_0.html
  RATELIMIT-HEADER: I-D.draft-ietf-httpapi-ratelimit-headers
  RFC8446:
  ROBOTSTXT: RFC9309

--- abstract

This document describes a JSON based format for clients using {{DIRECTORY}}
to advertise information about themselves.

--- middle

# Introduction

TODO

Bots developers, platform providers, and users of {{DIRECTORY}} are sharing metadata.
This include name, contact, logo, or even their targetted purpose. Not all providers
have the exact same set of attribute they'd like to share, but the attribute set
is sufficiently large to justify a common definition.

Existing discovery mechanism, such as {{OPENID-CONNECT-DISCOVERY}} do not have the necessary
granularity, and purposue different goals.

This document introduces a new IANA registry "Signature Agent Card parameters", and
defines initial attributes.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Signature Agent Card {#signature-agent-card}

{{Section 4.1 of DIRECTORY}}

~~~
{
  "name": "Example Bot",
  "contact": "bot-support@example.com",
  "logo": "https://example.com/",
  "expected-user-agent": "Mozilla/5.0 ExampleBot",
  "rfc9309-product-token": "ExampleBot",
  "rfc9309-compliance": ["User-Agent", "Allow", "Disallow", "Content-Usage"],
  "trigger": "fetcher",
  "purpose": "tdm",
  "targeted-content": "Cat pictures",
  "rate-control": undefined,
  "rate-expectation": "avg=10rps;max=100rps",
  "known-urls": ["/", "/robots.txt", "*.png"],
  "keys": {
    "kty": "OKP",
    "crv": "Ed25519",
    "kid": "NFcWBst6DXG-N35nHdzMrioWntdzNZghQSkjHNMMSjw",
    "x": "JrQLj5P_89iXES9-vFgrIy29clF9CC_oPPsw3c5D0bs",
    "use": "sig",
    "nbf": 1712793600,
    "exp": 1715385600
  }
}
~~~

## Name {#signature-agent-parameter-name}

A friendly name for your signature agent.

Example

* ExampleBot
* My remote browser company

## Contact {#signature-agent-parameter-contact}

Email or other reliable communication channel.

Example
* bot-support@example.com
* https://example.com/contact

## Logo {#signature-agent-parameter-logo}

Image for quick visual identification

Example

* data:image/svg+xml;base64,deadbeef
* https://example.com/logo.png

## Expected user agent {#signature-agent-parameter-user-agent}

Exact string(s) or fragment patterns

TODO: what is a fragment pattern? is this defined somewhere? a quick regex language should be good.

Quote {{!RFC9110}}

Example

* Mozilla/5.0 ExampleBot

## robots.txt product token {#signature-agent-parameter-robotstxt-token}

Product token to be used for robots.txt directives per {{Section 2.2.1 of ROBOTSTXT}}.

Example

* ExampleBot

## robots.txt compliance {#signature-agent-parameter-robotstxt-compliance}

List of primitive the crawler is in compliance with for robots.txt

Example

* `["User-Agent", "Disallow"]`
* `["User-Agent", "Disallow", "CrawlDelay"]`

## Trigger {#signature-agent-parameter-trigger}

Whether it acts as a fetcher (requested by user) or crawler (autonomous scanning).

Proposed values:

1. "fetcher"
2. "crawler"

## Purpose {#signature-agent-parameter-purpose}

Intended use for collected data

Example

* use aipref vocabulary

## Targeted content {#signature-agent-parameter-targeted-content}

Specific type of data your agent seeks

Example

* SEO analysis
* Vulnerability scanning
* Ads verification

## Rate control {#signature-agent-parameter-rate-control}

How can an origin control your crawl rate

Example

* None
* CrawlDelay in robots.txt (non-standard)
* Custom webmaster tool
* 429 + {{RATELIMIT-HEADER}}

## Rate expectation {#signature-agent-parameter-rate-expectation}

Expected traffic and intensity of requests

Example

* 500 rps
* Spikes during reindexing

## Known URLs {#signature-agent-parameter-known-urls}

Predictable endpoints accessed, if known

Example

* /
* /ads.txt
* /favicon.ico
* /index.html

## Keys {#signature-agent-parameter-keys}

JWKS endpoint.

If key is present, it is RECOMMENDED that the card is signed using {{HTTP-MESSAGE-SIGNATURES}}.
Content-Digest header should be part of the signature.

TODO: describe signature, CWS keys.

Example

* https://example.com/.well-known/http-message-signatures-directory
* JWKS-directory

# Discovery

Discovery is done via a registry.
A registry is a list of URLs, each one pointing to a signature card.

URLS can be:

* https
* http
* data

Example

~~~txt
https://bot1.example.com/.well-known/signature-agent-card
https://crawler2.example.com/.well-known/signature-agent-card
https://zerotrust-gateway.example.com/.well-known/signature-agent-card
data:application/json;,...
~~~

### Out-of-band communication between client and origin
A service submitting their key to an origin, or the origin manually adding a service to their trusted list.

### Public list

Could be a GitHub repository like the public suffix list. The issue is the gating of such repositories, and therefore its governance.

### Signature-Agent-Card header {#signature-agent-card-header}

This allows for backward compatibility with existing header agent filtering, and an upgrade to a cryptographically secured protocol.

# Security Considerations

Malicious actors may put properties which are not theirs in the registry. Client SHOULD verify signature if they are present.

# Privacy Considerations

TODO


# IANA Considerations {#iana}

## Signature Agent Card Parameters Registry {#signature-agent-card-registry}

IANA is requested to create a new "Signature Agent Card Parameters" registry in a new
"Signature Agent Card" page to list metadata for signature agent protocols.
defined for use with the Privacy Pass token authentication scheme. These
identifiers are two-byte values, so the maximum possible value is
0xFFFF = 65535.

### Registration template

New registrations need to list the following attributes:

**Parameter Name:**
: The name requested (e.g. "useragent"). This name is
case sensitive.  Names may not match other registered names in a
case-insensitive manner unless the Designated Experts state that
there is a compelling reason to allow an exception

**Parameter Description:**
: Brief description of the Header Parameter

**Change Controller:**
: For Standards Track RFCs, list the "IESG".  For others, give the
name of the responsible party.  Other details (e.g., postal
address, email address, home page URI) may also be included.

**Reference:**
: Where this parameter is defined

**Notes:**
: Any notes associated with the entry
{: spacing="compact"}


New entries in this registry are subject to the Specification Required
registration policy ({{!RFC8126, Section 4.6}}). Designated experts need to
ensure that the token type is defined to be used for both token issuance and
redemption. Additionally, the experts can reject registrations on the basis
that they do not meet the security and privacy requirements defined in TODO.

### Initial Registry content

This section registers the Signature Agent Card Parameter names defined
in {{signature-agent-card}} in this registry.


**Parameter Name:**
: name

**Parameter Description:**
: A friendly name for your signature agent.

**Change Controller:**
: IETF

**Reference:**
: {{signature-agent-parameter-name}}

**Notes:**
: N/A


**Parameter Name:**
: contact

**Parameter Description:**
: Email or any other reliable communication channel

**Change Controller:**
: IETF

**Reference:**
: {{signature-agent-parameter-contact}}

**Notes:**
: N/A


**Parameter Name:**
: logo

**Parameter Description:**
: Image for a quick visual identification

**Change Controller:**
: IETF

**Reference:**
: {{signature-agent-parameter-logo}}

**Notes:**
: N/A


**Parameter Name:**
: expected-user-agent

**Parameter Description:**
: String or fragment patterns

**Change Controller:**
: IETF

**Reference:**
: {{signature-agent-parameter-user-agent}}

**Notes:**
: N/A


**Parameter Name:**
: rfc9309-product-token

**Parameter Description:**
: Robots.txt product token your signature-agent satisfies.

**Change Controller:**
: IETF

**Reference:**
: {{signature-agent-parameter-robotstxt-token}}

**Notes:**
: N/A


**Parameter Name:**
: rfc9309-compliance

**Parameter Description:**
: Does your signature-agent respect robots.txt.

**Change Controller:**
: IETF

**Reference:**
: {{signature-agent-parameter-robotstxt-compliance}}

**Notes:**
: N/A


**Parameter Name:**
: trigger

**Parameter Description:**
: Fetcher/Crawler

**Change Controller:**
: IETF

**Reference:**
: {{signature-agent-parameter-trigger}}

**Notes:**
: N/A


**Parameter Name:**
: purpose

**Parameter Description:**
: Intended use for the collected data

**Change Controller:**
: IETF

**Reference:**
: {{signature-agent-parameter-purpose}}

**Notes:**
: N/A


**Parameter Name:**
: targeted-content

**Parameter Description:**
: Type of data your agent seeks

**Change Controller:**
: IETF

**Reference:**
: {{signature-agent-parameter-targeted-content}}

**Notes:**
: N/A


**Parameter Name:**
: rate-control

**Parameter Description:**
: How can an origin control your crawl rate

**Change Controller:**
: IETF

**Reference:**
: {{signature-agent-parameter-rate-control}}

**Notes:**
: N/A


**Parameter Name:**
: rate-expectation

**Parameter Description:**
: Expected traffic and intensity

**Change Controller:**
: IETF

**Reference:**
: {{signature-agent-parameter-rate-expectation}}

**Notes:**
: N/A


**Parameter Name:**
: known-urls

**Parameter Description:**
: Predictable endpoint accessed

**Change Controller:**
: IETF

**Reference:**
: {{signature-agent-parameter-known-urls}}

**Notes:**
: N/A


**Parameter Name:**
: keys

**Parameter Description:**
: JWKS Endpoint

**Change Controller:**
: IETF

**Reference:**
: {{signature-agent-parameter-keys}}

**Notes:**
: N/A

--- back

# Test Vectors

TODO

# Implementations

TODO


# Acknowledgments
{:numbered="false"}

TODO

The editor would also like to thank the following individuals (listed in alphabetical order) for feedback, insight, and implementation of this document -


# Changelog
{:numbered="false"}

v00

- Initial draft
