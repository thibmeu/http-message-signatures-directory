---
title: "Extensions to Robots Exclusion Protocol for Signature-Based Agent Control"
abbrev: "Extensions to Robots Exclusion Protocol for Signature-Based Agent Control"
category: std

docname: draft-meunier-signature-agent-rep-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Web and Internet Transport"
workgroup: "HTTP"
keyword:
 - not-yet
venue:
  group: "HTTP"
  type: ""
  mail: "ietf-http-wg@w3.org"
  arch: "https://lists.w3.org/Archives/Public/ietf-http-wg/"
  github: "thibmeu/http-message-signatures-directory"
  latest: "https://thibmeu.github.io/http-message-signatures-directory/draft-meunier-signature-agent-rep.html"

author:
 -
    fullname: Thibault Meunier
    organization: Cloudflare
    email: ot-ietf@thibault.uk

normative:
  BOT-AUTH: I-D.draft-meunier-web-bot-auth-architecture
  RFC9110:
  RFC9309:
  SIGNATURE-DIRECTORY: I-D.draft-meunier-http-message-signatures-directory

informative:


--- abstract

This document defines new extensions to the Robot Exclusion Protocol ({{RFC9309}}) including Signature-Agent ({{Section 4 of SIGNATURE-DIRECTORY}}) and Max-Crawl-Rate.


--- middle

# Introduction

Bots are increasigly using Signature-Agent as a way to convey identity.
As such, there is interest from Origins to define robot policy based on this header.

This documents extends Robot Exclusion Protocol to support these, by defining a new group starting with signature-agent and max-crawl-rate.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

# Specification

## Protocol Definition

Group
: One or more signature-agent lines that are followed by one or more rules. The group is terminated by a signature-agent line or end of file. See {{signature-agent-line}}. The last group may have no rules, which means it implicitly allows everything. This draft introduces the max-crawl-rate directive, which may appear within a signature-agent group. It allows origin servers to specify the maximum number of requests per second permitted for that signature agent.

## Formal Syntax

Based on the formal syntax defined in {{Section 2.2 of RFC9309}}

~~~
startgroupline = user-agent-line / signature-agent-line ; a group can either be a user-agent, a signature-agent, or both.

user-agent-line = *WS "user-agent" *WS ":" *WS product-token EOL
signature-agent-line = *WS "signature-agent" *WS ":" *WS directory-token EOL
max-crawl-rate-line  = 1*DIGIT
directory-token = fqdn

fqdn = ... ; domain as defined by signature-agent
~~~

### Signature-Agent line

Crawlers set their own identity, which is called a directory-agent, to find relevant groups.
The directory token MUST contain only lowercase letters ("a-z"), underscores ("_"), hyphens ("-"), and dots (".").
The directory token SHOULD be a valid FQDN suffix of the identification string that the crawler sends to the service.
For example, in the case of HTTP {{RFC9110}}, the product token SHOULD be a substring in the Signature-Agent header.
The identification string SHOULD describe the public cryptographic key material of the crawler.
Here's an example of a Signature-Agent HTTP request header with a link pointing to a page describing the purpose of the crawler.example.com crawler, which appears as a suffix in the Signature-Agent HTTP header and as a directory token in the robots.txt directory-agent line

~~~
+==========================================+==============================+
| Signature-Agent HTTP header              | robots.txt signature-agent   |
|                                          | line                         |
+==========================================+==============================+
| Signature-Agent: crawler.example.com     | signature-agent: example.com |
+------------------------------------------+------------------------------+
~~~

### Max-Crawl-Rate line
The max-crawl-rate directive specifies the maximum number of requests per second that a signature agent SHOULD make to the origin server. Well-behaved agents are expected to comply by limiting their request rate accordingly to reduce server load. However, this directive does not enforce technical access restrictions, and adherence is voluntary.
Servers or CDNs MAY monitor agent behavior and take other measures if necessary to protect resources.

# Security Considerations {#security}

Signature-Agent group shares the security consideration of {{RFC9309}}. In addition, given Signature-Agent MAY present
a domain name identifying crawlers public cryptographic key material, implementors should treat the content of signature-agent
line as possibly sensitive.


# IANA Considerations

This document has no IANA actions.


--- back

# Examples

## Signature-Agent on example.com

~~~
Signature-Agent: example.com
Allow: *
~~~

## Signature-Agent with raw keyid

Not in the draft yet. We don't want to incentivise not rotating public keys.

~~~
Signature-Agent: poqkLGiymh_W0uP6PZFw-dvez3QJT5SolqXBCW38r0U
Disallow: /path/to/resource
~~~

# Acknowledgments
{:numbered="false"}

TODO acknowledge.
