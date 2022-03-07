%%%
title = "JSON Proof Algorithms"
abbrev = "jpa"
docName = "draft-jmiller-json-proof-algorithms-latest"
category = "info"
ipr = "none"
workgroup="todo"
keyword = ["jose", "zkp", "jwp", "jws", "jpa"]

[seriesInfo]
name = "Internet-Draft"
value = "draft-jmiller-json-proof-algorithms"
status = "standard"

[pi]
toc = "yes"

[[author]]
initials = "J."
surname = "Miller"
fullname = "Jeremie Miller"
organization = "Ping Identity"
  [author.address]
   email = "jmiller@pingidentity.com"

[[author]]
initials = "M."
surname = "Jones"
fullname = "Michael B. Jones"
organization = "Microsoft"
  [author.address]
  email = "mbj@microsoft.com"
  uri = "https://self-issued.info/"

%%%

.# Abstract

The JSON Proof Algorithms (JPA) specification registers cryptographic algorithms and identifiers to be used with the JSON Web Proof (JWP) and JSON Web Key (JWK) specifications. It defines several IANA registries for these identifiers.

{mainmatter}

# Introduction

The JSON Web Proof (JWP) draft establishes a new secure container format that supports selective disclosure and unlinkability using Zero-Knowledge Proofs (ZKPs) or other cryptographic algorithms.

# Conventions and Definitions

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT", "SHOULD",
"SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and "OPTIONAL" in this
document are to be interpreted as described in BCP 14 [@!RFC2119] [@RFC8174]
when, and only when, they appear in all capitals, as shown here.

The roles of "issuer", "holder", and "verifier", are used as defined by the [Verifiable Credentials Data Model v1.1](https://www.w3.org/TR/2021/REC-vc-data-model-20211109/).  The term "presentation" is also used as defined by this source, but the term "credential" is avoided in this specification in order to minimize confusion with other definitions.

# Terminology

The terms "JSON Web Signature (JWS)", "Base64url Encoding", "Header Parameter", "JOSE Header", "JWS Payload", "JWS Signature", and "JWS Protected Header" are defined by the JWS specification [JWS].

The terms "JSON Web Proof (JWP)", "JWP Payload", "JWP Proof", and "JWP Protected Header" are defined by the JWP specification [JWP].

These terms are defined by this specification:

Stable Key
  An asymmetric key-pair used by a issuer that is also shared via an out-of-band mechanism to a Verifier in order to validate the signature.

Ephemeral Key
  An asymmetric key-pair that is generated for one-time use by a issuer and never stored or used again outside of the creation of a single JWP.

# Background

JWP defines a container binding together a protected header, one or more payloads, and a cryptographic proof.  It does not define any details about the interactions between an application and the cryptographic libraries that implement proof-supporting algorithms.

Due to the nature of ZKPs, this specification also documents the subtle but important differences in proof algorithms versus those defined by the JSON Web Algorithms RFC.  These differences help support more advanced capabilities such as blinded signatures and predicate proofs.

# Algorithm Basics

The four principal interactions that every proof algorithm MUST support are `[sign](#sign)`, `[verify_signature](#verify-signature)`, `[prove](#prove)`, and `[verify_proof](#verify-proof)`.

Some JPAs MAY also support two additional interactions of `[request_signature](#request-signature)` and `[request_proof](#request-proof)`.  While these do not use a JWP container as input or output, they are included here in order to maximize interoperability across JPA implementations.

## Sign

The JWP is first created as the output of a JPA's `sign` operation.

TODO:

* MUST support the protected header as an octet string
* MUST support one or more payloads, each as an octet string
* MAY support the output of the `request_signature` operation from the requesting party (for blinded payloads)
* MUST include integrity protection for the header and all payloads
* MUST specify all digest and hash2curve methods used

## Verify Signature

Performed by the requesting party to verify the signed JWP.

TODO:

* MAY support local/cached private state from the `request_signature` operation (the blinded payloads)
* MAY return a modified JWP for serialized storage without the local state (with the payloads unblinded)
* MUST fully verify the proof value against the protected header and all payloads
* MUST fail if given a proven JWP

## Prove

Used to apply any selective disclosure choices and perform any unlinkability transformations.

TODO:

* MAY support the output of the `request_proof` operation from the requesting party (for predicate proofs and verifiable computation requests)
* MUST support ability to hide any payload
* MUST always include the protected header
* MAY replace the proof value
* MUST indicate if the input JWP is able to be used again
* MAY support an input JWP that resulted from a previous `prove` operation

## Verify Proof

Performed by the requesting party on a JWP to verify any revealed payloads and/or assertions about them from the proving party, while also verifying they are the same payloads and ordering as witnessed by the signing party.

TODO:

* MUST verify the integrity of all revealed payloads
* MUST verify any included assertions about a hidden payload as true
* MAY support local state from the `request_proof` operation
* Out of scope is app interface to interact with the resulting verified assertions (may also be part of the request proof state)
* MAY indicate if the JWP can be re-used to generate a new proof
* MUST fail if given only a signed JWP

## Request Signature

TODO

## Request Proof

TODO

# Algorithm Specifications

This section defines how to use specific algorithms for JWPs.

## Single Use

The Single Use (SU) algorithm is based on composing multiple traditional JWS values into a single JWP proof value.  It enables a very simple form of selective disclosure without requiring any advanced cryptographic techniques.  It does not support unlinkability if the same JWP is presented multiple times, therefore when privacy is required the holder must be able to interact with the issuer again to receive new single-use JWPs (dynamically or in batches).

### Holder Setup

In order to support the protection of a presentation by the holder to the verifier, the holder must use a Proof-of-Possession (PoP) key both during the issuance and the presentation.  The SU algorithm uses [RFC 7800](https://datatracker.ietf.org/doc/html/rfc7800) to accomplish this, adding a "cnf" claim to the issuer's protected header that identifies the PoP key being used by the holder.

The PoP key's algorithm MUST be the same one as used to create the JWS values for the SU proof.  The issuer MUST verify that the holder has possession of this key before adding the "cnf" claim with the matching key information.  The holder-issuer communication to exchange this information is out of scope of this specification.

### Issuer Setup

To create a Single Use JWP the issuer must first generate a unique ephemeral key-pair using a JWS algorithm.  This key-pair will be used to sign the parts of a single JWP and then discarded.  The selected algorithm must be the same for both the ephemeral key and the issuer's stable key.

The issuer MUST choose an asymmetric JWS algorithm so that each signature is non-deterministic.  This ensures that no other party can brute-force any non-disclosed payloads based only on their individual signatures.

### Using JWS

JSON Web Signatures are used to create the signature values used by the SU algorithm.  This allows an implementation to use an existing JWS library directly for all necessary cryptographic operations without requiring any additional primitives.

Each individual JWS uses a fixed  protected header containing only the minimum required `alg` value.  Since this JWS protected header itself is the same for every JWS, it SHOULD be a static value in the form of `{"alg":"***"}` where `***` is the JWS asymmetric key algorithm being used.  This value is re-created by a Verifier using the correct algorithm value.

If an implementation uses an alternative JWS protected header than the fixed value then a base64url encoded serialized form of that fixed header MUST be included as the `proof_header` value in the JWP protected header.

### Protected Header

The JWK of the ephemeral key MUST be included in the JWP protected header with the property name of `proof_jwk` and contain only the REQUIRED values to represent the public key.

The final JWP protected header is then used directly as the body of a JWS and signed using the issuer's stable key.  The resulting JWS signature value as an unencoded octet string is the first value in the JWP Proof.

### Payloads

Each JWP Payload is processed in order and signed as a JWS body using the ephemeral key.  The resulting JWS signature value is appended to the JWP Proof.

The appended total of the stable header signature and ephemeral payload signatures as an octet string will be the fixed length of each signature (for example, 32 octets for the ES256 algorithm), multiplied by the number of payloads plus the JWP protected header (example total would be `64 * (1 protected header + 5 payloads) = 384 octets`).

### Selective Disclosure

The holder is able to derive a new Proof value when presenting it to a Verifier.  The presented Proof value will always contain the stable signature for the protected header as the first element.  It is then followed by only the ephemeral signatures for each payload that is disclosed with order preserved.  Non-disclosed payloads will NOT have their ephemeral signature value included.  For example, if the second and fifth payloads are hidden then the holder's derived Proof value would be of the length `64 * (1 header signature + the 1st, 2nd, and 4th payload signatures) = 256 octets`.

Since the individual signatures in the Proof value are not changed from the issuer, the JWP SHOULD only be used and presented a single time to each Verifier in order for the holder to remain unlinkable across multiple presentations.

### Verification

With each disclosed payload verified as described above, the Verifier MUST verify the JWP protected header against the first matching JWS signature part in the Proof value using the issuer's stable key.  With this verified, the ephemeral key can then be used from the protected header to verify the payload signatures.

The Verifier uses only the disclosed payloads and generates or uses the included fixed JWS protected header in order to perform validation of just those payloads.  It uses the matching JWS signature part from the Proof value to verify with the already verified ephemeral key.

### JPA Registration

Proposed JWP `alg` value is of the format "SU-" appended with the relevant JWS `alg` value for the chosen public and ephemeral key-pair algorithm, for example "SU-ES256".

## BBS

The BBS Signature Scheme under active standards development as a [work item](https://github.com/decentralized-identity/bbs-signature) within the DIF [Applied Cryptography Working Group](https://identity.foundation/working-groups/crypto.html).  Prior to this effort, a [V1 implementation of BBS](https://github.com/mattrglobal/bbs-signatures) has been released and maintained by a community of individuals with notable adoption in multiple early stage decentralized identity projects.

This JSON Proof Algorithm definition for BBS is based on the already released implementation and relies on the provided software API.  A future definition with a different `alg` value will be created to succeed this version as the BBS standardization effort progresses.

This algorithm supports both selective disclosure and unlinkability, enabling the holder to generate multiple proofs from one signed JWP without the verifier being able to correlate those proofs together.

### BLS Curve

The pairing friendly elliptic curve used for the BBS software implementation is part of the BLS family with an embedding degree of 12 over a 381-bit prime field.  For this JPA, only the group G2 is used.

In the implementation the method used to generate the key pairs is `generateBls12381G2KeyPair()`.

### Messages

BBS is a multi-message scheme and operates on an array of individual messages for signing and proof generation.  Each message is a single binary octet string.  The BBS implementation uses a hash-to-curve method to map each message to a point.

### Protected Header

The UTF-8 octet string of the JWP Protected Header is the first message in the input array at index 0.

### Payloads

The octet strings of each payload are placed into the BBS message array following the protected header message.  For example, first payload is at index 1 of the array and the last payload is always the last message in the array.

In future versions of this algorithm, there will be additional methods defined for transforming a payload into a point such that additional Zero-Knowledge Proof types can be supported by the holder such as range and membership predicates.

### Signing

The issuer's BLS12-381 G2 key pair is used to sign the completed message array input containing the octet strings of the Protected Header and every payload.  The result is a signature octet string that is used as the initial JWP Proof value.

In the implementation, the method used to perform the signing is `blsSign({keyPair, [header, payload1, payload2, ...]})` and returns a binary signature value.

### Proving

The holder must decode the JWP header and payload values in order to generate the identical message array that the issuer used.

To generate a JWP for a verifier, the holder must use a cryptographic nonce that is provided by that verifier as input.  This nonce MUST be a 32 byte octet string that the verifier generated by a secure RNG.

The holder also applies selective disclosure preferences by creating an array of indices of which messages in the input array are to be revealed to the verifier.  The revealed indices MUST include the value `0` so that the protected header message is always revealed to the verifier.

The result of creating a proof is an octet string that is used as the verifiable JWP proof value.

In the implementation, the method used to generate the proof is `blsCreateProof({signedProof, publicKey, [header, payload1, payload2, ...], nonce, [0, 2, ...])`.

### Verification

The verifier decodes the JWP header and payload values into a messages array, skipping any non-revealed payloads.  The current BBS implementation embeds the revealed indices into the output proof value so the verification messages array only needs to include the revealed messages.

In the implementation, the method used to verify the proof is `blsVerifyProof({verifyProof, publicKey, [header, payload2, ...] nonce)`.

### JPA Registration

Proposed JWP `alg` value for BBS based on the software implementation is "BBS-X".

### Example

The following example uses the given BLS12-384 key-pair:

Public:
```json
[174, 25, 240, 10, 149, 197, 107, 241, 79, 27, 32, 83, 19, 34, 100, 118, 209, 75, 211, 26, 186, 42, 67, 136, 206, 88, 202, 153, 44, 176, 141, 59, 16, 171, 59, 234, 35, 138, 148, 233, 234, 230, 121, 210, 157, 73, 248, 245, 13, 71, 6, 15, 244, 112, 20, 176, 167, 127, 12, 33, 115, 49, 243, 163, 157, 209, 244, 232, 99, 209, 128, 150, 67, 253, 208, 102, 97, 60, 220, 255, 204, 32, 159, 193, 241, 57, 198, 51, 118, 187, 18, 156, 46, 114, 66, 49]
```

Private:
```json
[44, 254, 245, 65, 205, 228, 50, 118, 124, 30, 71, 54, 61, 149, 218, 51, 168, 222, 50, 176, 12, 145, 125, 33, 213, 111, 123, 182, 153, 38, 87, 118]
```

The protected header used is:
```json
{
  "iss": "https://issuer.tld",
  "claims": [
    "family_name",
    "given_name",
    "email",
    "age"
  ],
  "typ": "JPT",
  "alg": "BBS-X"
}
```

The first payload is the string `"Doe"` with the octet sequence of `[34, 68, 111, 101, 34]` and base64url-encoded as `IkRvZSI`.  

The second payload is the string `"Jay"` with the octet sequence of `[34, 74, 97, 121, 34]` and base64url-encoded as `IkpheSI`.  

The third payload is the string `"jaydoe@example.org"` with the octet sequence of `[34, 106, 97, 121, 100, 111, 101, 64, 101, 120, 97, 109, 112, 108, 101, 46, 111, 114, 103, 34]` and base64url-encoded as `ImpheWRvZUBleGFtcGxlLm9yZyI`.  

The fourth payload is the string `42` with the octet sequence of `[52, 50]` and base64url-encoded as `NDI`.  

The message array used as an input to the BLS implementation is:

```json
[
  [123, 34, 105, 115, 115, 34, 58, 34, 104, 116, 116, 112, 115, 58, 47, 47, 105, 115, 115, 117, 101, 114, 46, 116, 108, 100, 34, 44, 34, 99, 108, 97, 105, 109, 115, 34, 58, 91, 34, 102, 97, 109, 105, 108, 121, 95, 110, 97, 109, 101, 34, 44, 34, 103, 105, 118, 101, 110, 95, 110, 97, 109, 101, 34, 44, 34, 101, 109, 97, 105, 108, 34, 44, 34, 97, 103, 101, 34, 93, 44, 34, 116, 121, 112, 34, 58, 34, 74, 80, 84, 34, 44, 34, 97, 108, 103, 34, 58, 34, 66, 66, 83, 45, 88, 34, 125],
  [34, 68, 111, 101, 34],
  [34, 74, 97, 121, 34],
  [34, 106, 97, 121, 100, 111, 101, 64, 101, 120, 97, 109, 112, 108, 101, 46, 111, 114, 103, 34],
  [52, 50]
]
```

Using the above inputs, the output of the `blsSign()` call is the octet string:
```json
[182, 93, 71, 253, 212, 100, 97, 101, 156, 37, 197, 153, 3, 41, 66, 160, 124, 185, 77, 145, 221, 108, 176, 44, 165, 253, 254, 67, 99, 173, 227, 178, 78, 75, 236, 41, 2, 224, 179, 27, 54, 221, 11, 83, 36, 218, 2, 24, 97, 65, 229, 41, 252, 220, 112, 78, 38, 239, 6, 153, 202, 130, 196, 144, 18, 197, 136, 173, 160, 231, 132, 54, 139, 224, 157, 181, 128, 96, 13, 217, 20, 14, 202, 20, 45, 32, 76, 112, 125, 76, 192, 97, 240, 118, 55, 215, 166, 87, 69, 52, 90, 199, 23, 31, 200, 2, 242, 213, 158, 44, 158, 49]
```

The resulting signed JWP in JSON serialization is:
```json
{
  "protected": "eyJpc3MiOiJodHRwczovL2lzc3Vlci50bGQiLCJjbGFpbXMiOlsiZmFtaWx5X25hbWUiLCJnaXZlbl9uYW1lIiwiZW1haWwiLCJhZ2UiXSwidHlwIjoiSlBUIiwiYWxnIjoiQkJTLVgifQ",
  "payloads": [
    "IkRvZSI",
    "IkpheSI",
    "ImpheWRvZUBleGFtcGxlLm9yZyI",
    "NDI"
  ],
  "proof": "tl1H_dRkYWWcJcWZAylCoHy5TZHdbLAspf3-Q2Ot47JOS-wpAuCzGzbdC1Mk2gIYYUHlKfzccE4m7waZyoLEkBLFiK2g54Q2i-CdtYBgDdkUDsoULSBMcH1MwGHwdjfXpldFNFrHFx_IAvLVniyeMQ"
}
```

The same JWP in compact serialization:
```
ImV5SnBjM01pT2lKb2RIUndjem92TDJsemMzVmxjaTUwYkdRaUxDSmpiR0ZwYlhNaU9sc2labUZ0YVd4NVgyNWhiV1VpTENKbmFYWmxibDl1WVcxbElpd2laVzFoYVd3aUxDSmhaMlVpWFN3aWRIbHdJam9pU2xCVUlpd2lZV3huSWpvaVFrSlRMVmdpZlEi.IkRvZSI~IkpheSI~ImpheWRvZUBleGFtcGxlLm9yZyI~NDI.tl1H_dRkYWWcJcWZAylCoHy5TZHdbLAspf3-Q2Ot47JOS-wpAuCzGzbdC1Mk2gIYYUHlKfzccE4m7waZyoLEkBLFiK2g54Q2i-CdtYBgDdkUDsoULSBMcH1MwGHwdjfXpldFNFrHFx_IAvLVniyeMQ
```

For verification a nonce is needed:
```json
[127, 207, 97, 120, 199, 184, 23, 159, 105, 123, 246, 10, 55, 7, 221, 96, 213, 104, 73, 98, 217, 32, 121, 237, 162, 90, 150, 9, 180, 72, 226, 188]
```

To generate a proof, the `blsCreateProof()` method is used with a revealed indexes array argument of `[0, 2, 4]` and results in the octet string:
```json
[0, 5, 21, 161, 191, 254, 45, 60, 183, 31, 8, 229, 8, 160, 190, 97, 252, 230, 185, 215, 158, 121, 125, 186, 81, 20, 232, 4, 201, 86, 118, 56, 241, 236, 207, 206, 210, 82, 31, 229, 175, 239, 6, 193, 68, 14, 80, 208, 205, 219, 106, 165, 188, 183, 147, 59, 186, 90, 209, 11, 27, 44, 167, 126, 166, 101, 132, 205, 207, 22, 35, 181, 148, 118, 44, 164, 163, 58, 16, 211, 245, 133, 115, 158, 158, 89, 182, 5, 151, 101, 91, 28, 59, 75, 224, 207, 48, 75, 220, 131, 79, 188, 57, 235, 164, 122, 170, 120, 251, 230, 158, 230, 73, 152, 138, 106, 61, 106, 90, 215, 124, 158, 179, 82, 2, 137, 107, 96, 130, 131, 215, 25, 41, 242, 146, 48, 158, 82, 61, 185, 135, 172, 63, 248, 149, 50, 183, 0, 0, 0, 116, 179, 185, 176, 214, 11, 160, 24, 235, 157, 189, 141, 33, 225, 119, 5, 146, 167, 175, 227, 188, 165, 196, 210, 156, 216, 164, 28, 167, 32, 97, 215, 36, 122, 137, 173, 182, 17, 233, 69, 41, 20, 44, 69, 203, 236, 226, 24, 242, 0, 0, 0, 2, 0, 199, 248, 173, 124, 33, 127, 253, 79, 170, 77, 153, 101, 169, 100, 88, 190, 206, 55, 24, 25, 80, 108, 215, 80, 134, 128, 70, 190, 240, 209, 148, 67, 202, 130, 180, 145, 51, 72, 188, 140, 185, 142, 193, 246, 155, 12, 248, 65, 14, 209, 72, 248, 119, 250, 123, 46, 136, 8, 191, 45, 173, 77, 125, 180, 76, 253, 10, 139, 244, 156, 138, 226, 149, 41, 76, 50, 45, 210, 231, 74, 18, 192, 91, 36, 154, 46, 158, 76, 243, 13, 119, 122, 107, 206, 91, 226, 160, 27, 80, 206, 97, 32, 74, 42, 155, 180, 137, 229, 210, 80, 236, 0, 0, 0, 4, 6, 143, 167, 94, 118, 29, 1, 252, 243, 190, 176, 42, 45, 175, 213, 182, 81, 7, 250, 244, 141, 136, 168, 27, 164, 110, 101, 28, 181, 237, 236, 54, 12, 67, 44, 104, 187, 33, 249, 159, 143, 104, 204, 234, 0, 251, 101, 79, 170, 206, 113, 44, 235, 62, 242, 254, 139, 252, 218, 73, 155, 48, 127, 131, 79, 41, 134, 186, 183, 174, 247, 112, 103, 33, 131, 82, 115, 233, 36, 235, 12, 179, 171, 71, 55, 154, 86, 221, 77, 206, 171, 190, 186, 31, 192, 43, 88, 0, 45, 109, 204, 217, 154, 132, 113, 93, 166, 177, 51, 31, 118, 153, 88, 116, 45, 113, 111, 187, 7, 150, 214, 184, 244, 198, 149, 222, 93, 101]
```

The resulting verifiable JWP in JSON serialization is:
```json
{
  "protected": "eyJpc3MiOiJodHRwczovL2lzc3Vlci50bGQiLCJjbGFpbXMiOlsiZmFtaWx5X25hbWUiLCJnaXZlbl9uYW1lIiwiZW1haWwiLCJhZ2UiXSwidHlwIjoiSlBUIiwiYWxnIjoiQkJTLVgifQ",
  "payloads": [
    null,
    "IkpheSI",
    null,
    "NDI"
  ],
  "proof": "AAUVob_-LTy3HwjlCKC-YfzmudeeeX26URToBMlWdjjx7M_O0lIf5a_vBsFEDlDQzdtqpby3kzu6WtELGyynfqZlhM3PFiO1lHYspKM6ENP1hXOenlm2BZdlWxw7S-DPMEvcg0-8Oeukeqp4--ae5kmYimo9alrXfJ6zUgKJa2CCg9cZKfKSMJ5SPbmHrD_4lTK3AAAAdLO5sNYLoBjrnb2NIeF3BZKnr-O8pcTSnNikHKcgYdckeomtthHpRSkULEXL7OIY8gAAAAIAx_itfCF__U-qTZllqWRYvs43GBlQbNdQhoBGvvDRlEPKgrSRM0i8jLmOwfabDPhBDtFI-Hf6ey6ICL8trU19tEz9Cov0nIrilSlMMi3S50oSwFskmi6eTPMNd3przlvioBtQzmEgSiqbtInl0lDsAAAABAaPp152HQH8876wKi2v1bZRB_r0jYioG6RuZRy17ew2DEMsaLsh-Z-PaMzqAPtlT6rOcSzrPvL-i_zaSZswf4NPKYa6t673cGchg1Jz6STrDLOrRzeaVt1Nzqu-uh_AK1gALW3M2ZqEcV2msTMfdplYdC1xb7sHlta49MaV3l1l"
}
```

The same JWP in compact serialization:
```
ImV5SnBjM01pT2lKb2RIUndjem92TDJsemMzVmxjaTUwYkdRaUxDSmpiR0ZwYlhNaU9sc2labUZ0YVd4NVgyNWhiV1VpTENKbmFYWmxibDl1WVcxbElpd2laVzFoYVd3aUxDSmhaMlVpWFN3aWRIbHdJam9pU2xCVUlpd2lZV3huSWpvaVFrSlRMVmdpZlEi.~IkpheSI~~NDI.AAUVob_-LTy3HwjlCKC-YfzmudeeeX26URToBMlWdjjx7M_O0lIf5a_vBsFEDlDQzdtqpby3kzu6WtELGyynfqZlhM3PFiO1lHYspKM6ENP1hXOenlm2BZdlWxw7S-DPMEvcg0-8Oeukeqp4--ae5kmYimo9alrXfJ6zUgKJa2CCg9cZKfKSMJ5SPbmHrD_4lTK3AAAAdLO5sNYLoBjrnb2NIeF3BZKnr-O8pcTSnNikHKcgYdckeomtthHpRSkULEXL7OIY8gAAAAIAx_itfCF__U-qTZllqWRYvs43GBlQbNdQhoBGvvDRlEPKgrSRM0i8jLmOwfabDPhBDtFI-Hf6ey6ICL8trU19tEz9Cov0nIrilSlMMi3S50oSwFskmi6eTPMNd3przlvioBtQzmEgSiqbtInl0lDsAAAABAaPp152HQH8876wKi2v1bZRB_r0jYioG6RuZRy17ew2DEMsaLsh-Z-PaMzqAPtlT6rOcSzrPvL-i_zaSZswf4NPKYa6t673cGchg1Jz6STrDLOrRzeaVt1Nzqu-uh_AK1gALW3M2ZqEcV2msTMfdplYdC1xb7sHlta49MaV3l1l
```

## ZKSnark

TBD

# Security Considerations

* Data minimization of the proof value
* Unlinkability of the protected header contents

# IANA Considerations

## JWP Algorithms Registry

This section establishes the IANA JWP Algorithms Registry.  It also registers the following algorithms.

TBD

{backmatter}

# Acknowledgements

TBD
