---
title: "Verifiable Presentation Profile for Identity Assertion JWT Authorization Grant"
abbrev: "VP Profile for ID-JAG"
category: std

docname: draft-parecki-oauth-id-jag-vp-latest
submissiontype: IETF
number:
date:
consensus: true
v: 3
area: "Security"
workgroup: "Web Authorization Protocol"
keyword:
 - cross-app-access
 - authorization
 - assertion
 - verifiable-credentials
 - verifiable-presentations
 - wallet
venue:
  group: "Web Authorization Protocol"
  type: "Working Group"
  mail: "oauth@ietf.org"
  arch: "https://mailarchive.ietf.org/arch/browse/oauth/"
  github: "aaronpk/draft-parecki-oauth-id-jag-vp"

author:
 -
    fullname: Aaron Parecki
    organization: Okta
    email: aaron@parecki.com

normative:
  RFC6749:
  RFC7519:
  RFC7521:
  RFC7523:
  RFC8414:
  RFC8693:
  RFC8725:
  I-D.ietf-oauth-identity-assertion-authz-grant:
  I-D.ietf-oauth-sd-jwt-vc:
  IANA.oauth-parameters:

informative:
  RFC7591:
  OpenID4VP:
    title: "OpenID for Verifiable Presentations - draft 28"
    target: https://openid.net/specs/openid-4-verifiable-presentations-1_0.html
    author:
      - ins: O. Terbu
      - ins: T. Lodderstedt
      - ins: K. Yasuda
      - ins: T. Looker
  OpenID4VCI:
    title: "OpenID for Verifiable Credential Issuance"
    target: https://openid.net/specs/openid-4-verifiable-credential-issuance-1_0.html
    author:
      - ins: T. Lodderstedt
      - ins: K. Yasuda
      - ins: T. Looker

--- abstract

This specification profiles the Identity Assertion JWT Authorization Grant {{I-D.ietf-oauth-identity-assertion-authz-grant}} to accept a Verifiable Presentation as the subject token in a Token Exchange request. It enables a Client to use a Verifiable Digital Credential held in an End-User's digital wallet to obtain an Identity Assertion JWT Authorization Grant (ID-JAG) from the credential issuer, which the Client can then exchange for an OAuth 2.0 access token at a Resource Authorization Server. The Resource Authorization Server processes only an ID-JAG and is not required to process Verifiable Credentials.

--- middle


# Introduction

The Identity Assertion JWT Authorization Grant (ID-JAG) {{I-D.ietf-oauth-identity-assertion-authz-grant}} defines a Cross-App Access (XAA) pattern in which an Identity Provider (IdP) Authorization Server brokers cross-domain access to a Resource Authorization Server's APIs on behalf of a signed-in End-User. In the base specification, the IdP Authorization Server obtains identity through OpenID Connect or SAML 2.0 single sign-on, and the End-User's identity is conveyed to the IdP as an OpenID Connect ID Token or a SAML 2.0 Assertion presented as the `subject_token` in a Token Exchange {{RFC8693}} request.

In the digital wallet ecosystem, an End-User's identity may instead be represented by a Verifiable Digital Credential (VDC) issued to a wallet by a Credential Issuer (for example, using OpenID for Verifiable Credential Issuance {{OpenID4VCI}}). Rather than authenticating to an IdP through an interactive sign-on flow, the End-User authorizes their wallet to release a Verifiable Presentation (VP) of one or more credentials to the Client, typically using OpenID for Verifiable Presentations {{OpenID4VP}}.

This specification profiles the base ID-JAG flow to accept a Verifiable Presentation as the `subject_token` in the Token Exchange request to the IdP Authorization Server. In this profile, **the Credential Issuer is the IdP Authorization Server** ({{trust-model}}). The Client presents a VP of a credential issued by that Credential Issuer back to the same Credential Issuer's Token Endpoint to obtain an ID-JAG. The Client then redeems the ID-JAG at the Resource Authorization Server using the same JWT Bearer ({{RFC7523}}) flow defined in the base specification.

A key property of this design is that **the Resource Authorization Server is not required to process Verifiable Credentials**. The VP is consumed only by the IdP Authorization Server (which is also the Credential Issuer). The Resource Authorization Server validates an ID-JAG using the same processing rules defined in {{Section 6.4 of I-D.ietf-oauth-identity-assertion-authz-grant}} that it would apply to any other ID-JAG. This mirrors the role of the SAML 2.0 Subject Token Interoperability section of the base specification, which bridges SAML to OAuth at the IdP boundary so that Resource Authorization Servers do not need to process XML.

This specification is a profile of {{I-D.ietf-oauth-identity-assertion-authz-grant}} and references it normatively for all unchanged behavior. Only the substitution of the `subject_token` and the validation rules specific to a Verifiable Presentation are defined here.


# Conventions and Definitions

{::boilerplate bcp14-tagged}

Common OAuth and JWT terms, and ID-JAG-specific terms such as Client, IdP Authorization Server, Resource Authorization Server, Resource Server, Identity Assertion, ID-JAG, and Subject Resolution, are used as defined in {{I-D.ietf-oauth-identity-assertion-authz-grant}}.

This specification additionally uses the following terms:

Verifiable Digital Credential (VDC)
: A cryptographically protected set of claims about a subject, issued to a digital wallet by a Credential Issuer. Examples include credentials in the SD-JWT VC format {{I-D.ietf-oauth-sd-jwt-vc}} and other formats supported by Verifiable Presentation profiles.

Verifiable Presentation (VP)
: A cryptographically protected set of claims derived from one or more Verifiable Digital Credentials and presented by a Wallet to a Verifier on behalf of an End-User, typically using OpenID for Verifiable Presentations {{OpenID4VP}}. A VP carries proof of holder binding (such as a Key Binding JWT for SD-JWT VC) that ties the presentation to the wallet that holds the credential.

Wallet
: Software, controlled by the End-User, that holds Verifiable Digital Credentials and creates Verifiable Presentations on the End-User's behalf.

Credential Issuer
: The party that issues a Verifiable Digital Credential to a Wallet. In this profile, the Credential Issuer also acts as the IdP Authorization Server defined in {{I-D.ietf-oauth-identity-assertion-authz-grant}}.

Verifier
: The party that receives a Verifiable Presentation from a Wallet and validates it. In this profile, the Client is the Verifier in the Wallet-to-Client interaction; the IdP Authorization Server independently validates the VP it receives as a `subject_token`. See {{vp-audience}}.


# Trust Model {#trust-model}

This profile preserves the trust pattern established by {{I-D.ietf-oauth-identity-assertion-authz-grant}}: the Resource Authorization Server trusts a single party (the IdP Authorization Server) to issue ID-JAGs that it will accept, and the Client has independent registrations with that same party and with the Resource Authorization Server. The substitution introduced by this profile is that the End-User's identity is conveyed to the IdP Authorization Server through a wallet-presented Verifiable Presentation rather than through an SSO-issued Identity Assertion.

The following relationships exist in this profile:

* **Client to IdP Authorization Server (Credential Issuer)** — The Client is registered as an OAuth 2.0 client with the IdP Authorization Server's Token Endpoint and authenticates as a confidential client when making the Token Exchange request. The Client also acts as a Verifier when requesting a Verifiable Presentation from the End-User's wallet, typically using {{OpenID4VP}}. How the wallet recognizes and authorizes a request from the Client (for example, through wallet-side trust frameworks or client identifier schemes defined by {{OpenID4VP}}) is out of scope of this specification.

* **Resource Authorization Server to IdP Authorization Server (Credential Issuer)** — The Resource Authorization Server trusts the IdP Authorization Server to issue ID-JAGs for the End-Users it credentials. This trust relationship is unchanged from {{I-D.ietf-oauth-identity-assertion-authz-grant}}; the Resource Authorization Server does not need to be aware that the IdP Authorization Server is also a Credential Issuer.

* **Client to Resource Authorization Server** — The Client is registered as an OAuth 2.0 client with the Resource Authorization Server. This relationship is unchanged from {{I-D.ietf-oauth-identity-assertion-authz-grant}}.

* **Wallet to Credential Issuer** — The Wallet holds a Verifiable Digital Credential previously issued by the Credential Issuer (for example, using {{OpenID4VCI}}). This relationship is established prior to and outside the scope of this specification.

The crucial implication of identifying the Credential Issuer with the IdP Authorization Server is that VP validation is trivial in the cryptographic sense: the IdP is validating a credential that it issued itself. Federation among multiple Credential Issuers — that is, an IdP Authorization Server accepting VPs of credentials issued by a third-party Credential Issuer — is out of scope of this specification and may be defined in a future profile.


# Verifiable Presentation Subject Token {#vp-subject-token}

A Verifiable Presentation Subject Token is a Verifiable Presentation, as defined by {{OpenID4VP}}, encoded as a UTF-8 string suitable for use as the value of the `subject_token` parameter in a Token Exchange request {{RFC8693}}.

The following Verifiable Presentation format is REQUIRED for compliance with this specification:

* SD-JWT VC presentations as defined in {{I-D.ietf-oauth-sd-jwt-vc}}, including a Key Binding JWT (KB-JWT) bound to the IdP Authorization Server's Token Endpoint use of the presentation as described in {{vp-audience}}.

Other Verifiable Presentation formats MAY be supported by IdP Authorization Servers that implement this specification. Format-specific processing rules for additional formats are out of scope of this specification.

The Verifiable Presentation Subject Token Type is identified by the URI:

`urn:ietf:params:oauth:token-type:vp`

This URI is registered by this specification (see {{iana}}).


# Token Exchange Request {#token-exchange}

The Client makes a Token Exchange request to the IdP Authorization Server's Token Endpoint as defined in {{Section 5.2 of I-D.ietf-oauth-identity-assertion-authz-grant}}, with the following changes:

`subject_token`:
: REQUIRED - A Verifiable Presentation Subject Token as defined in {{vp-subject-token}}.

`subject_token_type`:
: REQUIRED - The value `urn:ietf:params:oauth:token-type:vp`.

All other Token Exchange request parameters are unchanged from {{Section 5.2 of I-D.ietf-oauth-identity-assertion-authz-grant}}, including `grant_type`, `requested_token_type`, `audience`, `resource`, `scope`, and `authorization_details`. Client authentication to the IdP Authorization Server is unchanged.

The following non-normative example shows a Token Exchange request using a Verifiable Presentation as the `subject_token` and a JWT Bearer Assertion {{RFC7523}} for client authentication (presentation and client assertion truncated for brevity):

    POST /oauth2/token HTTP/1.1
    Host: issuer.example
    Content-Type: application/x-www-form-urlencoded

    grant_type=urn:ietf:params:oauth:grant-type:token-exchange
    &requested_token_type=urn:ietf:params:oauth:token-type:id-jag
    &audience=https://acme.chat.example/
    &resource=https://api.chat.example/
    &scope=chat.read+chat.history
    &subject_token=eyJhbGciOiJFUzI1NiIsImtpZCI6IjEifQ.eyJ2Y3...~WyJzYWx0...~eyJ0eXAiOiJrYitqd3QiLCJhbGciOiJFUzI1NiJ9.ey...
    &subject_token_type=urn:ietf:params:oauth:token-type:vp
    &client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
    &client_assertion=eyJhbGciOiJSUzI1NiIsImtpZCI6IjIyIn0...

How the Client obtains the Verifiable Presentation from the End-User's wallet is out of scope of this specification. {{OpenID4VP}} is the typical mechanism. The Client SHOULD request only the Verifiable Credential and claims required by the IdP Authorization Server to issue the requested ID-JAG.


# IdP Authorization Server Processing Rules {#processing-rules}

The IdP Authorization Server processes the Token Exchange request as defined in {{Section 5.2.3 of I-D.ietf-oauth-identity-assertion-authz-grant}}, with the following additional rules applied to the validation of the `subject_token` when its type is `urn:ietf:params:oauth:token-type:vp`.

The IdP Authorization Server MUST validate the Verifiable Presentation:

1. **Format**. Parse the Verifiable Presentation according to the format indicated by its serialization. Reject the request with an `invalid_grant` error if the format is not supported.

2. **Credential Issuer**. Validate the cryptographic protection of the Verifiable Credential carried by the Verifiable Presentation. The Verifiable Credential MUST have been issued by this IdP Authorization Server acting as a Credential Issuer. If the Verifiable Credential was issued by a different Credential Issuer, the IdP Authorization Server MUST reject the request with an `invalid_grant` error. Federated trust across multiple Credential Issuers is out of scope of this specification.

3. **Holder binding**. Validate the holder binding of the Verifiable Presentation according to the format. For SD-JWT VC, this means validating the Key Binding JWT (KB-JWT) signature against the key bound to the credential's `cnf` claim, and validating the KB-JWT's `iat`, `nonce`, and `aud` claims as described in {{vp-audience}} and {{vp-replay}}.

4. **Audience binding**. Validate that the Verifiable Presentation is bound to the authenticated Client. See {{vp-audience}}.

5. **Freshness**. Validate that the holder binding proof is fresh and has not been replayed. See {{vp-replay}}.

6. **Subject claims**. Process the disclosed claims of the Verifiable Presentation to determine the End-User identifiers and other identity claims to be carried in the issued ID-JAG. The IdP Authorization Server MUST be able to map the subject of the Verifiable Credential to the same End-User the IdP Authorization Server would identify with a `sub` claim in an ID Token. Because the IdP Authorization Server is also the Credential Issuer, it has authoritative knowledge of this mapping.

If any of the above validations fail, the IdP Authorization Server MUST reject the request with an `invalid_grant` error as defined in {{Section 5.2 of RFC6749}}.

If validation succeeds, the IdP Authorization Server proceeds with policy evaluation, scope and resource processing, and ID-JAG issuance as defined in {{Section 5.2.3 of I-D.ietf-oauth-identity-assertion-authz-grant}}. The issued ID-JAG and the Token Exchange response are unchanged by this profile.


## Audience Binding of the Verifiable Presentation {#vp-audience}

The Verifiable Presentation MUST be bound to the IdP Authorization Server's Token Endpoint as the audience of the holder binding proof. For SD-JWT VC, this means the KB-JWT's `aud` claim MUST be the value the IdP Authorization Server publishes for this purpose, typically the IdP Authorization Server's Token Endpoint URL or its issuer identifier as defined in {{RFC8414}}.

In addition, the IdP Authorization Server MUST validate that the OpenID for Verifiable Presentations Verifier identifier carried by the Verifiable Presentation, where applicable to the format, identifies the authenticated Client. This is the analog of the validation defined in {{Section 5.2.3 of I-D.ietf-oauth-identity-assertion-authz-grant}} that requires the audience of an ID Token or SAML Assertion `subject_token` to match the `client_id` of the authenticated Client.

This binding prevents a Client from substituting a Verifiable Presentation that was obtained for a different Verifier as the `subject_token` in a Token Exchange request authenticated as itself. See {{security-substitution}}.

How the Client and the IdP Authorization Server agree on the value of the audience of the holder binding proof, and on the encoding of the Verifier identifier within the Verifiable Presentation, is determined by the Verifiable Presentation profile in use (for example, {{OpenID4VP}}) and is out of scope of this specification.


## Replay Protection {#vp-replay}

The IdP Authorization Server MUST protect against replay of Verifiable Presentations. The mechanism depends on the Verifiable Presentation format and the Verifiable Presentation profile in use.

For SD-JWT VC presentations, the IdP Authorization Server SHOULD provide a fresh `nonce` value to the Client for inclusion in the KB-JWT, and MUST validate that the `nonce` returned in the KB-JWT was issued by the IdP Authorization Server, has not been used before, and is unexpired. The IdP Authorization Server MAY rely on a short `iat` window in combination with a server-bound nonce, or on other equivalent mechanisms appropriate to the deployment.

How the Client obtains the nonce from the IdP Authorization Server is out of scope of this specification. {{OpenID4VP}} defines mechanisms suitable for this purpose.


# Token Exchange Response

If the IdP Authorization Server validates the Verifiable Presentation and grants the request, it issues an ID-JAG and returns it in the Token Exchange response defined in {{Section 5.2.4 of I-D.ietf-oauth-identity-assertion-authz-grant}}. This response is unchanged by this profile.

The Client then exchanges the ID-JAG for an access token at the Resource Authorization Server using the JWT Bearer Authorization Grant flow defined in {{Section 6 of I-D.ietf-oauth-identity-assertion-authz-grant}}. The Resource Authorization Server's processing of the ID-JAG is unchanged.


# Refresh Token Bridge {#refresh-token-bridge}

A Client MAY exchange a Verifiable Presentation Subject Token for a Refresh Token instead of an ID-JAG. This allows the Client to obtain subsequent ID-JAGs without prompting the End-User to release a new Verifiable Presentation each time, which can be disruptive to the End-User experience because it typically requires interaction with the wallet.

This is the analog of the SAML 2.0 to Refresh Token bridge defined in {{Section 6.5 of I-D.ietf-oauth-identity-assertion-authz-grant}} and uses the same Token Exchange request shape, with `requested_token_type=urn:ietf:params:oauth:token-type:refresh_token` and a Verifiable Presentation as the `subject_token`.

To request a Refresh Token, the Client makes a Token Exchange request to the IdP Authorization Server's Token Endpoint with the following parameters:

`grant_type`:
: REQUIRED - `urn:ietf:params:oauth:grant-type:token-exchange`.

`requested_token_type`:
: REQUIRED - `urn:ietf:params:oauth:token-type:refresh_token`.

`subject_token`:
: REQUIRED - A Verifiable Presentation Subject Token as defined in {{vp-subject-token}}.

`subject_token_type`:
: REQUIRED - `urn:ietf:params:oauth:token-type:vp`.

`scope`:
: OPTIONAL - The OpenID Connect scopes `openid offline_access` SHOULD be requested when requesting a Refresh Token from the IdP Authorization Server. Additional scopes are allowed.

The IdP Authorization Server validates the Verifiable Presentation as defined in {{processing-rules}}. If validation succeeds and the IdP Authorization Server is willing to issue a Refresh Token, it returns the Refresh Token in the Token Exchange response.

The following non-normative example shows the request and response:

    POST /oauth2/token HTTP/1.1
    Host: issuer.example
    Content-Type: application/x-www-form-urlencoded

    grant_type=urn:ietf:params:oauth:grant-type:token-exchange
    &requested_token_type=urn:ietf:params:oauth:token-type:refresh_token
    &scope=openid+offline_access
    &subject_token=eyJhbGciOiJFUzI1NiIsImtpZCI6IjEifQ...
    &subject_token_type=urn:ietf:params:oauth:token-type:vp
    &client_assertion_type=urn:ietf:params:oauth:client-assertion-type:jwt-bearer
    &client_assertion=eyJhbGciOiJSUzI1NiIsImtpZCI6IjIyIn0...

    HTTP/1.1 200 OK
    Content-Type: application/json
    Cache-Control: no-store
    Pragma: no-cache

    {
      "issued_token_type": "urn:ietf:params:oauth:token-type:refresh_token",
      "access_token": "vF9dft4qmTcXkZ26zL8b6u",
      "token_type": "N_A",
      "scope": "openid offline_access",
      "expires_in": 1209600
    }

The Client can then use the Refresh Token as the `subject_token` in subsequent Token Exchange requests with `requested_token_type=urn:ietf:params:oauth:token-type:id-jag` and `subject_token_type=urn:ietf:params:oauth:token-type:refresh_token`, as defined in {{Section 5.2.2 of I-D.ietf-oauth-identity-assertion-authz-grant}}.


# Authorization Server Metadata {#metadata}

This specification extends the IdP Authorization Server metadata defined in {{Section 7.1 of I-D.ietf-oauth-identity-assertion-authz-grant}} and {{RFC8414}} with a new metadata parameter for advertising support for Verifiable Presentation Subject Tokens.

`identity_chaining_subject_token_types_supported`:
: OPTIONAL - JSON array of strings indicating the `subject_token_type` values the IdP Authorization Server accepts in Token Exchange requests for identity chaining.

To advertise support for Verifiable Presentation Subject Tokens, the IdP Authorization Server SHOULD include the value `urn:ietf:params:oauth:token-type:vp` in this array.

The Resource Authorization Server metadata defined in {{Section 7.2 of I-D.ietf-oauth-identity-assertion-authz-grant}} is unchanged. A Resource Authorization Server that supports the Identity Assertion JWT Authorization Grant profile by definition supports ID-JAGs that originated through this profile, because the Resource Authorization Server does not process the Verifiable Presentation directly.


# Use Case: Consumer Cross-App Access with a Wallet-Held Credential

The following non-normative example illustrates how this profile enables Cross-App Access in a consumer scenario.

A consumer service operator (`https://consumer-id.example/`) operates a Credential Issuer that issues Verifiable Digital Credentials to its End-Users' digital wallets, and also operates the IdP Authorization Server defined in this profile at the same origin. Two unrelated SaaS providers operate Resource Authorization Servers (`https://music.example/` and `https://photos.example/`) that have each independently established a trust relationship with `https://consumer-id.example/` for ID-JAGs.

A productivity application (the Client, `https://productivity-app.example/`) is registered as an OAuth client at `https://consumer-id.example/` and at both Resource Authorization Servers.

When the End-User wants the productivity application to access the music and photos services on their behalf:

1. The productivity application requests a Verifiable Presentation from the End-User's wallet using {{OpenID4VP}}, identifying itself as the Verifier.

2. The wallet, after End-User consent, returns a Verifiable Presentation of a credential previously issued by `https://consumer-id.example/`, with a Key Binding JWT bound to the productivity application as the OpenID for Verifiable Presentations Verifier and to the IdP Authorization Server's Token Endpoint as the audience of the holder binding.

3. The productivity application makes a Token Exchange request to the IdP Authorization Server at `https://consumer-id.example/`, presenting the Verifiable Presentation as the `subject_token` with `subject_token_type=urn:ietf:params:oauth:token-type:vp`, and requesting an ID-JAG with `audience=https://music.example/`.

4. The IdP Authorization Server validates the Verifiable Presentation as the Credential Issuer of the underlying credential, validates the holder binding and audience binding to the productivity application, evaluates policy, and issues an ID-JAG.

5. The productivity application exchanges the ID-JAG for an access token at `https://music.example/` as defined in {{Section 6 of I-D.ietf-oauth-identity-assertion-authz-grant}}. `https://music.example/` validates the ID-JAG using the same trust relationship it would use for any other ID-JAG from `https://consumer-id.example/`, without processing any Verifiable Credential.

6. To access `https://photos.example/`, the productivity application MAY repeat the Token Exchange step with a fresh Verifiable Presentation, OR it MAY have previously obtained a Refresh Token via the bridge defined in {{refresh-token-bridge}} and use that Refresh Token to obtain a new ID-JAG with `audience=https://photos.example/`, avoiding a second wallet interaction.


# Security Considerations

## Client Authentication {#security-client-auth}

This profile inherits all security considerations from {{Section 8 of I-D.ietf-oauth-identity-assertion-authz-grant}}, including the requirement that this profile be supported only for confidential clients. Public clients SHOULD instead use an interactive authorization code flow with the Resource Authorization Server.


## Verifiable Presentation Substitution {#security-substitution}

A Client could attempt to substitute a Verifiable Presentation issued for a different Verifier as the `subject_token` in a Token Exchange request authenticated as itself. The audience binding requirements in {{vp-audience}} prevent this attack: the IdP Authorization Server MUST validate that the Verifiable Presentation's holder binding proof is bound to the IdP Authorization Server as audience AND that the OpenID for Verifiable Presentations Verifier identifier carried by the Verifiable Presentation identifies the authenticated Client.

This is the analog of the requirement in {{Section 5.2.3 of I-D.ietf-oauth-identity-assertion-authz-grant}} that the audience of an ID Token or SAML Assertion `subject_token` match the `client_id` of the authenticated Client.


## Replay of Verifiable Presentations

A Verifiable Presentation contains material that, if presented again, may be accepted by an IdP Authorization Server. The IdP Authorization Server MUST implement replay protection as described in {{vp-replay}}. Implementers SHOULD prefer server-issued nonces over reliance on `iat` windows alone, and SHOULD limit the validity window of accepted holder binding proofs.


## Wallet Authenticity

Whether a Wallet is genuine, attested, or running on an attested device is out of scope of this specification. Deployments where wallet authenticity matters SHOULD rely on wallet attestation mechanisms defined by their Verifiable Presentation profile (for example, those provided by {{OpenID4VP}}).


## Holder Binding Scope

Holder binding established by the Verifiable Presentation terminates at the IdP Authorization Server. The IdP Authorization Server validates holder binding when accepting the `subject_token`; the resulting ID-JAG is not, by virtue of this profile alone, sender-constrained to the wallet's key. If sender constraining of the ID-JAG or the resulting access token is desired, the existing mechanisms defined in {{Section 8.6 of I-D.ietf-oauth-identity-assertion-authz-grant}} (for example, DPoP-based key binding established between the Client and the IdP Authorization Server) apply unchanged. Defining a mechanism to bridge wallet-held holder binding into the ID-JAG `cnf` claim is out of scope of this specification.


## Selective Disclosure

Verifiable Presentation formats such as SD-JWT VC support selective disclosure of credential claims. The Client requests claims from the Wallet that are necessary for the IdP Authorization Server to issue the requested ID-JAG. The IdP Authorization Server MUST be able to perform Subject Resolution from the disclosed claims to populate the `sub` claim and any other required claims of the issued ID-JAG.

The Client SHOULD NOT request, and the Wallet SHOULD NOT release, claims beyond those required by the IdP Authorization Server to issue the requested ID-JAG. Claims disclosed beyond the minimum required may be visible to the Client, which may not be desirable depending on the deployment.


## Federation Across Credential Issuers

This specification scopes the trust relationship to a single Credential Issuer that is the same party as the IdP Authorization Server. An IdP Authorization Server MUST NOT accept a Verifiable Presentation of a credential issued by a different Credential Issuer under this profile. Profiles that extend this specification to support federation across Credential Issuers need to define how the IdP Authorization Server establishes trust in third-party Credential Issuers, how Subject Resolution works across Credential Issuers, and how to mitigate impersonation and downgrade attacks.


# IANA Considerations {#iana}

## OAuth URI Registration

This section registers `urn:ietf:params:oauth:token-type:vp` in the "OAuth URI" subregistry of the "OAuth Parameters" registry {{IANA.oauth-parameters}}.

* URN: `urn:ietf:params:oauth:token-type:vp`
* Common Name: Token type URI for a Verifiable Presentation Subject Token
* Change Controller: IETF
* Specification Document: This document

## OAuth Authorization Server Metadata Registration

This section registers `identity_chaining_subject_token_types_supported` in the "OAuth Authorization Server Metadata" registry of the "OAuth Parameters" registry {{IANA.oauth-parameters}}.

* Metadata Name: `identity_chaining_subject_token_types_supported`
* Metadata Description: JSON array of `subject_token_type` values the IdP Authorization Server accepts in Token Exchange requests for identity chaining
* Change Controller: IETF
* Specification Document: {{metadata}}


--- back

# Acknowledgments
{:numbered="false"}

The authors thank the OAuth Working Group and the OpenID for Verifiable Credentials community for ongoing discussions that informed this profile.


# Document History
{:numbered="false"}

\[\[ To be removed from the final specification ]]

-00

* Initial revision
