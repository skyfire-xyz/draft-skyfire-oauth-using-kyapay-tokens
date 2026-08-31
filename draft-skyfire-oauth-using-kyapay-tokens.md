---
title: "Using KYAPay Tokens"
#abbrev: ""
category: std

docname: draft-skyfire-oauth-using-kyapay-tokens-latest
submissiontype: IETF  # also: "independent", "editorial", "IAB", or "IRTF"
#number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - agent
 - agentic commerce
 - agentic
 - commerce
 - identity
 - JWT
 - bot management
 - fraud
 - account takeover
 - payment
venue:
  github: "skyfire-xyz/draft-skyfire-oauth-using-kyapay-tokens"
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  latest: "https://skyfire-xyz.github.io/draft-skyfire-oauth-using-kyapay-tokens/draft-skyfire-oauth-using-kyapay-tokens.html"

author:
-
  name: Ankit Agarwal
  organization: Skyfire Systems Inc.
  email: ankit_agarwal@yahoo.com
  uri: https://skyfire.xyz
-
  name: Michael B. Jones
  organization: Self-Issued Consulting
  email: michael_b_jones@hotmail.com
  uri: https://self-issued.info/
-
  name: Srinivasa Thumma
  organization: Akamai
  email: sthumma@akamai.com

# see https://github.com/cabo/kramdown-rfc/wiki/Syntax2#authors-contributors

normative:
  RFC2119:
  RFC8174:
  RFC7515: # JWS
  RFC7519: # JWT
  RFC8693: # OAuth 2.0 Token Exchange
  RFC9110: # HTTP Semantics
  RFC9421: # HTTP Message Signatures
  RFC7800: # JWT proof-of-possession (cnf)
  I-D.skyfire-oauth-kyapay-token:
  I-D.skyfire-oauth-kyapay-token-exchange:

informative:
  RFC6749: # OAuth 2.0
  RFC8725: # JWT BCP
  RFC8176: # AMR values
  RFC8705: # OAuth 2.0 Mutual-TLS
  RFC9449: # DPoP
  RFC9901: # SD-JWT
  I-D.skyfire-oauth-amr-values:
  I-D.skyfire-oauth-id-verification:
  I-D.skyfire-oauth-aml-methods:
  I-D.ietf-oauth-client-id-metadata-document:
  NIST-800-63A:
    title: "Digital Identity Guidelines: Identity Proofing and Enrollment (NIST SP 800-63A)"
    target: https://pages.nist.gov/800-63-3/sp800-63a.html
    date: false
    author:
      - org: National Institute of Standards and Technology
  MCP:
    target: https://modelcontextprotocol.io/specification/2025-11-25
    title: Model Context Protocol Specification
    date: November 25, 2025
    author:
      - org: "Anthropic and the Model Context Protocol Contributors"
  A2A:
    title: Agent2Agent (A2A) Protocol Specification
    date: 2025
    target: https://a2a-protocol.org/latest/specification
    author:
      - org: A2A Project
  UCP:
    title: Universal Commerce Protocol (UCP) Specification
    date: 2025
    target: https://ucp.dev/latest/specification/overview/
    author:
      - org: UCP Contributors
  VINTENT:
    title: Verifiable Intent Specification
    date: 2025
    target: https://verifiableintent.dev/spec/
    author:
      - org: Verifiable Intent Contributors
  SPIFFE:
    title: "Secure Production Identity Framework for Everyone (SPIFFE)"
    target: "https://spiffe.io/docs/latest/spiffe-about/overview/"
    date: false
    author:
      - org: "SPIFFE Project / Cloud Native Computing Foundation"
  KYAPAY-ORG:
    title: "KYAPay: Verified Agent Identity and Payments"
    target: "https://kyapay.org/"
    date: false
    author:
      - org: "Skyfire Systems Inc."
  OpenID.Core:
    target: https://openid.net/specs/openid-connect-core-1_0.html
    title: "OpenID Connect Core 1.0 incorporating errata set 2"
    date: 2023-12-15
    author:
      - name: Nat Sakimura
      - name: John Bradley
      - name: Michael B. Jones
      - name: Breno de Medeiros
      - name: Chuck Mortimore

...

--- abstract

The KYAPay Token is a JSON Web Token (JWT) that carries verified identity ("Know Your Agent", KYA) and payment (PAY) information for requests made by software agents on behalf of human principals.
This document describes how security intermediaries -- bot managers, fraud managers, account-takeover (ATO) protection systems, and customer identity and access management (CIAM) systems -- consume KYAPay tokens to answer a question that traditional bot detection cannot: "did a verified human authorize this agent?", rather than "is this a human?".
It specifies how KYAPay tokens are carried in HTTP requests, how they are validated (including in combination with request-signing layers such as HTTP Message Signatures), and how the verified, layered identity in a token is used to make access, routing, fraud detection, account-lifecycle, and step-up decisions.
It defines the token-consuming "verifier" role that the KYAPay Token leaves unspecified.
It is intentionally non-prescriptive about how tokens are created, because agent architectures, agent-identity technologies, and agent-communication protocols are diverse and still emerging;
the token itself is the interoperability contract.

--- middle

# Introduction

For most of the history of the web, the operative question asked by web security infrastructure was binary: is this request from a human or from a bot?
Bots were treated as unwelcome, and bot managers, web application firewalls, and related systems were built to detect and block automated traffic.
As automation became more nuanced, the question evolved into a three-way distinction: human, "good bot" (such as a search-engine crawler or a monitoring probe), and "bad bot".

The rise of capable AI agents changes the question again.
The relevant distinction is now:

* a **human** interacting directly;
* a **human acting through an agent** (a "human-via-agent" request), where a person or organization has authorized a software agent to act on their behalf; and
* a **bot**: unattended automation acting without the authorization of, or on behalf of, an identified human principal.

Put succinctly, the distinction that matters is no longer human versus good bot versus bad bot;
it is human-direct and human-via-agent, together, versus bots.
And the sharper form of the question is "did a verified human authorize this agent?", rather than "is this a human?".
An AI agent is neither a bot to be managed nor a human to be onboarded through a conventional flow;
it is a new category of legitimate client that existing bot and fraud detection cannot, on its own, distinguish from malicious automation.

Bots were unwelcome, and they remain unwelcome.
The new requirement is that both human requests and human-via-agent requests must be able to succeed across the existing web security infrastructure.
A human-via-agent request is, at the transport and application layers, frequently indistinguishable from a bot;
it is programmatic, it may originate from data-center IP space, and it may not carry a conventional interactive browser fingerprint.
Absent a reliable signal, security intermediaries either block legitimate agents (destroying utility for the human principal) or relax their defenses (admitting malicious bots).
What has been missing is a consistent, verifiable signal of human authorization behind an otherwise programmatic request.

KYAPay tokens {{I-D.skyfire-oauth-kyapay-token}} supply that signal.
A KYAPay token is a signed JWT {{RFC7519}} {{RFC7515}} that conveys verified identity claims about the human principal, the agent, and the agent platform (the KYA, or "Know Your Agent", information), and optionally payment credentials (the PAY information).
A validated KYA token asserts that a trusted issuer has verified that this human principal (to some level of assurance) authorized this agent (running on this platform) to act on their behalf;
a validated PAY token further asserts that a trusted issuer has authorized this agent to pay a specific amount to a specific target.
Together, they form a chain of trust from the human principal to the action, and -- for payments -- to settlement.

## Scope

This document describes how *consumers* of KYAPay tokens -- in particular the security intermediaries that guard websites, APIs, and accounts -- convey, validate, and act on KYAPay tokens.
It does not redefine the token;
the claims, their meanings, and the base validation rules are specified in the KYAPay Token {{I-D.skyfire-oauth-kyapay-token}} and are referenced normatively here.

This document is deliberately **normative about consumption** and **non-prescriptive about creation**.
{{SecInt}} places requirements on how intermediaries validate and use tokens.
{{TokCreat}} discusses token creation as a software-architecture problem and intentionally does not mandate a mechanism;
the rationale is given in {{NonPrescCreat}}.

## Relationship to Other Documents

This document builds directly on the KYAPay Token {{I-D.skyfire-oauth-kyapay-token}}, which defines the KYA, PAY, and KYA-PAY token types, the claim schema (including the layered identity claims `hid`, `apd`, and `aid`), and the core token validation procedure.
The specification defines the token and the roles of issuer, initiator, and target, but it does not define the role of the party that receives and validates a token in order to make a security decision.
This document defines that role -- the *verifier* (or relying party) -- and specifies its behavior.

Three related specifications define JWT claims and values that a verifier can use to reason about how a principal was authenticated and verified, and that MAY appear in KYAPay tokens:
the Additional Authentication Method Reference Values specification {{I-D.skyfire-oauth-amr-values}}, which defines additional `amr` {{RFC8176}} claim values,
the Identity Verification Methods Values specification {{I-D.skyfire-oauth-id-verification}}, which defines the `ivm` claim and values, and
the Anti-Money Laundering Methods Values specification {{I-D.skyfire-oauth-aml-methods}}, which defines the `aml` claim and values.

# Conventions and Definitions

{::boilerplate bcp14-tagged}

## Terminology

This specification uses the Initiator/Target and agent identity terms Initiator Agent, Initiator Principal, Agent Platform,
and the identity grouping claims `hid`, `apd`, and `aid` defined by {{I-D.skyfire-oauth-kyapay-token}}.

This document defines the following terms:

**Human-via-agent request**:
: An HTTP request or protocol message issued by an agent that carries a valid KYAPay token attesting that the agent is acting on behalf of an identified, authorizing human principal (an individual or an organization).

**Bot**:
: Unattended automation that is not acting on behalf of, or with the authorization of, an identified human principal, and that does not present a valid KYAPay token.
  This document does not attempt to further classify bots;
  existing "good bot" / "bad bot" mechanisms are orthogonal and continue to apply.

**Verifier (Relying Party)**:
: The entity that receives a KYAPay token, validates it, and acts on the result.
  A verifier may be a security intermediary (below), a Target's own resource server, or an edge/CDN provider acting on a Target's behalf.

**Security intermediary**:
: A verifier, typically operated by a security vendor or by the Target, that inspects requests and makes an admission, routing, scoring, or lifecycle decision before or as the request reaches the Target's application.
  This document addresses four common (and often overlapping) kinds of security intermediary:

  **Bot manager**:
  : A system operating primarily at the network and transport layer that decides whether to admit, challenge, throttle, or block a request based on whether it appears to be automated.

  **Fraud manager**:
  : An application-layer system that scores the risk of a transaction or action, typically using identity, reputation, behavioral, and device signals.

  **Account-takeover (ATO) protector**:
  : A system that detects and prevents unauthorized access to existing accounts, for example by distinguishing legitimate sessions from hijacked or automated ones.

  **CIAM (customer identity and access management) system**:
  : A system that manages account creation, authentication, and login for a Target's customers.

**Human presence signal**:
: The aggregate assurance, derived from a validated KYAPay token, that a request is authorized by an identified human principal.
  See {{HumanPresence}}.

**Assurance level**:
: A per-entity indication of how strongly an identity in the token was verified (for example, the identity-proofing level of the human principal).
  See {{HumanPresence}}.

# Design Principles {#DesignPrinc}

This section is informative.
It records the assumptions that motivate the normative requirements in later sections.

## Human Identity is Paramount

An underlying hypothesis of KYAPay is that commerce -- with or without payment -- is conducted between humans and human organizations, not between machines.
An agent is a tool through which a human or organization acts.
Consequently, the identity that matters most is the human identity behind the request, and liability for a request must ultimately be assignable to a human party: the principal, the developer of the agent, the operator of the agent platform, or some other identified party.
KYAPay tokens are structured to convey these identities so that this auditable, cryptographically verifiable chain of responsibility -- from agent action back to an accountable human -- can be established.

## Ubiquity of Access on the Existing Web

Agents are useful in proportion to the breadth of the web they can act across.
If a human must pre-configure access for an agent at each site before the agent can act, the result is indistinguishable from the workflow automation that already exists, and the principal loses the probabilistic discovery -- of new merchants, products, services, and alternatives -- that is one of the main reasons to use an agent at all.
The goal is therefore agentic access to the existing web, not a parallel, agent-only surface that must be set up in advance.
This goal can only be met if security intermediaries admit legitimate human-via-agent requests by default when a valid token is present.

## Works with Today's Web at Zero Merchant Tech Lift

Websites exist today, in the millions, and merchants and services already know how to sell through them, manage fraud, make offers, build brand loyalty, and test changes.
A key property of a website (and, similarly, of a merchant-controlled MCP {{MCP}} server) is that the merchant controls its own interface and is not locked into an API schema that must be carefully versioned;
agents driven by large language models (LLMs) can frequently learn to use these interfaces automatically.
A protocol for delivering agent identity and payment credentials should therefore work with today's web without requiring merchants to build new APIs.
KYAPay meets this by conveying credentials in the HTTP request itself ({{ConveyingRequests}}), which requires no change to how a merchant's site is built.

## One Source of Truth Across All Layers of Defense

Bot managers operate at the transport/network layer;
fraud managers, ATO protectors, and CIAM systems operate at the application layer.
Today, each makes its decision in isolation, and their verdicts on the same request can contradict one another.
A single validated KYAPay token gives every layer the same verified identity and, where present, payment context, so that their decisions are more likely to be consistent than conflicting.

## Runtime Business Decisions

A further goal is to give the Target the information it needs to make a business decision at request time: do I want to do business with this human, arriving via this agent, on this platform, for this action?
Because the token carries verified, layered identity and (optionally) payment context, the Target and its intermediaries can decide at runtime whether to admit, price, personalize, step up, or decline.
As one illustration, a merchant that recognizes a KYAPay-bearing request MAY adapt its existing pages for agent consumption using the same techniques it already uses to restyle pages, retaining ownership of its interface and its existing fraud and good/bad-actor signals rather than exposing a separate API surface.

# The Trust Stack: Bearer Tokens Today, Proof of Possession Ahead {#TrustStack}

This section is informative.
It situates KYAPay tokens among the other mechanisms a verifier encounters and, importantly, distinguishes what is deployed today from the planned evolution, so that the consumption requirements in {{SecInt}} are read against the correct baseline.

## KYA is an Issuer-Signed Bearer Token Today

As currently deployed, a KYA (or KYA-PAY) token is an issuer-signed, short-lived bearer token.
Its single most valuable operational property is that a Target has to do almost nothing to consume it: verify a JWT against the issuer's public key via JWKS, exactly as it would verify any OpenID Connect {{OpenID.Core}} or OAuth {{RFC6749}} token.
Any Target with a standard JWT library can participate, including those behind CDNs, API gateways, and serverless platforms.
Preserving that low bar is a primary design goal ({{DesignPrinc}});
mechanisms that would raise it are weighed against the adoption they would cost.

A bearer token establishes identity but not proof of possession: possession of the token is all that is needed to present it, so a token captured within its validity window can be replayed.
Today this residual, in-window risk is bounded rather than eliminated, by means that do not raise the consumption bar:

* short token lifetimes (`exp`), so a captured token expires quickly;
* audience binding (`aud`, and where used `tdm`/`tsi`), so a token minted for one Target cannot be presented to another;
* TLS on every hop, which mitigates on-the-wire capture and man-in-the-middle; and
* issuer-gated onboarding (KYC/KYB), which mitigates malicious agents and Targets being admitted to the network in the first place.

The accepted residual threats, and the irreducible ones, are stated in {{SecCon}}.

## Identity and Proof of Possession are Different Layers

It is useful to separate two questions a verifier may want answered:

* *Who is behind this request?*
  This is answered by the KYA token: a trusted, neutral issuer that performs KYC/KYB attests to the identity of the human principal (`hid`), agent platform (`apd`), and agent (`aid`), acting as a client-side trust anchor analogous to a Certificate Authority.
  KYA is a set of verifiable *nouns* -- identity, verification status, authorization, payment -- and, being a JWT, it is transport-agnostic: the same token rides HTTP, WebSockets, gRPC metadata, a message queue, or a stdio MCP channel, and can be logged and independently re-verified at rest.

* *Did the sender actually hold the credential's key?*
  This is proof of possession, which a bearer token does not provide.
  Establishing it requires the agent to sign the request itself;
  over HTTP, HTTP Message Signatures {{RFC9421}} are a mechanism for this.
  A signature proves control of a key but says nothing about whose key it is -- it is a *verb*, not an identity format.

Because these are different layers, HTTP Message Signatures {{RFC9421}} and similar request-signing mechanisms are complementary to KYA, not alternatives to it;
request signing supplies authenticity and possession, while KYA supplies identity, authorization, and payment.
A complete proof-of-possession design uses both, bound together, as described next.

## The Proof-of-Possession Path (Planned Evolution)

When per-request signing becomes practical at scale, the planned evolution of KYA is a Certificate-Authority-like model that adds proof of possession without changing the trust anchor:

1. the issuer onboards the agent as a CA would;
1. the agent generates a key pair and provides its public key to the issuer;
1. the issuer binds that public key into the KYA token using the JWT confirmation claim `cnf` {{RFC7800}}. The token MUST carry `cnf.jwk` -- the key itself. A key identifier would satisfy {{RFC7800}} only on its own terms, which hold "provided the recipient is able to obtain the identified key": a verifier meeting the agent for the first time holds no such key, and a thumbprint is a one-way hash from which none can be recovered. The issuer cannot determine when that proviso is met -- it knows neither which verifiers a token will reach, nor the initiator's intent, nor the state of any interaction between initiator and target. Nor can it rely on continuity between tokens: where a target de-duplicates on `jti` to prevent replay, each token is fresh and must stand on its own. Carrying the key is therefore the only form that works in the general case. Other confirmation members MAY additionally be present; a verifier MUST NOT be required to obtain the key by any means other than reading `cnf.jwk`; and
1. the agent signs each request in place -- a signature over the request (method, path, body digest, timestamp/nonce) using {{RFC9421}} -- verified against the key in `cnf`, with no separate proof-of-possession token.

This yields an unbroken chain from request to key to identity: the request is signed by a key, and that key is vouched for as the initiator's by the issuer.
Preferring a request signed against the `cnf` key over a detached, DPoP-style proof {{RFC9449}} keeps a single token -- the verifier validates the KYA token once (per session), caches the `cnf` key, then performs only a fast per-request signature check, instead of validating two artifacts and confirming they are bound to each other.
When present, this binding is consumed as specified in {{ProcModel}} and closes the in-window replay and malicious-Target-reuse risks of the bearer model ({{SecCon}}).

Adoption remains the constraint on when this becomes mandatory.
Per-request signing is costly: hardware-protected keys are secure but slow, software keys are fast but exfiltratable, and post-quantum algorithms do not make signing faster.
Defeating replay of the proof itself further requires either per-Target nonce caches or binding the signature to the request content, and other request-bound designs considered (mutual TLS {{RFC8705}}, TLS channel binding, session-based schemes) each raise the participation bar or add Target-side state.
Until fast, secure signing and a workable anti-replay approach exist that keep the Target's job about as simple as verifying a JWT, KYA remains a bearer token by default and proof of possession is an optional, forward-compatible upgrade rather than a precondition for participation.

# Conveying KYAPay Tokens in Requests {#ConveyingRequests}

## The KYAPay-Token HTTP Header Field {#HeaderField}

When KYAPay tokens are used with websites and HTTP APIs, a human-via-agent request carries its token(s) in an HTTP header field, in the same spirit that a human's browser conveys identity and payment details in the request itself.
For a directly interacting human, such details are typically carried in the request body (for example, form fields at checkout);
for a human-via-agent request, they are carried in the `KYAPay-Token` header field defined here.

The `KYAPay-Token` header field is an HTTP request header field whose value is a KYA, PAY, or KYA-PAY token, as defined in {{I-D.skyfire-oauth-kyapay-token}}.
HTTP field names are case-insensitive {{RFC9110}}.

    KYAPay-Token = token-jwt *( OWS "," OWS token-jwt )
    token-jwt    = 1*( ALPHA / DIGIT / "-" / "_" / "." )

A request MAY convey more than one token (for example, separate KYA and PAY tokens) either as a comma-separated list in a single `KYAPay-Token` header field or as multiple `KYAPay-Token` header fields;
per {{Section 5.2 of RFC9110}}, multiple field lines with the same name are equivalent to a single comma-separated field line.
Because a JWT uses only the base64url alphabet plus the period (".") separator, it never contains a comma, so list parsing is unambiguous.

A sender:

* MUST place exactly one JWT in each list member;
* SHOULD NOT send more than one token of the same `typ` ({{I-D.skyfire-oauth-kyapay-token}}) in a single request; and
* MUST NOT rely on the ordering of tokens within the field to convey meaning.

A recipient:

* MUST determine the type of each token from its `typ` header parameter ({{I-D.skyfire-oauth-kyapay-token}}), not from its position; and
* MUST ignore a `KYAPay-Token` member it cannot parse as a JWT, while continuing to process the remaining members, unless local policy requires rejecting the whole request.

Intermediaries and caches MUST treat `KYAPay-Token` as containing sensitive, request-specific credentials;
it MUST NOT be cached in a way that would allow it to be replayed on a different request, and it MUST NOT be logged or forwarded to parties other than legitimate participants (see {{PrivCon}}).

The `KYAPay-Token` header field is intended for use over TLS.
A token MUST NOT be sent over a non-TLS-protected connection.
When the token carries a proof-of-possession key ({{TrustStack}}), it SHOULD be accompanied by a request Message Signature {{RFC9421}} over that key;
otherwise it is a bearer token, subject to the controls and accepted risks in {{SecCon}}.

## Conveying Tokens over Other Interfaces

Beyond HTTP requests to websites and APIs, agents interact through a growing set of interfaces and protocols, including agent-to-tool interactions such as MCP {{MCP}}, agent-to-agent interactions such as A2A {{A2A}}, agent-to-API interactions (for example, OpenAPI descriptions converted into locally callable tools), and emerging protocols such as Universal Commerce Protocol {{UCP}} and Verifiable Intent {{VINTENT}}.
KYAPay is designed so that the same self-contained token can be conveyed across any of these interfaces -- as an HTTP header field, a message field, or a tool argument -- to deliver verified identity and payment credentials to the receiving service, API, or agent.
Because a KYA token is a self-contained JWT, it is transport-agnostic in a way that a request-signing mechanism is not: HTTP Message Signatures {{RFC9421}} are meaningful only for HTTP exchanges, whereas the same token can ride HTTP, WebSockets, gRPC metadata, message queues, or a stdio MCP channel, and can also sit at rest in a store or audit log for later verification.
This document specifies only the HTTP header binding in {{HeaderField}};
bindings for other protocols are expected to be defined by those protocols or by companion specifications.
In all cases, the validation and usage rules of {{SecInt}} apply to the token, once received.

## Authenticating the Counterparty

Conveying a token securely also requires that the sending agent deliver it to the intended recipient and that the recipient be able to bind the token to itself.
The `aud` claim, together with the optional `tdm` (target domain) and `tsi` (target service identifier) claims defined in {{I-D.skyfire-oauth-kyapay-token}}, allows a recipient to confirm that a token was minted for it and to mitigate replay to a different party ({{SecCon}}).
At the transport and discovery layers, mechanisms such as DNS domain names (with TLS server authentication), A2A agent cards {{A2A}}, and MCP server identification {{MCP}} help an agent confirm that it is communicating with the correct counterparty before presenting a token.

# Using KYAPay Tokens at Security Intermediaries {#SecInt}

This section is normative.
It defines a common processing model and then describes how each kind of security intermediary uses the validated token.
Throughout, "validate the token" means to perform the validation procedure defined in {{I-D.skyfire-oauth-kyapay-token}}, Section 4 (JWT header validation, signature and issuer validation, and validation of `exp`, `iat`, `jti`, `aud`, and `env`), plus the PAY-specific validation for `pay+jwt` and `kya-pay+jwt` tokens.

## Common Processing Model {#ProcModel}

On receiving a request that carries one or more KYAPay tokens, a security intermediary:

1. MUST extract each token as described in {{ConveyingRequests}}.
1. MUST confirm that the token's issuer is one the verifier trusts for the claims being relied upon (see {{SecCon}}).
   This check MUST precede any processing that resolves a value taken from the token, and in particular any network retrieval driven by the `iss` claim.
   Until the token has been verified, `iss` is supplied by the sender: resolving it before the issuer is known to be trusted would allow an unauthenticated request to cause an outbound fetch to an arbitrary URL of the sender's choosing.
   The trusted-issuer set is configured out of band and is not derived from the token.
1. MUST validate each token as specified in {{I-D.skyfire-oauth-kyapay-token}}, Section 4.
   A token that fails validation MUST NOT be treated as conveying a human presence signal, and the intermediary MUST fall back to its default (token-absent) policy for that request.
1. MUST verify the token signature against a key obtained from the issuer's JWK Set, discovered via the `iss` claim using the `/.well-known/jwks.json` mechanism ({{I-D.skyfire-oauth-kyapay-token}}).
   Because KYAPay tokens are self-contained, this verification SHOULD be performed locally, without a synchronous callout to the issuer per request;
   issuer keys SHOULD be cached and refreshed according to standard JWK Set practice.
1. MUST confirm that the token is intended for this recipient by checking the `aud` claim (and, if used, the `tdm` and `tsi` claims) as described in {{SecCon}}.
1. MUST confirm that the `env` claim (for example, `production`) matches the environment the intermediary is operating in.
   When the token carries a proof-of-possession key (the `cnf` claim {{RFC7800}}, Section 4), MUST verify that the request is signed by that key -- over HTTP, using HTTP Message Signatures {{RFC9421}} -- and reject the request if the signature is absent or invalid.
1. When the token carries no such key, it is a bearer token: the intermediary MUST rely on the controls in {{SecCon}} (short lifetime, audience binding, TLS, issuer-gated onboarding) and MUST NOT assume that mere presentation of the token proves the sender holds a key bound to it.
1. SHOULD use the token's claims to inform its decision, as described in the following subsections, only after successful validation.

A security intermediary MUST NOT treat the mere presence of a `KYAPay-Token` header field as a human presence signal;
only a successfully validated token from a trusted issuer conveys that signal.

## Determining Human Presence and Assurance {#HumanPresence}

The core purpose of consuming a KYAPay token is to distinguish a human-via-agent request from a bot, and to gauge how strongly to trust it.
After validation, a verifier SHOULD derive the human presence signal from the layered identity in the token, treating each layer as a distinct entity verified by distinct means:

* the human principal (`hid`): the presence and content of the human identity claim, and how strongly it was verified -- for example a verified status and verifier ({{I-D.skyfire-oauth-kyapay-token}}), and, where a deployment conveys it, an identity-proofing assurance level (such as an Identity Assurance Level per {{NIST-800-63A}}) and the
`amr` {{I-D.skyfire-oauth-amr-values}},
`ivm` {{I-D.skyfire-oauth-id-verification}}, and
`aml` {{I-D.skyfire-oauth-aml-methods}}
claims describing how the principal was authenticated and verified;

* the agent platform (`apd`): the identity of the operator that built and runs the agent, and whether it was verified (for example, a business/Know-Your-Business attestation), which supports reputation-based logic about the platform; and

* the agent instance (`aid`): the specific agent, including a meaningful name and, where present, references to its OAuth identity ({{I-D.ietf-oauth-client-id-metadata-document}}) and a proof-of-possession key bound to the token via `cnf` {{RFC7800}} (or a request-signing key published in an A2A Agent Card {{A2A}}).

Because these are different entities verified to different degrees, a verifier SHOULD NOT collapse them into a single yes/no signal.
Instead, it SHOULD apply per-entity, per-action policy.
For example, a verifier might require a higher human-principal assurance level and a "registered" agent for a payment than for browsing:

    /checkout: require hid assurance >= document+biometric
	       require apd verified as a business
	       require aid bound to a verified domain
    /payment:  require hid assurance >= document+biometric
	       require hid authenticated with a hardware key
	       require aid registered by its platform

The strength of the human presence signal is a function of which identities are present, how strongly each was verified, and how much the verifier trusts the attesting issuer and verifier.
A verifier MAY combine the token signal with its existing signals rather than replacing them, and MAY accept a lower assurance for low-risk actions while requiring step-up ({{StepUp}}) for higher-risk ones.

## Bot Managers {#BotMgr}

A bot manager operates at the transport/network layer and is chiefly concerned with admitting or blocking a request with minimal added latency.
KYAPay tokens are well suited to this role:

* **Low Latency.**
  Because the token is a self-contained, signed JWT, the bot manager performs standard JWT verification and needs no per-request callout to a third party.
  Issuer keys SHOULD be cached.

* **Access Control by Identity.**
  A validated token gives the bot manager verified initiator identity (and, when the request targets an agent, target identity), which it MAY use to admit the request rather than challenging or blocking it as suspected automation.

* **Replay Resistance.**
  A bearer token captured within its validity window can be replayed;
  a bot manager bounds this with short lifetimes, audience binding, and TLS ({{SecCon}}).
  Where a token carries a proof-of-possession key ({{TrustStack}}), the bot manager (or the edge/CDN provider acting for the Target) SHOULD additionally verify the per-request signature {{RFC9421}} against that key, which removes the in-window replay exposure.

* **Routing.**
  Target identity claims MAY be used for routing decisions.

Network-origin claims (for example, an agent's declared source IP ranges) MAY be used as a weak corroborating signal only.
They are unreliable as proof of origin -- addresses change and can be masked by VPNs, CGNAT, or shared pools -- and a verifier MUST NOT treat an IP match alone as sufficient to bypass other controls, nor treat an IP mismatch as conclusive.
Short lifetimes and audience binding today, and proof-of-possession where available ({{TrustStack}}), not IP correlation, are the recommended controls against token misuse.

A bot manager that completes the checks above SHOULD admit the request (subject to rate and abuse limits) rather than subjecting it to bot-blocking challenges designed for unattended automation.

## Fraud Managers

A fraud manager operates at the application layer and scores the risk of an action or transaction.
It benefits from the token's layered identity and verification metadata:

* It SHOULD incorporate the per-entity verified identity (`hid`, `apd`, `aid`) and assurance levels into its risk model, distinguishing a request backed by a strongly verified human principal on a reputable platform from a weakly verified or anonymous one.

* It SHOULD use the `verifier`/`verified`/`verification_id` sub-claims, and the
`amr` {{I-D.skyfire-oauth-amr-values}}
`ivm` {{I-D.skyfire-oauth-id-verification}}, and
`aml` {{I-D.skyfire-oauth-aml-methods}}
claims where present, to weight how the principal was authenticated and verified.

* For `pay+jwt` and `kya-pay+jwt` tokens, it SHOULD use the payment context (`amt`, `cur`, `stp`, `sti`, and the pricing claims `tpr` and `tps`) as additional transaction risk signals, after performing the PAY-token validation of {{I-D.skyfire-oauth-kyapay-token}}, Section 4.
  Because the PAY token pins amount, currency, and target, the fraud manager MAY rely on those as hard limits enforced by the token rather than re-deriving them.

A fraud manager need not sit inline in the request path.
In practice the token is made available to it either inline (the Target forwards the received token to its fraud manager) or out of band (for example, script running on the Target's page captures the `KYAPay-Token` value and sends it to the fraud manager before the request reaches the Target's application).
The same token MAY be inspected before the action, after it, or both.
In all of these arrangements, the token issuer is not in the transaction path;
the fraud manager verifies the self-contained token locally by JWKS, as described in {{ProcModel}}.
The fraud manager MAY continue to apply its existing behavioral and device models;
the token supplements, and does not replace, those signals.

## Step-Up Authentication {#StepUp}

Not all actions carry the same risk;
viewing a loyalty balance is far less sensitive than making a large payment.
A KYAPay token attests the assurance established at the time of issuance, which may be insufficient for a high-value action.
A verifier SHOULD be able to require a higher assurance level for sensitive actions and to signal that requirement so that a fresh token can be obtained after additional verification of the human principal (for example, a step-up authentication such as a push-notification approval, a one-time code, or a re-run of identity verification).

This document does not define a wire mechanism for a Target to request step-up out of band from an issuer;
that is an open item (see {{SecCon}}).
Until such a mechanism is standardized, verifiers SHOULD express assurance requirements as local policy over the assurance signals in {{HumanPresence}} and decline actions whose required assurance is not met.

## Account-Takeover Protectors {#ATOProt}

An ATO protector distinguishes legitimate account sessions from hijacked or unauthorized ones.
KYAPay tokens let it separate three cases that previously looked alike: a direct human-present session, a human-initiated agentic session, and an unauthorized automated session.

* It SHOULD treat a validated KYAPay token, bound to its request per {{TrustStack}}, as evidence that a session is a human-initiated agentic session rather than an unattended attack, and SHOULD record that distinction for auditing.

* When the agent needs to act within an existing account, the token MAY be exchanged for an OAuth access token using KYAPay Token Exchange {{I-D.skyfire-oauth-kyapay-token-exchange}} or OAuth 2.0 Token Exchange {{RFC8693}}.
A Security Token Service, Identity Provider, or OAuth 2.0 {{RFC6749}} Authorization Server validates the KYA token, extracts principal claims such as the email address from `hid`, and issues an access token that the agent uses against the Target.

* The access token resulting from such an exchange SHOULD be sender-constrained (for example, via DPoP {{RFC9449}} or mutual TLS {{RFC8705}}) rather than issued as a freely transferable bearer token, so that exchanging the KYA token does not simply reintroduce a replayable credential.

* Cross-domain trust must be established: the Target's authorization server can fetch the issuer's keys via the `iss` JWKS endpoint, but it must also decide whether to trust that issuer and how to interpret its claims (see {{SecCon}}).

* A valid token attests authorization at the moment of issuance, not continuous human control.
  An ATO protector SHOULD NOT assume that the human remains in control for the token's whole lifetime, SHOULD prefer short token lifetimes, and SHOULD apply step-up ({{StepUp}}) for sensitive actions.

## CIAM Systems

A CIAM system manages account creation and login for a Target's customers.
With KYAPay tokens it can manage agents consistently with humans:

* It MAY create an account for an agent (or for the human principal behind an agent) directly from a validated KYA or KYA-PAY token, drawing account attributes such as the email address from the `hid` claim.

* It SHOULD use KYAPay Token Exchange {{I-D.skyfire-oauth-kyapay-token-exchange}} or OAuth 2.0 Token Exchange {{RFC8693}} to exchange a KYA token for an access token, enabling logged-in experiences (loyalty, saved preferences, offers) for human-via-agent sessions, applying the sender-constraining and cross-domain-trust considerations of {{ATOProt}}.

* It SHOULD maintain the linkage between a human principal and the agents authorized to act for them, so that a principal can be given visibility into, and control over, agent activity and vice versa.

## Consistency Across Layers

Because all of the intermediaries above validate the *same* token and draw on the *same* verified claims, an operator SHOULD configure them to reach consistent verdicts on a given request: a request admitted as a verified human-via-agent request by the bot manager should not be scored as an anonymous bot by the fraud manager.

## Granular Bad-Actor Mitigation and Triage {#BadActor}

The layered identity in a KYA token (`hid` for the principal, `apd` for the platform, `aid` for the agent instance) lets intermediaries choose the scope of a mitigation deliberately, rather than being forced into a single broad, "nuclear" block.
Because every request carries all three identities, a verifier can select the narrowest scope that is effective and escalate only as warranted:

* the individual **human principal** (`hid`) -- e.g., block or throttle one abusive user while all other users of the same agent continue to be served;

* the specific **agent instance** (`aid`) -- e.g., isolate one compromised or malfunctioning agent without affecting the principal's other agents or the rest of the platform; or

* the entire **agent platform** (`apd`) -- e.g., when an operator is systematically abusive, unresponsive, or untrusted, block or de-rate all traffic from that platform.

The same granularity supports responses short of blocking the requests, such as throttling, requiring step-up ({{StepUp}}), or lowering an assurance score, applied at whichever tier is appropriate.
Choosing the tightest effective scope isolates threats while protecting the reputation of well-behaved platforms and avoiding collateral damage to legitimate human-via-agent requests;
reserving platform-wide action for cases that genuinely warrant it keeps that option available without making it the default.

## Feedback to the Token Issuer {#IssuerFeedback}

The mitigation described above is local: a verifier acts on its own traffic.
Because the token issuer is the party that vouches for the initiator and can decline to vouch again, there is value in a feedback loop that lets a security intermediary report observed bad behavior back to the issuer, so that action can be taken at the source rather than only at each verifier independently.
When a security intermediary observes abusive or anomalous behavior that it can attribute, using the layered identity in the token, to a particular entity, it MAY report that observation to the token's issuer (identified by `iss`).
The attribution follows the same tiers as {{BadActor}}:

* the individual human principal (`hid`);
* the specific agent instance (`aid`); or
* the agent platform as a whole (`apd`).

On receiving such feedback, and subject to its own verification and anti-abuse safeguards, the issuer MAY take appropriate action, up to and including ceasing to issue tokens to the implicated initiator, agent, or platform -- the issuance-side counterpart to a verifier's refusal to accept a token, and complementary to the token-lifetime and revocation considerations of {{SecCon}}.
This mirrors the Certificate-Authority analogy of {{SecCon}}: just as a CA can stop issuing (and can revoke) certificates for a misbehaving subscriber, a KYAPay issuer can stop vouching for a misbehaving initiator.

This document does not define the mechanism, format, or trust model for this feedback channel;
how a verifier authenticates to an issuer, how reports are structured, how issuers guard against false or malicious reports, and what evidence is required are all open items.
Feedback SHOULD be treated as sensitive: reports identify principals, agents, or platforms and MUST be shared only with the relevant issuer and handled per {{PrivCon}}.
An issuer SHOULD corroborate feedback (for example, requiring reputation, multiple independent reports, or evidence) before taking action that would affect a principal, so that the channel cannot itself be used to deny service to legitimate initiators.

## Non-Payment and Handoff Flows

Not every human-via-agent flow includes a payment made by the agent.
In a common pattern, the agent performs discovery and assembles a cart but does not pay;
it then hands off to the human, who completes checkout in a browser, in a later session, or even in a physical store.
In such flows, the KYA token (carried without a PAY token) still provides the human presence signal that lets a security intermediary admit the agent's discovery and cart-building activity.
Intermediaries SHOULD accept KYA-only requests (those conveying a KYA token but no PAY token) for actions that do not themselves settle a payment, and MUST NOT treat the absence of a PAY token as a reason to classify an otherwise valid human-via-agent request as a bot.

Because the human identity claim `hid` (for example, an email address) is stable across these touchpoints, a Target or fraud manager MAY link the agent's earlier work to the human's eventual purchase -- for attribution, personalization, and fraud analysis -- even when the purchase completes outside the agent session or on a different channel.

# Token Creation Considerations {#TokCreat}

This section is non-normative with respect to *how* tokens are created.
It records considerations that KYAPay Token Issuers and agent developers should weigh, but it deliberately does not mandate a mechanism.
The normative interoperability contract is the token defined by {{I-D.skyfire-oauth-kyapay-token}} and consumed as specified in {{SecInt}};
how a given agent obtains a valid token is an implementation and architecture choice.

## Why this Document is Not Prescriptive about Creation {#NonPrescCreat}

Readers accustomed to fixed API contracts may expect a specification to prescribe exactly how a credential is minted.
For agents, such prescription is neither possible nor desirable at this time, for several reasons:

* **Agent architectures are diverse.**
  Agents are built in fundamentally different ways -- computer-use agents that drive a desktop, browser-use agents that drive a browser, and programmatic agents that call APIs or tools -- and there is no single formula for building one.
  An agent may be an individual's personal, ad hoc creation, or a highly governed and deterministic enterprise agent for a regulated workflow, or anything in between.

* **Credentials may need to be shielded from the LLM.**
  In many designs, it is desirable that the LLM driving an agent never sees raw credentials.
  Token creation may therefore be placed inside deterministic tools that the LLM can invoke but whose outputs (the tokens, or the secrets used to mint them) it cannot read.
  Whether this matters depends on the deployment;
  a specification that mandated a single creation flow could not accommodate both shielded and unshielded designs.

* **Agent-identity and communication protocols are still emerging.**
  Multiple protocols for agent identity and for agent communication -- agent-to-tool (MCP {{MCP}}), agent-to-agent (A2A {{A2A}}), request signing ({{RFC9421}}), OAuth client identity ({{I-D.ietf-oauth-client-id-metadata-document}}), agent-to-API, Verifiable Intent {{VINTENT}}, UCP {{UCP}}, and others -- are under active development and require significant investment to reach production.
  Standardizing a single creation mechanism now would bind KYAPay to choices that are not yet settled.

* **The web already exists.**
  In contrast to those emerging interfaces, websites and merchant-controlled endpoints exist today at scale and require no new integration effort from merchants ({{DesignPrinc}}).
  A usable protocol must therefore work now, over the existing web, which argues for standardizing the delivered token rather than the creation path.

KYAPay's position is that the token is the fixed point, and creation is allowed to vary.
This is what lets a single token type serve individual and enterprise agents, shielded and unshielded designs, and today's web alongside tomorrow's agent protocols.

## Guidance for KYAPay Token Issuers

Given the above, KYAPay Token Issuers are encouraged to make token creation possible and easy regardless of the technology an agent uses:

* **Match the issuing interface to the agent.**
  A programmatic agent using a deterministic tool may be well served by a single REST endpoint with optional arguments -- one call that can mint a KYA, PAY, or KYA-PAY token with any supported settlement type (payment cards, stablecoins, and so on).
  An agent driven through MCP {{MCP}} may instead be better served by several narrow, specific tools (one per token type and per settlement type), so as to reduce the decision burden placed on the LLM.

* **Support multiple identity technologies.**
  Because there is no single agent-identity standard today, issuers should be prepared to accept and bind a variety of identity inputs when minting tokens, including (for example) CIMD {{I-D.ietf-oauth-client-id-metadata-document}}, WIMSE / SPIFFE {{SPIFFE}}, Agent Name Service (ANS), request-signing keys published in A2A Agent Cards {{A2A}}, and public/private key pairs.

* **Be available across agent technologies and protocols.**
  Issuers should be reachable by agents irrespective of the agent's runtime technology, the identity protocol it uses, and the communication protocol it speaks.

Origination and issuance need not be performed by the same party.
The `iss` claim identifies the party that signs (issues) the token, while the optional `ori` claim ({{I-D.skyfire-oauth-kyapay-token}}) identifies the party that originated -- that is, assembled and requested -- it;
these can be the same entity or different entities.
Likewise, the verified identity information carried in `hid` and `apd` may be supplied by the agent platform or a third-party verifier and merely attested by the issuer.
Verification itself need not be completed up front: it can be progressive, beginning with a lightweight check (such as email verification) and stepping up as a Target's requirements demand.
Consumers observe the result of these choices through the `verified`/`verifier` sub-claims and, where used, the
`ivm` {{I-D.skyfire-oauth-id-verification}},
`amr` {{I-D.skyfire-oauth-amr-values}}, and
`aml` {{I-D.skyfire-oauth-aml-methods}}
claims, rather than through any mandated creation flow.

The objective of all of the above is a single outcome: a valid KYAPay JWT that is consumable by every transacting party -- the Initiator, the Target, and the bot managers, fraud managers, ATO protectors, and CIAM systems in between.

# Security Considerations {#SecCon}

When validating the JWTs described here and in {{I-D.skyfire-oauth-kyapay-token}}, implementers MUST follow the JSON Web Token Best Current Practices {{RFC8725}}, in addition to the validation steps of {{I-D.skyfire-oauth-kyapay-token}}, Section 4.

## Transport Confidentiality

KYAPay tokens convey identity and, for PAY tokens, payment credentials.
They MUST be transmitted over TLS and MUST NOT be sent in the clear.

## Token Freshness and Lifetime

A long-lived token is a credential that can be captured, farmed, and resold for reuse against the same Target.
The `exp` claim MUST be present;
verifiers SHOULD require short lifetimes (for high-assurance actions, on the order of a few minutes) and MUST reject tokens whose lifetime exceeds their local policy maximum.
Verifiers SHOULD reject tokens whose `iat` is outside an acceptable window.

## Replay and Proof of Possession

A bare bearer token can be replayed by anyone who captures it within its validity window.
As deployed today ({{TrustStack}}), KYAPay accepts this bounded in-window risk and mitigates rather than eliminates it: verifiers MUST validate the `aud` claim (and `tdm`/`tsi` where used) so that a token minted for one Target cannot be presented to another, MUST keep accepted lifetimes short, and SHOULD use the `jti` claim to detect replay to the same recipient.
Where a token carries a proof-of-possession key (`cnf` {{RFC7800}}), verifiers MUST additionally verify a per-request signature {{RFC9421}} over that key ({{ProcModel}}), which removes the in-window replay exposure.
Access tokens produced by token exchange ({{ATOProt}}) SHOULD be sender-constrained ({{RFC9449}}, {{RFC8705}}).

## Malicious or Compromised Target Reuse

A Target that legitimately receives a bearer token can, within the token's window, reuse it to act elsewhere on the agent's behalf.
Audience binding limits this to the intended Target;
the proof-of-possession model ({{TrustStack}}), which binds each use to a fresh request signature, closes it.
Verifiers and Targets MUST NOT forward received tokens to other parties (see {{PrivCon}}).

## Compromised Agent Host (Irreducible)

Malware resident on the agent's host can drive the agent's legitimate key as a signing oracle for as long as it is present.
Proof of possession does not prevent this: the malware's requests convey valid bindings and are indistinguishable from the real agent's.
This risk is bounded not by the token bindings but by short token lifetimes -- so anything signed during a compromise expires quickly once the malware is evicted -- together with host-level controls such as attestation, anomaly detection, and hygiene.

## Issuer Trust

A token is only as trustworthy as its issuer and the verifiers it cites.
The system is federated: many issuers are possible, and a verifier MUST maintain an explicit set of trusted issuers and the claims and assurance levels it will accept from each.
Establishing this trust at scale is an open problem analogous to the Certificate Authority model (audits, a maintained issuer list, a removal mechanism, and possibly transparency logs);
this document does not define such a framework, and deployments should not assume one exists.
Until it does, trust is established out of band (for example, configured relationships with well-known issuers).

## Revocation

An agent whose platform is deregistered, or a human whose identity verification is revoked, may still hold unexpired tokens.
This specification does not define a revocation mechanism.
Verifiers SHOULD keep accepted lifetimes short to bound exposure, and SHOULD consult an issuer's live-status or revocation endpoint, where one is offered, for high-assurance or high-value actions.

## Continuity of Human Control

A valid token attests that the human principal authorized the agent at the time of issuance.
It does not prove the human remains in control -- for example, if the agent is later prompt-injected, or if credentials are exfiltrated from the agent or its host.
Verifiers SHOULD treat high-value actions as warranting step-up ({{StepUp}}) rather than relying solely on a previously issued token.

## Key Management

If an issuer or platform rotates a signing key, tokens signed with the old key may fail validation prematurely.
Issuers SHOULD retain superseded verification keys in their JWK Set for at least the maximum token lifetime after rotation.

# Privacy Considerations {#PrivCon}

The privacy considerations of {{I-D.skyfire-oauth-kyapay-token}} apply.
In particular:

## Minimal Disclosure

Only the information needed to facilitate the intended interaction should be placed in a token and conveyed to intermediaries and Targets.
Issuers and senders SHOULD prefer the least revealing set of claims sufficient for the decision at hand -- for example, an assurance level or a boolean verified status rather than underlying identity documents or personally identifying verification data.
Because KYA tokens are JWTs, selective disclosure (for example, SD-JWT {{RFC9901}}) is a forward-compatible path to revealing only the claims a given Target needs.

## Consent

Tokens assert that an agent acts on behalf of a principal.
That assertion is only legitimate when the principal has authorized the interaction.
Systems SHOULD be designed so that tokens are minted only with the principal's authorization.

## Handling by Intermediaries

Security intermediaries receive verified personal data in the course of making decisions.
They MUST treat `KYAPay-Token` values and their decoded contents as sensitive personal data: not caching them for replay, not logging them in the clear, and not sharing them with parties other than legitimate participants in the interaction.

# IANA Considerations

## HTTP Field Name Registration

IANA is requested to register the following entry in the "Hypertext Transfer Protocol (HTTP) Field Name" registry established by {{RFC9110}}:

* Field Name: KYAPay-Token
* Status: permanent
* Structured Type: List
* Reference: {{HeaderField}} of this document
* Comments: Carries one or more KYAPay tokens in an HTTP request

## JWT and Media Type Registrations

This document defines no new JWT claims, JWT confirmation methods, or media types.
The claims and media types used by KYAPay tokens are registered by {{I-D.skyfire-oauth-kyapay-token}}.
The `amr` extensions and the `ivm` and `aml` claims and values are registered by
{{I-D.skyfire-oauth-amr-values}},
{{I-D.skyfire-oauth-id-verification}}, and
{{I-D.skyfire-oauth-aml-methods}}, respectively.

--- back

# Acknowledgments
{:numbered="false"}

The authors thank the contributors to the KYAPay Token {{I-D.skyfire-oauth-kyapay-token}} specification and the partners in the KYAPay consortium (see {{KYAPAY-ORG}}) -- including bot-management, fraud, CIAM, and ATO vendors and merchants -- whose deployment and review experience informed the usage patterns and open issues described here.

# Document History
{: numbered="false"}

[[ to be removed by the RFC Editor before publication as an RFC ]]

-00

* Initial draft.
