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
    fullname: Maxime Guerreiro
    organization: Cloudflare
    email: maxime.guerreiro@gmail.com
 -
    fullname: Ulas Kirazci
    organization: Amazon
    email: ulaskira@amazon.com
 -
    fullname: Thibault Meunier
    organization: Cloudflare
    email: ot-ietf@thibault.uk

normative:
  ABNF: RFC5234
  AIPREF-VOCAB: I-D.draft-ietf-aipref-vocab
  DIRECTORY: I-D.draft-meunier-http-message-signatures-directory
  HTTP-MESSAGE-SIGNATURES: RFC9421
  HTTP: RFC9110
  HTTP-CACHE: RFC9111
  HTTP-MORE-STATUS-CODE: RFC6585
  JWK: RFC7517
  JWK-OKP: RFC8037
  JWK-THUMBPRINT: RFC7638


informative:
  DATAURL: RFC2397
  OAUTH-BEARER: RFC6750
  OPENID-CONNECT-DISCOVERY:
    title: OpenID Connect Discovery 1.0
    target: https://openid.net/specs/openid-connect-discovery-1_0.html
  RATELIMIT-HEADER: I-D.draft-ietf-httpapi-ratelimit-headers
  RFC8446:
  ROBOTSTXT: RFC9309
  UTF8: RFC3629

--- abstract

This document describes a JSON based format for clients using {{DIRECTORY}}
to advertise information about themselves.

This document describes a JSON-based "Signature Agent Card" format for signature agent using {{DIRECTORY}} to advertise metadata about themselve. This includes identity, purpose, rate expectations,
and cryptographic keys. It also establishes an IANA registry for Signature Agent
Card parameters, enabling extensible and interoperable discovery of agent
information.

--- middle

# Introduction

Signature Agents are entities that originate or forward signed HTTP requests on behalf
of users or services. They include bots developers, platforms providers,
and other intermediaries using {{DIRECTORY}}. These agents often
need to identify themselves, and establish
trust with origin servers.

Today, the mechanisms for doing so are inconsistent: some rely on User-Agent
strings (e.g. `MyCompanyBot/1.0`), others on IP address lists hosted on file servers (e.g. `badbots.com`), and still others on out-of-band
definitions (e.g. documentation on docs.example.com/mybot). This diversity makes it difficult for operators and origin servers
to reliably discover and share a Signature Agent’s purpose, contact information, or rate
expectations.
Existing discovery mechanisms, such as {{OPENID-CONNECT-DISCOVERY}}, do not have the necessary
granularity, and pursue different goals.

This document introduces a JSON-based "Signature Agent Card" format for Signature
Agents, to be published in registries and discovered by servers. It also
creates a new IANA registry of "Signature Agent Card Parameters" to ensure
extensibility and consistency of future attributes.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Signature Agent Card {#signature-agent-card}

Signature-Agent header is defined in {{Section 4.1 of DIRECTORY}}.
This section describes Signature Agent Card, a JSON object containing parameters describing the Signature Agent.

~~~
{
  "client_name": "Example Bot",
  "client_uri": "https://example.com/bot/about.html",
  "logo_uri": "https://example.com/",
  "contacts": ["mailto:bot-support@example.com"],
  "expected-user-agent": "Mozilla/5.0 ExampleBot",
  "rfc9309-product-token": "ExampleBot",
  "rfc9309-compliance": ["User-Agent", "Allow", "Disallow", "Content-Usage"],
  "trigger": "fetcher",
  "purpose": "tdm",
  "targeted-content": "Cat pictures",
  "rate-control": "429",
  "rate-expectation": "avg=10rps;max=100rps",
  "known-urls": ["/", "/robots.txt", "*.png"],
  "keys": [{
    "kty": "OKP",
    "crv": "Ed25519",
    "kid": "NFcWBst6DXG-N35nHdzMrioWntdzNZghQSkjHNMMSjw",
    "x": "JrQLj5P_89iXES9-vFgrIy29clF9CC_oPPsw3c5D0bs",
    "use": "sig",
    "nbf": 1712793600,
    "exp": 1715385600
  }]
}
~~~

Unless otherwise specified, all parameters in this document are OPTIONAL.
There MUST be at least one parameter set.

Parameters for which the value is unknown MUST be ignored.
All string values are UTF-8.

## Client Name {#signature-agent-parameter-name}

The `client_name` parameter provides a friendly identifier for the Signature Agent.

Example

* `ExampleBot`
* `My remote browser company`

## Client URI {#signature-agent-parameter-about}

The `client_uri` parameter provides inline content or a web page describing the bot: e.g. what does it do, how it handles data it fetches.

Only http, https or data:text/plain are allowed.

Example

* `https://example.com/bot/about.html`
* `data:text/plain,The Example bot is about providing an example.`

## Contacts {#signature-agent-parameter-contact}

The `contacts` parameter provides reliable communication channels in URI forms.
Typically, this is an email address.

Example

* `["mailto:bot-support@example.com"]`
* `["https://example.com/contact"]`

## Logo URI {#signature-agent-parameter-logo}

The `logo_uri` parameter provides an image reference for visual identification.

TODO: Recommendation for size and format, if there is a clear consensus or reference we can point to.

Example

* `data:image/svg+xml;base64,deadbeef`
* `https://example.com/logo.png`

## Expected user agent {#signature-agent-parameter-user-agent}

The `expected-user-agent` parameter specifies one or more `User-Agent` strings as defined in {{Section 10.1.5 of HTTP}}
or prefix matches. Prefixes MAY use `*` as a wildcard.

Example

* `Mozilla/5.0 ExampleBot`

## robots.txt product token {#signature-agent-parameter-robotstxt-token}

The `rfc9309-product-token` parameter specifies the product token used for
`robots.txt` directives per {{Section 2.2.1 of ROBOTSTXT}}.

Example

* `ExampleBot`

## robots.txt compliance {#signature-agent-parameter-robotstxt-compliance}

The `rfc9309-compliance` parameter lists directives from `robots.txt` that the
agent implements.

Example

* `["User-Agent", "Disallow"]`
* `["User-Agent", "Disallow", "CrawlDelay"]`

## Trigger {#signature-agent-parameter-trigger}

The `trigger` parameter indicates the operational mode of the agent.

Valid values:

1. `fetcher` - request initiated by the user
2. `crawler` - autonomous scanning

## Purpose {#signature-agent-parameter-purpose}

The `purpose` parameter describes the intended use of collected data. Values
SHOULD be drawn from a controlled vocabulary, such as {{AIPREF-VOCAB}}.

Example

* `search`
* `tdm`

## Targeted content {#signature-agent-parameter-targeted-content}

The `targeted-content` parameter specifies the type of data the agent seeks.
Its format is arbitrary UTF-8 encoded string.

Example

* `SEO analysis`
* `Vulnerability scanning`
* `Ads verification`

## Rate control {#signature-agent-parameter-rate-control}

The `rate-control` parameter indicates how origins can influence the agent’s
request rate.

TODO: specify a format

Example

* CrawlDelay in robots.txt (non-standard)
* Custom tool
* 429 + {{RATELIMIT-HEADER}}

## Rate expectation {#signature-agent-parameter-rate-expectation}

The `rate-expectation` parameter specifies anticipated request volume or
burstiness.

TODO: consider a format such as `avg=10rps;max=100rps`

Example

* 500 rps
* Spikes during reindexing

## Known URLs {#signature-agent-parameter-known-urls}

The `known-urls` parameter lists predictable endpoints accessed by the agent.


These URLs may be absolute URLs like `https://example.com/index.html`.
They could be relative path like `/ads.txt`.
Or they can use `*` as wildcard such as `*.png`.

Example

* `["/"]`
* `["/ads.txt"]`
* `["/favicon.ico"]`
* `["/index.html"]`

## Keys {#signature-agent-parameter-keys}

The `keys` parameter contains a JWKS as defined in {{Section 5 of JWK}}.

If `keys` is present, it is RECOMMENDED that the card is signed using {{HTTP-MESSAGE-SIGNATURES}}.
`Content-Digest` header MUST be included in the covered components.

TODO: describe signature, CWS keys.

Example

* https://example.com/.well-known/http-message-signatures-directory
* JWKS-directory

# Discovery

A registry is a list of URLs, each refering to a signature agent card.

The URI scheme MUST be one of:

* https (RECOMMENDED): Points to an HTTPS resource serving a signature agent card
* http: Points to an HTTP resource serving a signature agent card
* data: Contains an inline signature agent card

Example

~~~txt
# An example list of bots
https://bot1.example.com/.well-known/signature-agent-card
https://crawler2.example.com/.well-known/signature-agent-card

# Now the list of platforms
https://zerotrust-gateway.example.com/.well-known/signature-agent-card

# Below is an inlined card with the data URL scheme
data:application/json;,... # Invalid, not defined
~~~

## Formal Syntax

Below is an Augmented Backus-Naur Form (ABNF) description, as described in {{ABNF}}.

The below definition imports `http-URI` and `https-URI` from {{HTTP}}, and
`dataurl` from {{DATAURL}}.

~~~
registry = *(cardendpointline / emptyline)
cardendpointline = (
    http-URI /       ; As defined in Section 4.2.1 of RFC 9110
    https-URI /      ; As defined in Section 4.2.2 of RFC 9110
    dataurl          ; As defined in Section 3 of RFC 2397
) EOL


comment = "#" *(UTF8-char-noctl / WS / "#")
emptyline = EOL
EOL = *WS [comment] NL ; end-of-line may have
                       ; optional trailing comment
NL = %x0D / %x0A / %x0D.0A
WS = %x20 / %x09

; UTF8 derived from RFC 3629, but excluding control characters

UTF8-char-noctl = UTF8-1-noctl / UTF8-2 / UTF8-3 / UTF8-4
UTF8-1-noctl = %x21 / %x22 / %x24-7F ; excluding control, space, "#"
UTF8-2 = %xC2-DF UTF8-tail
UTF8-3 = %xE0 %xA0-BF UTF8-tail / %xE1-EC 2UTF8-tail /
         %xED %x80-9F UTF8-tail / %xEE-EF 2UTF8-tail
UTF8-4 = %xF0 %x90-BF 2UTF8-tail / %xF1-F3 3UTF8-tail /
         %xF4 %x80-8F 2UTF8-tail

UTF8-tail = %x80-BF
~~~

## Out-of-band communication between client and origin

A signature agent MAY submit their signature agent card to an origin, or the
origin MAY manually add them to their local registry.

## Public list

A registry MAY be provided via a GitHub repository, a public file server, or a
dedicated endpoint.

The registry SHOULD be served over HTTPS.

A client application SHOULD validate the directory format and reject malformed
entries.

## Signature-Agent header {#signature-agent-header}

Signature Agent Card format defined in {{signature-agent-card}} extends the
format of `Signature-Agent` header as defined in {{Section 4.1 of DIRECTORY}}.

When used for HTTP Message Signatures, and hosted on a well-known URL, Signature
Agent Card MAY be discovered via a `Signature-Agent` header.

# Security Considerations

Malicious actors may put properties which are not theirs in the registry. If signatures are present, clients MUST verify them.
Clients SHOULD reject cards with invalid signatures.

# Privacy Considerations

TODO


# IANA Considerations {#iana}

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

#### Client Name Parameter

**Parameter Name:**
: client_name

**Parameter Description:**
: A friendly name for your signature agent.

**Change Controller:**
: IETF

**Reference:**
: {{signature-agent-parameter-name}}

**Notes:**
: N/A

#### Client URI Parameter

**Parameter Name:**
: client_uri

**Parameter Description:**
: Describes what the bot does inline with data URI or with an HTTP link to an external resource

**Change Controller:**
: IETF

**Reference:**
: {{signature-agent-parameter-about}}

**Notes:**
: N/A

#### Logo URI Parameter

**Parameter Name:**
: logo_uri

**Parameter Description:**
: Image for a quick visual identification

**Change Controller:**
: IETF

**Reference:**
: {{signature-agent-parameter-logo}}

**Notes:**
: N/A

#### Contacts Parameter

**Parameter Name:**
: contacts

**Parameter Description:**
: An array of URI with a reliable communication channel; typically email addresses

**Change Controller:**
: IETF

**Reference:**
: {{signature-agent-parameter-contact}}

**Notes:**
: N/A

#### Expected User Agent Parameter

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

#### RFC9309 Product Token Parameter

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

#### RFC9309 Compliance Parameter

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

#### Trigger Parameter

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

#### Purpose Parameter

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

#### Targeted Content Parameter

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

#### Rate control Parameter

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

#### Rate expectation Parameter

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

#### Known URLs Parameter

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

#### Keys Parameter

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
<!--
Ulas Kirazci.
-->

The editor would also like to thank the following individuals (listed in alphabetical order) for feedback, insight, and implementation of this document -


# Changelog
{:numbered="false"}

v01

- Add contributors
- Aligning registry draft with oauth dynamic client registration iana registry
- Add an about-url field
- Add ABNF for discovery of signature-agent card (registry)
- Add precisions about known URLs
- Add placeholder for image size

v00

- Initial draft
