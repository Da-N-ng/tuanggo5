---
title: "Privacy Pass Issuance Protocol"
abbrev: PP Issuance
docname: draft-ietf-privacypass-protocol-latest
date:
category: info

ipr: trust200902
keyword: Internet-Draft

stand_alone: yes
pi: [toc, sortrefs, symrefs]

author:
 -
    ins: S. Celi
    name: Sofía Celi
    org: Cloudflare
    city: Lisbon
    country: Portugal
    email: sceli@cloudflare.com
 -
    ins: A. Davidson
    name: Alex Davidson
    org: Brave Software
    city: Lisbon
    country: Portugal
    email: alex.davidson92@gmail.com
 -
    ins: A. Faz-Hernandez
    name: Armando Faz-Hernandez
    org: Cloudflare
    street: 101 Townsend St
    city: San Francisco
    country: United States of America
    email: armfazh@cloudflare.com
 -
    ins: S. Valdez
    name: Steven Valdez
    org: Google LLC
    email: svaldez@chromium.org
 -
    ins: C. A. Wood
    name: Christopher A. Wood
    org: Cloudflare
    street: 101 Townsend St
    city: San Francisco
    country: United States of America
    email: caw@heapingbits.net

normative:
  RFC2119:
  HTTP-Authentication:
    title: The Privacy Pass HTTP Authentication Scheme
    target: https://datatracker.ietf.org/doc/html/draft-pauly-privacypass-auth-scheme-00
  I-D.ietf-privacypass-architecture:

--- abstract

This document specifies two variants of the the two-message issuance protocol
for Privacy Pass tokens: one that produces tokens that are privately
verifiable, and another that produces tokens that are publicly verifiable.
The privately verifiable issuance protocol optionally supports public
metadata during the issuance flow.

--- middle

# Introduction

The Privacy Pass protocol provides a privacy-preserving authorization
mechanism. In essence, the protocol allows clients to provide
cryptographic tokens that prove nothing other than that they have been
created by a given server in the past {{I-D.ietf-privacypass-architecture}}.

This document describes the issuance protocol for Privacy Pass. It specifies
two variants: one that is privately verifiable based on the oblivious
pseudorandom function from {{!OPRF=I-D.irtf-cfrg-voprf}}, and one that is
publicly verifiable based on the blind RSA signature scheme
{{!BLINDRSA=I-D.irtf-cfrg-rsa-blind-signatures}}.

This document DOES NOT cover the architectural framework required for
running and maintaining the Privacy Pass protocol in the Internet
setting. In addition, it DOES NOT cover the choices that are necessary
for ensuring that client privacy leaks do not occur. Both of these
considerations are covered in {{I-D.ietf-privacypass-architecture}}.

# Terminology

{::boilerplate bcp14}

The following terms are used throughout this document.

- Client: An entity that provides authorization tokens to services
  across the Internet, in return for authorization.
- Issuer: A service produces Privacy Pass tokens to clients.
- Private Key: The secret key used by the Issuer for issuing tokens.
- Public Key: The public key used by the Issuer for issuing and verifying
  tokens.

We assume that all protocol messages are encoded into raw byte format
before being sent across the wire.

# Configuration {#setup}

Issuers MUST provide one parameter for configuration:

1. Issuer Request URI: a token request URL for generating access tokens.
   For example, an Issuer URL might be https://issuer.example.net/example-token-request.
   This parameter uses resource media type "text/plain".

The Issuer parameters can be obtained from an Issuer via a directory object, which is a JSON
object whose field names and values are raw values and URLs for the parameters.

| Field Name           | Value                                            |
|:---------------------|:-------------------------------------------------|
| issuer-request-uri   | Issuer Request URI resource URL as a JSON string |

As an example, the Issuer's JSON directory could look like:

~~~
 {
    "issuer-request-uri": "https://issuer.example.net/example-token-request"
 }
~~~

Issuer directory resources have the media type "application/json"
and are located at the well-known location /.well-known/token-issuer-directory.

# Token Challenge Requirements

Clients receive challenges for tokens, as described in {{AUTHSCHEME}}.
The basic token issuance protocols described in this document can be
interactive or non-interactive, and per-origin or cross-origin.

# Issuance Protocol for Privately Verifiable Tokens with Public Metadata {#private-flow}

The Privacy Pass issuance protocol is a two message protocol that takes
as input a challenge from the redemption protocol and produces a token,
as shown in the figure below.

~~~
   Origin          Client                   Issuer
                (pkI, info)            (skI, pkI, info)
                  +------------------------------------\
  Challenge   ----> TokenRequest ------------->        |
                  |                       (evaluate)   |
    Token    <----+     <--------------- TokenResponse |
                  \------------------------------------/
~~~

Issuers provide a Private and Public Key, denoted skI and pkI, respectively,
used to produce tokens as input to the protocol. See {{issuer-configuration}}
for how this key pair is generated.

Clients provide the following as input to the issuance protocol:

- Issuer name, identifying the Issuer. This is typically a host name that
  can be used to construct HTTP requests to the Issuer.
- Issuer Public Key pkI, with a key identifier `key_id` computed as
  described in {{issuer-configuration}}.
- Challenge value `challenge`, an opaque byte string. For example, this might
  be provided by the redemption protocol in {{HTTP-Authentication}}.

Both Client and Issuer also share a common public string called `info`.

Given this configuration and these inputs, the two messages exchanged in
this protocol are described below. This section uses notation described in
{{OPRF, Section 4}}, including SerializeElement and DeserializeElement,
SerializeScalar and DeserializeScalar, and DeriveKeyPair.

## Client-to-Issuer Request {#client-to-issuer}

The Client first creates a context as follows:

~~~
client_context = SetupPOPRFClient(0x0004, pkI)
~~~

Here, 0x0004 is the two-octet identifier corresponding to the
OPRF(P-384, SHA-384) ciphersuite in {{OPRF}}. SetupPOPRFClient
is defined in {{OPRF, Section 3.2}}.

The Client then creates an issuance request message for a random value `nonce`
using the input challenge and Issuer key identifier as follows:

~~~
nonce = random(32)
context = SHA256(challenge)
token_input = concat(0x0001, nonce, context, key_id)
blind, blinded_msg, tweaked_key = client_context.Blind(nonce, info)
~~~

The Blind function is defined in {{OPRF, Section 3.3.3}}.
If the Blind function fails, the Client aborts the protocol. Otherwise,
the Client then creates a TokenRequest structured as follows:

~~~
struct {
   uint16_t token_type = 0x0001;
   uint8_t token_key_id;
   uint8_t blinded_request[Ne];
} TokenRequest;
~~~

The structure fields are defined as follows:

- "token_type" is a 2-octet integer, which matches the type in the challenge.

- "token_key_id" is the least significant byte of the `key_id`.

- "blinded_request" is the Ne-octet blinded message defined above, computed as
  `SerializeElement(blinded_msg)`. Ne is as defined in {{OPRF, Section 4}}.

The value `tweaked_key` is stored locally and used later as described in {{finalization}}.
The Client then generates an HTTP POST request to send to the Issuer,
with the TokenRequest as the body. The media type for this request
is "message/token-request". An example request is shown below.

~~~
:method = POST
:scheme = https
:authority = issuer.example.net
:path = /example-token-request
accept = message/token-response
cache-control = no-cache, no-store
content-type = message/token-request
content-length = <Length of TokenRequest>

<Bytes containing the TokenRequest>
~~~

Upon receipt of the request, the Issuer validates the following conditions:

- The TokenRequest contains a supported token_type.
- The TokenRequest.token_key_id corresponds to a key ID of a Public Key owned by the issuer.
- The TokenRequest.blinded_msg is of the correct size.

If any of these conditions is not met, the Issuer MUST return an HTTP 400 error
to the Client, which will forward the error to the client.

## Issuer-to-Client Response {#issuer-to-client}

If the Issuer is willing to produce a token token to the Client, the Issuer
completes the issuance flow by computing a blinded response as follows:

~~~
server_context = SetupPOPRFServer(0x0004, skI, pkI)
evaluate_msg, proof = server_context.Evaluate(skI,
    TokenRequest.blinded_message, info)
~~~

SetupPOPRFServer is in {{OPRF, Section 3.2}} and Evaluate is
defined in {{OPRF, Section 3.3.3}}. The Issuer then creates
a TokenResponse structured as follows:

~~~
struct {
   uint8_t evaluate_response[Nk];
   uint8_t evaluate_proof[Ns+Ns];
} TokenResponse;
~~~

The structure fields are defined as follows:

- "evaluate_response" is the Ne-octet evaluated messaged, computed as
  `SerializeElement(evaluated_msg)`.

- "evaluate_proof" is the (Ns+Ns)-octet serialized proof, which is a pair of Scalar values,
  computed as `concat(SerializeScalar(proof[0]), SerializeScalar(proof[1]))`,
  where Ns is as defined in {{OPRF, Section 4}}.

The Issuer generates an HTTP response with status code 200 whose body consists
of TokenResponse, with the content type set as "message/token-response".

~~~
:status = 200
content-type = message/token-response
content-length = <Length of TokenResponse>

<Bytes containing the TokenResponse>
~~~

## Finalization

Upon receipt, the Client handles the response and, if successful, deserializes
the body values TokenResponse.evaluate_response and TokenResponse.evaluate_proof.
If deserialization of any value fails, the Client aborts the protocol.
Otherwise, when deserialization yields `evaluated_msg` and `proof`, the Client
processes the response as follows:

~~~
authenticator = client_context.Finalize(context, blind, pkI,
  evaluated_msg, blinded_msg, info, tweaked_key)
~~~

The Finalize function is defined in {{OPRF, Section 3.3.3}}. If this
succeeds, the Client then constructs a Token as follows:

~~~
struct {
    uint16_t token_type = 0x0001
    uint8_t nonce[32];
    uint8_t context[32];
    uint8_t key_id[32];
    uint8_t authenticator[Nk];
} Token;
~~~

Otherwise, the Client aborts the protocol.

## Issuer Configuration

Issuers are configured with Private and Public Key pairs, each denoted skI and
pkI, respectively, used to produce tokens. Each key pair MUST be generated as
follows:

~~~
seed = random(Ns)
(skI, pkI) = DeriveKeyPair(seed, "PrivacyPass")
~~~

The key identifier for this specific key pair, denoted `key_id`, is computed
as follows:

~~~
key_id = SHA256(0x0001 || SerializeElement(pkI))
~~~

# Issuance Protocol for Publicly Verifiable Tokens {#public-flow}

This section describes a variant of the issuance protocol in {{private-flow}}
for producing publicly verifiable tokens. It differs from the previous variant
in two important ways:

1. The output tokens are publicly verifiable by anyone with the Issuer public
   key; and
1. The issuance protocol does not admit public or private metadata to bind
   additional context to tokens.

Otherwise, this variant is nearly identical. In particular, Issuers provide a
Private and Public Key, denoted skI and pkI, respectively, used to produce tokens
as input to the protocol. See {{public-issuer-configuration}} for how this key
pair is generated.

Clients provide the following as input to the issuance protocol:

- Issuer name, identifying the Issuer. This is typically a host name that
  can be used to construct HTTP requests to the Issuer.
- Issuer Public Key pkI, with a key identifier `key_id` computed as
  described in {{public-issuer-configuration}}.
- Challenge value `challenge`, an opaque byte string. For example, this might
  be provided by the redemption protocol in {{HTTP-Authentication}}.

Given this configuration and these inputs, the two messages exchanged in
this protocol are described below.

## Client-to-Issuer Request {#public-request}

The Client first creates an issuance request message for a random value
`nonce` using the input challenge and Issuer key identifier as follows:

~~~
nonce = random(32)
context = SHA256(challenge)
token_input = concat(0x0002, nonce, context, key_id)
blinded_msg, blind_inv = rsabssa_blind(pkI, token_input)
~~~

The rsabssa_blind function is defined in {{BLINDRSA, Section 5.1.1.}}.
The Client then creates a TokenRequest structured as follows:

~~~
struct {
   uint16_t token_type = 0x0002
   uint8_t token_key_id;
   uint8_t blinded_msg[Nk];
} TokenRequest;
~~~

The structure fields are defined as follows:

- "token_type" is a 2-octet integer, which matches the type in the challenge.

- "token_key_id" is the least significant byte of the `key_id`.

- "blinded_msg" is the Nk-octet request defined above.

The Client then generates an HTTP POST request to send to the Issuer,
with the TokenRequest as the body. The media type for this request
is "message/token-request". An example request is shown below, where
Nk = 512.

~~~
:method = POST
:scheme = https
:authority = issuer.example.net
:path = /example-token-request
accept = message/token-response
cache-control = no-cache, no-store
content-type = message/token-request
content-length = <Length of TokenRequest>

<Bytes containing the TokenRequest>
~~~

Upon receipt of the request, the Issuer validates the following conditions:

- The TokenRequest contains a supported token_type.
- The TokenRequest.token_key_id corresponds to a key ID of a Public Key owned by the issuer.
- The TokenRequest.blinded_msg is of the correct size.

If any of these conditions is not met, the Issuer MUST return an HTTP 400 error
to the Client, which will forward the error to the client.

## Issuer-to-Client Response {#public-response}

If the Issuer is willing to produce a token token to the Client, the Issuer
completes the issuance flow by computing a blinded response as follows:

~~~
blind_sig = rsabssa_blind_sign(skI, TokenRequest.blinded_rmsg)
~~~

The rsabssa_blind_sign function is defined in {{BLINDRSA, Section 5.1.2.}}.
The Issuer generates an HTTP response with status code 200 whose body consists
of `blind_sig`, with the content type set as "message/token-response".

~~~
:status = 200
content-type = message/token-response
content-length = <Length of TokenResponse>

<Bytes containing the TokenResponse>
~~~

## Finalization

Upon receipt, the Client handles the response and, if successful, processes the
body as follows:

~~~
authenticator = rsabssa_finalize(pkI, nonce, blind_sig, blind_inv)
~~~

The rsabssa_finalize function is defined in {{BLINDRSA, Section 5.1.3.}}.
If this succeeds, the Client then constructs a Token as described in
{{HTTP-Authentication}} as follows:

~~~
struct {
    uint16_t token_type = 0x0002
    uint8_t nonce[32];
    uint8_t context[32];
    uint8_t key_id[32];
    uint8_t authenticator[Nk];
} Token;
~~~

Otherwise, the Client aborts the protocol.

## Issuer Configuration {#public-issuer-configuration}

Issuers are configured with Private and Public Key pairs, each denoted skI and
pkI, respectively, used to produce tokens. Each key pair MUST be generated as
as a valid 4096-bit RSA private key according to [TODO]. The key identifier
for a keypair (skI, pkI), denoted `key_id`, is computed as SHA256(encoded_key),
where encoded_key is a DER-encoded SubjectPublicKeyInfo object carrying pkI.

# Security considerations

This document outlines how to instantiate the Issuance protocol
based on the VOPRF defined in {{OPRF}} and blind RSA protocol defnied in
{{BLINDRSA}}. All security considerations described in the VOPRF document also
apply in the Privacy Pass use-case. Considerations related to broader privacy
and security concerns in a multi-Client and multi-Issuer setting are deferred
to the Architecture document {{I-D.ietf-privacypass-architecture}}.

# IANA considerations

## Token Type

This document updates the "Token Type" Registry with the following values.

| Value  | Name                   | Publicly Verifiable | Public Metadata | Private Metadata | Nk  | Reference        |
|:-------|:-----------------------|:--------------------|:----------------|:-----------------|:----|:-----------------|
| 0x0001 | OPRF(P-384, SHA-384)   | N                   | Y               | N                | 48  | {{private-flow}} |
| 0x0002 | Blind RSA, 4096        | Y                   | N               | N                | 512 | {{public-flow}}  |
{: #aeadid-values title="Token Types"}

## Media Types

This specification defines the following protocol messages, along with their
corresponding media types:

- TokenRequest: "message/token-request"
- TokenResponse: "message/token-response"

The definition for each media type is in the following subsections.

### "message/token-request" media type

Type name:

: message

Subtype name:

: token-request

Required parameters:

: N/A

Optional parameters:

: None

Encoding considerations:

: only "8bit" or "binary" is permitted

Security considerations:

: see {{security-considerations}}

Interoperability considerations:

: N/A

Published specification:

: this specification

Applications that use this media type:

: N/A

Fragment identifier considerations:

: N/A

Additional information:

: <dl>
  <dt>Magic number(s):</dt><dd>N/A</dd>
  <dt>Deprecated alias names for this type:</dt><dd>N/A</dd>
  <dt>File extension(s):</dt><dd>N/A</dd>
  <dt>Macintosh file type code(s):</dt><dd>N/A</dd>
  </dl>

Person and email address to contact for further information:

: see Authors' Addresses section

Intended usage:

: COMMON

Restrictions on usage:

: N/A

Author:

: see Authors' Addresses section

Change controller:

: IESG

### "message/token-response" media type

Type name:

: message

Subtype name:

: access-token-response

Required parameters:

: N/A

Optional parameters:

: None

Encoding considerations:

: only "8bit" or "binary" is permitted

Security considerations:

: see {{security-considerations}}

Interoperability considerations:

: N/A

Published specification:

: this specification

Applications that use this media type:

: N/A

Fragment identifier considerations:

: N/A

Additional information:

: <dl>
  <dt>Magic number(s):</dt><dd>N/A</dd>
  <dt>Deprecated alias names for this type:</dt><dd>N/A</dd>
  <dt>File extension(s):</dt><dd>N/A</dd>
  <dt>Macintosh file type code(s):</dt><dd>N/A</dd>
  </dl>

Person and email address to contact for further information:

: see Authors' Addresses section

Intended usage:

: COMMON

Restrictions on usage:

: N/A

Author:

: see Authors' Addresses section

Change controller:

: IESG

--- back

# Acknowledgements

The authors of this document would like to acknowledge the helpful
feedback and discussions from Benjamin Schwartz, Joseph Salowey, Sofía
Celi, and Tara Whalen.

