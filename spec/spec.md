# Wallet And Credential Interactions

> WACI [__wak__-ee]

**Specification Status:** Draft

**Editors:**
~ [Afshan Aman](https://www.linkedin.com/in/afshan-aman/) (Bloom)
~ [Eddie Hedges](https://www.linkedin.com/in/eddiehedges/) (Bloom)

**Contributors:**
~ [Jace Hensley](https://www.linkedin.com/in/jacehensley/) (Bloom)

**Participate:**
~ [GitHub repo](https://github.com/decentralized-identity/wallet-and-credential-interactions)
~ [File a bug](https://github.com/decentralized-identity/wallet-and-credential-interactions/issues)
~ [Commit history](https://github.com/decentralized-identity/wallet-and-credential-interactions/commits/master)

---

## Abstract

There are interactions between a wallet and relying party that require passing information between the two. WACI provides a standard for these interactions.

## Status of This Document

WACI is a draft specification under development by Bloom and Affinidi.

## Interactions

All interactions use the same common blocks:

### QR Code or Link

Two ways of initiating an interaction is for the relying party to display either a QR code or a link to the user. There could be other ways to initiate an interaction but this document will be discussing QR codes and links.

If the user is using an app/webiste on something other than the device that their wallet is on, then they would be able to scan a QR code with the wallet. _But_ if the user is using the device that also has their wallet then they wouldn't be able to scan a QR code, they would need to be able to click a link that will open their mobile wallet.

There are of course other use cases where you might need one over the other or both. For example, in an email you may want to display both a link and a QR code because you won't be able to dynamically choose between the two.

### Payload

Some mediums do not allow for a large amount of data to be sent (e.g a QR code) to support those cases the initial payload contains instructions for fetching the challenge token.

This is payload can be displayed in a QR code or added to a link as a query parameter.

<tab-panels selected-index="0">

<nav>
  <button type="button">QR</button>
  <button type="button">Link</button>
</nav>

<section>

![QR Code](./resources/qr.png)

</section>

<section>

<a target="_blank" rel="noopener noreferrer" href="https://deep-link-into-app.com/waci?payload=eyJjaGFsbGVuZ2VUb2tlblVybCI6Imh0dHBzOi8vZXhhbXBsZS5jb20vYXBpL2dldC10b2tlbi9mYjI4YjRlYy00MjA5LTRiNzgtODUwNy03Y2M0OTE2NGNiMzciLCJ2ZXJzaW9uIjoiMSJ9">Click Me!</a>

</section>

</tab-panels>

::: example QR/Link Payload

```json
{
  "challengeTokenUrl": "https://example.com/api/get-token/fb28b4ec-4209-4b78-8507-7cc49164cb37",
  "version": "1"
}
```

:::

- `challengeTokenUrl`:
  - MUST be unique to the interaction
  - MUST be a `GET` endpoint that returns the [Token Payload](#token-url-response)
- `version`:
  - This is the version of just the QR/link payload, not the rest of the interaction

### Token URL Response

The result from `GET`ing the provided `challengeTokenUrl`. This contains the initial JWT that really starts the interaction.

::: example Token URL Response

```json
{
  "challengeToken": "{{JWT String}}"
}
```

:::

#### Challenge Token

::: example Challenge Token Header

```json
// Header
{
  "alg": "...",
  "kid": "did:example:ebfeb1f712ebc6f1c276e12ec21#primary"
}
```

:::

- MUST have `alg` and `kid`, so the JWT can be verified

::: example Challenge Token Payload

```json
// Payload
{
  "jti": "...",
  "iss": "did:example:ebfeb1f712ebc6f1c276e12ec21",
  "aud": "...",
  "callbackUrl": "https://example.com/api/callback-url",
  "purpose": "...",
  "version": "..."
}
```

:::

- MUST have `iss`
- MUST have `jti` to protect against replay attacks
- CAN have `aud` if the DID of the wallet is known
- MUST have `callbackUrl`
  - MUST be a `POST` endpoint to take the wallet's reponse (payload determined by the `purpose`)
- MUST have `purpose`
- MUST have `version`
  - This is specific to the `purpose`

### Callback URL

#### Request

Each interaction will `POST` data to the `callbackUrl`:

::: example Callback URL Request Payload

```json
{
  "responseToken": "{{Signed JWT}}",
  "from": "qr" | "link"
}
```

:::

- `responseToken`
  - A JWT signed by the user, will contain the `challengeToken`
- `from`
  - MUST be either "qr" or "link"
  - The issuer may need to handle things differently based on how the user is claiming the credentials

##### Response Token

The response token is signed by the user and acts as a way to prove ownership of their DID and to pass aditional data back to the relying party.

:::example Response Token Header

```json
{
  "alg": "...",
  "kid": "did:example:c276e12ec21ebfeb1f712ebc6f1#primary"
}
```

:::

- MUST have `alg` and `kid`, so the JWT can be verified

:::example Response Token Payload

```json
{
  "iss": "did:example:c276e12ec21ebfeb1f712ebc6f1",
  "aud": "did:example:ebfeb1f712ebc6f1c276e12ec21",
  "challenge": "{{CHALLENGE TOKEN}}"
}
```

:::

- MUST have `iss`
- MUST have `aud`
  - `aud` MUST be the `iss` of the challenge token
- MUST have `challenge`
  - `challenge` MUST be the challenge token given by the issuer

#### Reponse

The `POST` to the provided `callbackUrl` can return with a simple successful HTTP response or it can return a success with follow up details. A `redirectLink` that the app will open in a browser or `challengeToken` that will start a new interaction.

::: example Response Redirect Payload

```json
{
  "redirectUrl": "https://example.com/redirect-url?id={{Some id that identifies the user}}"
}
```

:::

This could be used to show a success message or bring them back to the website/app to continue where they left off. Most of the time `redirectUrl` will only be used when the user is already using their phone (see [above](#qr-code-or-link)).

**OR**

::: example Response Chain Payload

```json
{
  "challengeToken": "{{JWT String}}"
}
```

:::

This could be used to follow up a request interaction with an offer interaction, or even a chain of request interactions that are based on the previously shared VCs.

### Token Storage

Because the challenge token is always sent back to the relying party, the token doesn't need to be stored on creation. And this allows the relying party to not have to worry about someone spamming their API and driving up their storage costs.

But no storage at all can lead to replay attacks. One suggested way to mitigate replay attacks while keeping storage to a minimum is to only store the hash of "used" tokens and have a cron job that cleans this storage based on expiration date of the tokens.

### Swimlane

Each interaction will be slightly different but will follow this general pattern:

<tab-panels selected-index="0">

<nav>
  <button type="button">QR Based</button>
  <button type="button">Link Based</button>
</nav>

<section>

```mermaid
sequenceDiagram
  title: Interaction (QR)

  activate Wallet

  Wallet ->>+ Relying Party's Interface: Scan QR Code
  Relying Party's Interface -->>- Wallet: Retrieve `challengeTokenUrl`

  Wallet ->>+ Relying Party: GET `challengeTokenUrl`
  Relying Party -->> Wallet: Return `challengeToken`

  Wallet ->> Wallet: Verify/decode `challengeToken`
  Wallet ->> Wallet: Create/sign a `responseToken`, with `challengeToken` as the `challenge`

  Wallet ->> Relying Party: POST the `responseToken` to `challengeToken`'s `callBackUrl`
  Relying Party ->> Relying Party: Verify the `responseToken`
  Relying Party ->> Relying Party: Verify the `responseToken`'s challenge token (valid JWT, signed by Relying Party, and not used before)
  Relying Party -->>- Wallet: Return success

  opt `redirectUrl` or `challengeToken` is provided
    alt `redirectUrl` is provided
      Wallet ->> Browser: Open `redirectUrl`
    else `challengeToken` is provided
      Wallet ->> Wallet: Start new interaction
    end
  end

  deactivate Wallet
```

</section>

<section>

```mermaid
sequenceDiagram
  title: Interaction (Link)

  User ->>+ Relying Party's Interface: Click link

  Relying Party's Interface ->>- Wallet: Open Wallet with deep link

  activate Wallet

  Wallet ->> Wallet: Parse deep link

  Wallet ->>+ Relying Party: GET `challengeTokenUrl`
  Relying Party -->> Wallet: Return `challengeToken`

  Wallet ->> Wallet: Verify/decode `challengeToken`
  Wallet ->> Wallet: Create/sign a `responseToken`, with `challengeToken` as the `challenge`

  Wallet ->> Relying Party: POST the `responseToken` to `challengeToken`'s `callBackUrl`
  Relying Party ->> Relying Party: Verify the `responseToken`
  Relying Party ->> Relying Party: Verify the `responseToken`'s challenge token (valid JWT, signed by Relying Party, and not used before)
  Relying Party -->>- Wallet: Return success

  opt `redirectUrl` or `challengeToken` is provided
    alt `redirectUrl` is provided
      Wallet ->> Browser: Open `redirectUrl`
    else `challengeToken` is provided
      Wallet ->> Wallet: Start new interaction
    end
  end

  deactivate Wallet
```

</section>

</tab-panels>

## Offer/Claim

The offer/claim interaction is for the use case where an issuer wants to give credential(s) to the user.

### Challenge Token

An example of an `offer` challenge token has the following properties (in addition to the base [properties](#challenge-token)):

:::example Offer Challenge Token Header

```json
// Header
{
  "alg": "...",
  "kid": "did:example:ebfeb1f712ebc6f1c276e12ec21#primary"
}
```

:::

:::example Offer Challenge Token Payload

```json
// Payload
{
  "jti": "...",
  "iss": "did:example:ebfeb1f712ebc6f1c276e12ec21",
  "aud": "...",
  "callbackUrl": "https://example.com/api/callback-url",
  "purpose": "offer",
  "version": "0.1",
  "credential_manifest": {
    "issuer": {
      /* ... */
    },
    "output_descriptors": [
      /* ... */
    ],
    "presentation_definition": {
      /* ... */
    }
  }
}
```

:::

- `purpose` MUST be `"offer"`
- MUST have `credential_manifest`
  - Uses the [Credential Manifest](#credential-manifest-working-copy) to define the available credentials
  - If the `credential_manifest` provides a `presentation_definition` the response MUST include a `verifiable_presentation`

### Callback URL

#### Request

The offer/claim interaction follows the standard [Callback URL Request](#request) payload, but the `responseToken`'s is specific to the this interaction.

:::example Offer Callback URL Request Payload

```json
{
  "responseToken": "{{Signed JWT}}",
  "from": "qr" | "link"
}
```

:::

##### Response Token

In addition to the standard `responseToken` the offer/claim interaction adds `verifiable_presentation` to the payload.

:::example Offer Response Token Header

```json
// Header
{
  "alg": "...",
  "kid": "did:example:c276e12ec21ebfeb1f712ebc6f1#primary"
}
```

:::

:::example Offer Response Token Payload

```json
// Payload
{
  "iss": "did:example:c276e12ec21ebfeb1f712ebc6f1",
  "aud": "did:example:ebfeb1f712ebc6f1c276e12ec21",
  "challenge": "{{CHALLENGE TOKEN}}",
  "verifiable_presentation": {
    /* ... */
    "type": ["VerifiablePresentation", "PresentationSubmission"],
    "presentation_submission": {
      /* ... */
    }
    /* ... */
  }
}
```

:::

- MUST have `verifiable_presentation` IF the challenge token provides a `presentation_definition`
  - Uses Presentation Exchange's [Presentation Submission](https://identity.foundation/presentation-exchange/#presentation-submission)
  - This `VerifiablePresentation` MUST be a `PresentationSubmission`

#### Response

In addition to the standard [Callback URL Response](#response) payload, the offer/claim flow adds `verifiable_presentation`:

:::example Offer Default Response Payload

```json
{
  "verifiable_presentation": [
    {
      /* ... */
      "type": ["VerifiablePresentation", "CredentialFulfillment"],
      "credential_fulfillment": {
        /* ... */
      }
      /* ... */
    }
  ]
}
```

:::

**OR**

:::example Offer Redirect Response Payload

```json
{
  "verifiable_presentation": [
    {
      /* ... */
      "type": ["VerifiablePresentation", "CredentialFulfillment"],
      "credential_fulfillment": {
        /* ... */
      }
      /* ... */
    }
  ]
  "redirectUrl": "https://example.com/redirect-url?id={{Some id that identifies the user}}"
}
```

:::

**OR**

:::example Offer Chain Response Payload

```json
{
  "verifiable_presentation": [
    {
      /* ... */
      "type": ["VerifiablePresentation", "CredentialFulfillment"],
      "credential_fulfillment": {
        /* ... */
      }
      /* ... */
    }
  ]
  "challengeToken": "{{JWT String}}"
}
```

:::

- `verifiable_presentation` MUST be a `CredentialFulfillment` presentation
  - Uses Credential Manifest's [Credential Fulfillment](#credential-fulfillment)

### Swimlane

<tab-panels selected-index="0">

<nav>
  <button type="button">QR Based</button>
  <button type="button">Link Based</button>
</nav>

<section>

```mermaid
sequenceDiagram
  title: Offer/Claim (QR)

  activate Wallet

  Wallet ->>+ Issuer's Interface: Scan QR Code
  Issuer's Interface -->>- Wallet: Retrieve `challengeTokenUrl`

  Wallet ->>+ Issuer: GET `challengeTokenUrl`
  Issuer -->> Wallet: Return `challengeToken`

  Wallet ->> Wallet: Verify/decode `challengeToken`

  alt `challengeToken`'s `credential_manifest` contains a `presentation_definition`
    Wallet ->> Wallet: Collect VCs that are described in the `challengeToken`s `presentation_definition`
    Wallet ->> Wallet: Create VP containing the VCs, with `challengeToken` as the `challenge`
    Wallet ->> Wallet: Create/sign a `responseToken` containing the VP, with `challengeToken` as the `challenge`
  else `challengeToken` is provided
    Wallet ->> Wallet: Create/sign a `responseToken`, with `challengeToken` as the `challenge`
  end

  Wallet ->> Issuer: POST `responseToken` to `challengeToken`'s `callBackUrl`
  Issuer ->> Issuer: Verify `responseToken`
  Issuer ->> Issuer: Verify `responseToken`'s `challenge` (valid JWT, signed by issuer, and not used before)
  Issuer -->>- Wallet: Return list of signed credentials

  Wallet ->> Wallet: Store credentials

  opt `redirectUrl` or `challengeToken` is provided
    alt `redirectUrl` is provided
      Wallet ->> Browser: Open `redirectUrl`
    else `challengeToken` is provided
      Wallet ->> Wallet: Start new interaction
    end
  end

  deactivate Wallet
```

</section>

<section>

```mermaid
sequenceDiagram
  title: Offer/Claim (Link)

  User ->>+ Issuer's Interface: Click link

  Issuer's Interface ->>- Wallet: Open Wallet with deep link

  activate Wallet

  Wallet ->> Wallet: Parse deep link

  Wallet ->>+ Issuer: GET `challengeTokenUrl`
  Issuer -->> Wallet: Return `challengeToken`

  Wallet ->> Wallet: Verify/decode `challengeToken`

  alt `challengeToken`'s `credential_manifest` contains a `presentation_definition`
    Wallet ->> Wallet: Collect VCs that are described in the `challengeToken`s `presentation_definition`
    Wallet ->> Wallet: Create VP containing the VCs, with `challengeToken` as the `challenge`
    Wallet ->> Wallet: Create/sign a `responseToken` containing the VP, with `challengeToken` as the `challenge`
  else `challengeToken` is provided
    Wallet ->> Wallet: Create/sign a `responseToken`, with `challengeToken` as the `challenge`
  end

  Wallet ->> Issuer: POST `responseToken` to `challengeToken`'s `callBackUrl`
  Issuer ->> Issuer: Verify `responseToken`
  Issuer ->> Issuer: Verify `responseToken`'s `challenge` (valid JWT, signed by issuer, and not used before)
  Issuer -->>- Wallet: Return list of signed credentials

  Wallet ->> Wallet: Store credentials

  opt `redirectUrl` or `challengeToken` is provided
    alt `redirectUrl` is provided
      Wallet ->> Browser: Open `redirectUrl`
    else `challengeToken` is provided
      Wallet ->> Wallet: Start new interaction
    end
  end

  deactivate Wallet
```

</section>

</tab-panels>

## Request/Share

The request/share interaction is for the use case where an verifier wants a user to share credential(s) with them.

### Challenge Token

An example of a `request` challenge token has the following properties (in addition to the base [properties](#challenge-token)):

:::example Request Challenge Token Header

```json
// Header
{
  "alg": "...",
  "kid": "did:example:ebfeb1f712ebc6f1c276e12ec21#primary"
}
```

:::

:::example Request Challenge Token Payload

```json
// Payload
{
  "jti": "...",
  "iss": "did:example:ebfeb1f712ebc6f1c276e12ec21",
  "aud": "...",
  "callbackUrl": "https://example.com/api/callback-url",
  "purpose": "request",
  "version": "0.1",
  "presentation_definition": {
    // ...
  }
}
```

:::

- `purpose` MUST be `"request"`
- MUST have `presentation_definition`
  - Uses [Presentation Exchange](https://identity.foundation/presentation-exchange/) to define the requirements

### Callback URL

#### Request

In addition to the standard [Callback URL Request](#request) payload, the offer/claim flow adds `presentation`

:::example Request Callback URL Request Payload

```json
{
  "responseToken": "{{Signed JWT}}",
  "from": "qr" | "link"
}
```

:::

##### Response Token

In addition to the standard `responseToken` the offer/claim interaction adds `verifiable_presentation` to the payload.

:::example Request Response Token Header

```json
// Header
{
  "alg": "...",
  "kid": "did:example:c276e12ec21ebfeb1f712ebc6f1#primary"
}
```

:::

:::example Request Response Token Payload

```json
// Paylaod
{
  "iss": "did:example:c276e12ec21ebfeb1f712ebc6f1",
  "aud": "did:example:ebfeb1f712ebc6f1c276e12ec21",
  "challenge": "{{CHALLENGE TOKEN}}",
  "verifiable_presentation": {
    /* ... */
    "type": ["VerifiablePresentation", "PresentationSubmission"],
    "presentation_submission": {
      /* ... */
    }
    /* ... */
  }
}
```

:::

- MUST have `verifiable_presentation`
  - Using Presentation Exchange's [Presentation Submission](https://identity.foundation/presentation-exchange/#presentation-submission)
  - This `VerifiablePresentation` MUST be a `PresentationSubmission`
  - This `VerifiablePresentation`'s `proof.challenge` MUST be the challenge token given by the issuer

#### Response

The request/share flow does not add anything to the [Callback URL Response](#response).

### Swimlane

<tab-panels selected-index="0">

<nav>
  <button type="button">QR Based</button>
  <button type="button">Link Based</button>
</nav>

<section>

```mermaid
sequenceDiagram
  title: Request/Share (QR)

  activate Wallet

  Wallet ->>+ Verifier's Interface: Scan QR Code
  Verifier's Interface -->>- Wallet: Retrieve `challengeTokenUrl`

  Wallet ->>+ Verifier: GET `challengeTokenUrl`
  Verifier -->> Wallet: Return `challengeToken`

  Wallet ->> Wallet: Verify/decode `challengeToken`
  Wallet ->> Wallet: Collect VCs that are described in the `challengeToken`s `presentation_definition`
  Wallet ->> Wallet: Create VP containing the VCs, with `challengeToken` as the `challenge`
  Wallet ->> Wallet: Create/sign a `responseToken` containing the VP, with `challengeToken` as the `challenge`

  Wallet ->> Verifier: POST the `responseToken` to `challengeToken`'s `callBackUrl`
  Verifier ->> Verifier: Verify the `responseToken`
  Verifier ->> Verifier: Verify the `responseToken`'s challenge token (valid JWT, signed by verifier, and not used before)
  Verifier -->>- Wallet: Return success

  opt `redirectUrl` or `challengeToken` is provided
    alt `redirectUrl` is provided
      Wallet ->> Browser: Open `redirectUrl`
    else `challengeToken` is provided
      Wallet ->> Wallet: Start new interaction
    end
  end

  deactivate Wallet
```

</section>

<section>

```mermaid
sequenceDiagram
  title: Request/Share (Link)

  User ->>+ Verifier's Interface: Click link

  Verifier's Interface ->>- Wallet: Open Wallet with deep link

  activate Wallet

  Wallet ->> Wallet: Parse deep link

  Wallet ->>+ Verifier: GET `challengeTokenUrl`
  Verifier -->> Wallet: Return `challengeToken`

  Wallet ->> Wallet: Verify/decode `challengeToken`
  Wallet ->> Wallet: Collect VCs that are described in the `challengeToken`s `presentation_definition`
  Wallet ->> Wallet: Create VP containing the VCs, with `challengeToken` as the `challenge`
  Wallet ->> Wallet: Create/sign a `responseToken` containing the VP, with `challengeToken` as the `challenge`

  Wallet ->> Verifier: POST the `responseToken` to `challengeToken`'s `callBackUrl`
  Verifier ->> Verifier: Verify the `responseToken`
  Verifier ->> Verifier: Verify the `responseToken`'s challenge token (valid JWT, signed by verifier, and not used before)
  Verifier -->>- Wallet: Return success

  opt `redirectUrl` or `challengeToken` is provided
    alt `redirectUrl` is provided
      Wallet ->> Browser: Open `redirectUrl`
    else `challengeToken` is provided
      Wallet ->> Wallet: Start new interaction
    end
  end

  deactivate Wallet
```

</section>

</tab-panels>

## Credential Manifest (Working Copy)

Because the [Credential Manifest spec](https://identity.foundation/credential-manifest) is just a strawman at the moment we will rely on a "frozen" copy of the spec outline below. The WACI spec will not be considered stable until the Credential Manifest spec is, but we want to be able to implement early prototypes against something so we need a version of the spec to base it off of.

This is the CM spec as of 02/22/2021.

Alterations in the spec are colored in <span style="color:darkgreen;">green</span>

---

Credential Manifest
==================

**Specification Status:** Strawman

**Latest Draft:**
  [identity.foundation/credential-manifest](https://identity.foundation/credential-manifest)

**Editors:**
~ [Daniel Buchner](https://www.linkedin.com/in/dbuchner/) (Microsoft)
~ [Brent Zundel](https://www.linkedin.com/in/bzundel/) (Evernym)
<!-- -->
**Participate:**
~ [GitHub repo](https://github.com/decentralized-identity/credential-manifest)
~ [File a bug](https://github.com/decentralized-identity/credential-manifest/issues)
~ [Commit history](https://github.com/decentralized-identity/credential-manifest/commits/master)

------------------------------------

### Abstract

For User Agents (e.g. wallets) and other service that wish to engage with Issuers to acquire credentials, there must exist a mechanism for assessing what inputs are required from a Subject to process a request for credential(s) issuance. The _Credential Manifest_ is a common data format for describing the inputs a Subject must provide to an Issuer for subsequent evaluation and issuance of the credential(s) indicated in the Credential Manifest.

_Credential Manifests_ do not themselves define the contents of the output credential(s), the process the Issuer uses to evaluate the submitted inputs, or the protocol Issuers, Subjects, and their User Agents rely on to negotiate credential issuance.

### Status of This Document

Credential Manifest is a draft specification being developed within the [Decentralized Identity Foundation](https://identity.foundation) (DIF), and intended for ratification as a DIF recommended data format. This spec will be updated to reflect relevant changes, and participants are encouraged to contribute at the following repository location: https://github.com/decentralized-identity/credential-manifest


### Terminology

[[def:Decentralized Identifiers, Decentralized Identifier, DID]]
~ Unique ID URI string and PKI metadata document format for describing the
cryptographic keys and other fundamental PKI values linked to a unique,
user-controlled, self-sovereign identifier in a target system (i.e. blockchain,
distributed ledger).

[[def:Claim, Claims]]
~ An assertion made about a [[ref:Subject]]. Used as an umbrella term for
Credential, Assertion, Attestation, etc.

[[def:Issuer, Issuers]]
~ Issuers are entities that issue credentials to a [[ref:Holder]].

[[def:Holder, Holders]]
~ Holders are entities that recieve credentials from [[ref:Issuers]], possibly first submitting proofs the the Issuer to satisfy the requirements described in a Presentation Definition.

[[def:Output Descriptor, Output Descriptors]]
~ Output Descriptors are used by an Issuer to describe the credentials they are offering to a [[ref:Holder]]. See [Output Descriptor](#output-descriptor)

[[def:Output Descriptor Object, Output Descriptor Objects]]
~ Output Descriptor Objects are populated with properties describing the [[ref:Claims]] the [[ref:Issuer]] is offering the [[ref:Holder]]

[[def:Credential Fulfillment, Credential Fulfillments]]
~ Credential Fulfillments are objects embedded within target claim negotiation
formats that unify the presentation of [[ref:Claims]] to a [[ref:Holder]] in
accordance with the output an [[ref:Issuer]] specified in a
[[ref:Credential Manifest]]. See
[Credential Fulfillment](#credential-fulfillment).

### Resource Definition

_Credential Manifests_ are a resource format that defines preconditional requirements, Issuer style preferences, and other facets User Agents utilize to help articulate and select the inputs necessary for processing and issuance of a specified credential.

::: example Credential Manifest - All features exercised
```json
{
  "id": "WA-DL-CLASS-A",
  "version": "0.1.0",
  "issuer": {
    "id": "did:example:123?linked-domains=3",
    "name": "Washington State Government",
    "styles": {
      "thumbnail": {
        "uri": "https://dol.wa.com/logo.png",
        "alt": "Washington State Seal"
      },
      "hero": {
        "uri": "https://dol.wa.com/people-working.png",
        "alt": "People working on serious things"
      },
      "background": {
        "color": "#ff0000"
      },
      "text": {
        "color": "#d4d400"
      }
    }
  },
  "output_descriptors": [
    {
      "schema": [{
        "uri": "http://washington-state-schemas.org/1.0.0/driver-license.json"
      }],
      "display": {
        "title": {
          "path": ["$.name", "$.vc.name"],
          "text": "Washington State Driver License"
        },
        "subtitle": {
          "path": ["$.class", "$.vc.class"],
          "text": "Class A, Commercial"
        },
        "description": {
          "text": "License to operate a vehicle with a gross combined weight rating (GCWR) of 26,001 or more pounds, as long as the GVWR of the vehicle(s) being towed is over 10,000 pounds."
        },
        "properties": [
          {
            "path": ["$.donor", "$.vc.donor"],
            "label": "Organ Donor"
          }
        ]
      },
      "styles": {
        "thumbnail": {
          "uri": "https://dol.wa.com/logo.png",
          "alt": "Washington State Seal"
        },
        "hero": {
          "uri": "https://dol.wa.com/happy-people-driving.png",
          "alt": "Happy people driving"
        },
        "background": {
          "color": "#ff0000"
        },
        "text": {
          "color": "#d4d400"
        }
      }
    }
  ],
  "presentation_definition": {
    // As defined in the Presentation Exchange specification
  }
}
```
:::

#### General Composition

_Credential Manifests_ are JSON objects composed as follows:

  - The object ****MUST**** contain an `issuer` property, and its value ****MUST**** be an object composed as follows:
      - The object ****must**** contain a `id` property, and its value ****must**** be a valid URI string that identifies who the issuer of the credential(s) will be.
      - The object ****MAY**** contain a `name` property, and its value ****must**** be a string that ****SHOULD**** reflect the human-readable name the Issuer wishes to be recognized by.
      - The object ****MAY**** contain a `styles` property, and its value ****must**** be an object composed as defined in the [`styles` properties](#styles-properties) section.
  - The object ****MUST**** contain an `output_descriptors` property. It's vault ****MUST**** be an array of Output Descriptor Objects, the composition of which are described in the [`Output Descriptor`](#output-descriptor) section below
  - The object ****MAY**** contain a `presentation_definition` object, and its value ****MUST**** be a [Presentation Definition](https://identity.foundation/presentation-exchange/#presentation-definition) object, as defined by the [DIF Presentation Exchange](https://identity.foundation/presentation-exchange) specification.


#### Output Descriptor

[[ref:Output Descriptors]] are objects used to describe the [[ref:Claims]] a [[ref:Issuer]] if offering to a [[ref:Holder]].

[[ref:Output Descriptor Objects]] contain schema URI that links to the schema of the offered output data, and information about how to display the output to the Holder.

:::example
```json
{
  "output_descriptors": [
    {
      "id": "driver_license_output",
      "schema": "https://schema.org/EducationalOccupationalCredential",
      "display": {
        "title": {
          "path": ["$.name", "$.vc.name"],
          "text": "Washington State Driver License"
        },
        "subtitle": {
          "path": ["$.class", "$.vc.class"],
          "text": "Class A, Commercial"
        },
        "description": {
          "text": "License to operate a vehicle with a gross combined weight rating (GCWR) of 26,001 or more pounds, as long as the GVWR of the vehicle(s) being towed is over 10,000 pounds."
        },
        "properties": [
          {
            "path": ["$.donor", "$.vc.donor"],
            "label": "Organ Donor"
          }
        ]
      },
      "styles": {
        "thumbnail": {
          "uri": "https://dol.wa.com/logo.png",
          "alt": "Washington State Seal"
        },
        "hero": {
          "uri": "https://dol.wa.com/happy-people-driving.png",
          "alt": "Happy people driving"
        },
        "background": {
          "color": "#ff0000"
        },
        "text": {
          "color": "#d4d400"
        }
      }
    }
  ]
}
```
:::

##### Output Descriptor Object

[[ref:Output Descriptor Objects]] are composed as follows:

- The [[ref:Output Descriptor Object]] ****MUST**** contain an `id` property.
  The value of the `id` property ****MUST**** be a string that does not conflict
  with the `id` of another [[ref:Output Descriptor Object]] in the same
  [[ref:Credential Manifest]].
- The [[ref:Output Descriptor Object]] ****MUST**** contain a `schema` property, and its value ****MUST**** be an array composed of schema objects for the schema(s) of the credentials to be issued.
- The [[ref:Output Descriptor Object]] ****MAY**** contain a `name` property, and if present its value ****SHOULD**** be a human-friendly name that describes what the credential represents.
- The [[ref:Output Descriptor Object]] ****MAY**** contain a `description` property, and if present its value ****MUST**** be a string that describes what the credential is in greater detail.
- The [[ref:Output Descriptor Object]] ****MAY**** contain a `styles` property, and its value ****must**** be an object composed as defined in the [`styles` properties](#styles-properties) section.
- The [[ref:Output Descriptor Object]] ****MAY**** contain a `display` property, and its value ****must**** be an object composed as defined in the [`display` properties](#display-properties) section.

#### `styles` properties

Within a `Credential Manifest`, there are two areas where styling affordances are provided: under the `issuer` property, where the Issuer expresses information about themselves - including how a User Agent should style UI that represents the Issuer, and under the `credential` property, where the Issuer expresses information about the credntial itself - including how a User Agent should style UI for the credential itself. Under each of these areas an implementer ****MAY**** include a `styles` property, and if present, its value ****must**** be an object composed of the following properties:

- The object ****MAY**** contain a `thumbnail` property, and if present, its value ****MUST**** be an object with the following optional properties:
    - The object ****MUST**** contain a `uri` property, and if present its value ****MUST**** be a valid URI string to an image resource.
    - The object ****MAY**** contain an `alt` property, and if present its value ****MUST**** be a string that describes the alternate text for the logo image.
- The object ****MAY**** contain a `hero` property, and if present, its value ****MUST**** be an object with the following optional properties:
    - The object ****MUST**** contain a `uri` property, and if present its value ****MUST**** be a valid URI string to an image resource.
    - The object ****MAY**** contain an `alt` property, and if present its value ****MUST**** be a string that describes the alternate text for the logo image.
- The object ****MAY**** contain a `background` property, and if present, its value ****MUST**** be an object with the following optional properties:
    - The object ****MAY**** contain a `color` property, and if present its value ****MUST**** be a HEX string color value (e.g. #000000).
- The object ****MAY**** contain a `text` property, and if present, its value ****MUST**** be an object with the following optional properties:
    - The object ****MAY**** contain a `color` property, and if present its value ****MUST**** be a HEX string color value (e.g. #000000).

#### `display` properties

The `credential` property of a `Credential Manifest` is an object that ****MAY**** contain a `display` property defining various content and data pointers for representation of a credential in UI. The properties in the object use _Display Mapping Objects_ to assign text and data about the credential to common UI presentation elements, either by selecting data from the credential itself or providing it directly. The `display` object is constructed as follows

- The object ****MAY**** contain a `title` property, and if present, its value ****MUST**** be a _Display Mapping Object_. User Agents ****SHOULD**** render the data in an area of UI that conveys the general title of the credential being rendered.
- The object ****MAY**** contain a `subtitle` property, and if present, its value ****MUST**** be a _Display Mapping Object_. User Agents ****SHOULD**** render the data in close proximity to the `title` value and ****SHOULD**** display the information in a way that is noticably less pronounced than that of the `title` value.
- The object ****MAY**** contain a `description` property, and if present, its value ****MUST**** be a _Display Mapping Object_. User Agents ****SHOULD**** render the data in an area of UI that is appropreate for verbose, descriptive textual data.
- The object ****MAY**** contain a `properties` property, and if present, its value ****MUST**** be an array of _Display Mapping Objects_. User Agents ****SHOULD**** render the data specified by each _Display Mapping Object_ in an area of UI that is appropreate for the rendering of a flat list of labeled values.

##### _Display Mapping Objects_

::: example Display Mapping Object
```json
{
  "display": {
    "title": {
      "path": ["$.name", "$.vc.name"],
      "text": "Washington State Driver License"
    },
    "properties": [
      {
        "path": ["$.vision_aid", "$.vc.vision_aid"],
        "label": "Vision aid required"
      },
      {
        "path": ["$.donor", "$.vc.donor"],
        "label": "Organ Donor"
      }
    ]
  }
}
```
:::

The _Display Mapping Objects_ are JSON objects constructed as follows:

- The object ****MAY**** contain a `path` property, and if present, its value ****MUST**** be a [JSONPath](https://goessner.net/articles/JsonPath/) string expression <span style="color:green;">and the result ****MUST**** be a string or numeric value that is rendered in the UI</span>
- The object ****MAY**** contain a `text` property, and if present, its value ****MUST**** be a string or numeric value that is rendered in the UI if no `path` property is specified within the object, or all of the `path` property's array of [JSONPath](https://goessner.net/articles/JsonPath/) string expressions fail to select data within the target credential.
- The object ****MAY**** contain a `label` property, and if present, its value ****MUST**** be a string that is rendered in the UI where a labled display of the `path` or `text` value is appropreate. If the property is intended for labeled display, the label ****SHOULD**** be shown in the UI and the value paired with the label ****SHOULD**** be either data selected from the processing of the `path` property's [JSONPath](https://goessner.net/articles/JsonPath/) string expressions, or the value specified by the `text` property. If neither is present, display of the label and any fallback value is at the election of the implementer.

### Resource Location

Credential Manifests ****should**** be retrievable at known, semantic locations that are generalized across all entities, protocols, and transports. This specification does not stipulate how Credential Manifests must be located, hosted, or retrieved, but does advise that Issuers ****SHOULD**** make their Credential Manifests available via an instance of the forthcoming semantic personal datastore standard being developed by DIF, W3C, and other groups (e.g. Identity Hubs).

### Credential Fulfillment

[[ref:Credential Fulfillments]] are objects embedded within target
[[ref:Claim]] negotiation formats that express how the outputs presented as
proofs to a [[ref:Holder]] are provided in accordance with the outpus
specified in a [[ref:Credential Manifest]]. Embedded
[[ref:Credential Fulfillment]] objects ****MUST**** be located within target
data format as the value of a `credential_fulfillment` property, which is
composed and embedded as follows:

- The `credential_fulfillment` object ****MUST**** be included at the
  top-level of an Embed Target, or in the specific location described in the
  [Embed Locations table](#embed-locations) in the [Embed Target](#embed-target)
  section below.
- The `credential_fulfillment` object ****MUST**** contain an `id` property.
  The value of this property ****MUST**** be a unique identifier, such as a
  [UUID](https://tools.ietf.org/html/rfc4122).
- The `credential_fulfillment` object ****MUST**** contain a `manifest_id`
  property. The value of this property ****MUST**** be the `id` value of a valid
  [[ref:Credential Manifest]].
- The `credential_fulfillment` object ****MUST**** include a `descriptor_map`
  property. The value of this property ****MUST**** be an array of
  _Output Descriptor Mapping Objects_, composed as follows:
    - The `descriptor_map` object ****MUST**** include an `id` property. The
      value of this property ****MUST**** be a string that matches the `id`
      property of the [[ref:Output Descriptor]] in the
      [[ref:Credential Manifest]] that this [[ref:Credential Fulfillment]]
      is related to.
    - The `descriptor_map` object ****MUST**** include a `format` property. The
      value of this property ****MUST**** be a string that matches one of the
      [Claim Format Designation](#claim-format-designations). This denotes the
      data format of the [[ref:Claim]].
    - The `descriptor_map` object ****MUST**** include a `path` property. The
      value of this property ****MUST**** be a
      [JSONPath](https://goessner.net/articles/JsonPath/) string expression. The
      `path` property indicates the [[ref:Claim]] submitted in relation to the
      identified [[ref:Output Descriptor]], when executed against the top-level
      of the object the [[ref:Credential Fulfillment]] is embedded within.
    - The object ****MAY**** include a `path_nested` object to indicate the
      presence of a multi-[[ref:Claim]] envelope format. This means the
      [[ref:Claim]] indicated is to be decoded separately from its parent
      enclosure.
      + The format of a `path_nested` object mirrors that of a `descriptor_map`
        property. The nesting may be any number of levels deep. The `id`
        property ****MUST**** be the same for each level of nesting.
      + The `path` property inside each `path_nested` property provides a
        _relative path_ within a given nested value.


::: example Basic Credential Fulfillment
```json
{
  // NOTE: VP, OIDC, DIDComm, or CHAPI outer wrapper properties would be here

  "credential_fulfillment": {
    "id": "a30e3b91-fb77-4d22-95fa-871689c322e2",
    "manifest_id": "32f54163-7166-48f1-93d8-ff217bdb0653",
    "descriptor_map": [
      {
        "id": "banking_output_2",
        "format": "jwt_vc",
        "path": "$.verifiableCredential[0]"
      },
      {
        "id": "employment_output",
        "format": "ldp_vc",
        "path": "$.verifiableCredential[1]"
      },
      {
        "id": "citizenship_output_1",
        "format": "ldp_vc",
        "path": "$.verifiableCredential[2]"
      }
    ]
  }
}
```
:::

#### Processing of `path_nested` Entries

::: example Nested Credential Fulfillment
```json
{
  "credential_fulfillment": {
    "id": "a30e3b91-fb77-4d22-95fa-871689c322e2",
    "manifest_id": "32f54163-7166-48f1-93d8-ff217bdb0653",
    "descriptor_map": [
      {
        "id": "banking_output_2",
        "format": "jwt_vp",
        "path": "$.outerClaim[0]",
        "path_nested": {
          "id": "banking_output_2",
          "format": "ldp_vc",
          "path": "$.innerClaim[1]",
          "path_nested": {
            "id": "banking_output_2",
            "format": "jwt_vc",
            "path": "$.mostInnerClaim[2]"
          }
        }
      }
    ]
  }
}
```
:::

When the `path_nested` property is present in a [[ref:Credential Fulfillment]]
object, process as follows:

1. For each _Nested Submission Traversal Object_ in the `path_nested` array:
   1. Execute the [JSONPath](https://goessner.net/articles/JsonPath/)
      expression string on the
      [_Current Traversal Object_](#current-traversal-object){id="current-traversal-object"},
      or if none is designated, the top level of the Embed Target.
   1. Decode and parse the value returned from
      [JSONPath](https://goessner.net/articles/JsonPath/) execution in
      accordance with the [Claim Format Designation](#claim-format-designations)
      specified in the object's `format` property. If the value parses and
      validates in accordance with the
      [Claim Format Designation](#claim-format-designations) specified, let the
      resulting object be the
      [_Current Traversal Object_](#current-traversal-object)
   1. If present, process the next _Nested Submission Traversal Object_ in the
      current `path_nested` property.
2. If parsing of the _Nested Submission Traversal Objects_ in the `path_nested`
   property produced a valid value, process it as the submission against the
   [[ref:Output Descriptor]] indicated by the `id` property of the containing
   _Output Descriptor Mapping Object_.

#### Embed Targets

The following section details where the _Credential Fulfillment_ is to be
embedded within a target data structure, as well as how to formulate the
[JSONPath](https://goessner.net/articles/JsonPath/) expressions to select the
[[ref:Claims]] within the target data structure.

##### Embed Locations

The following are the locations at which the `credential_manifest` object
****MUST**** be embedded for known target formats. For any location besides
the top level of the embed target, the location is described in JSONPath syntax.

Target     | Location
---------- | --------
VP         | top-level

#### JSON Schema
The following JSON Schema Draft 7 definition summarizes the rules above:

```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "title": "Credential Fulfillment",
  "type": "object",
  "properties": {
    "credential_fulfillment": {
      "type": "object",
      "properties": {
        "id": { "type": "string" },
        "manifest_id": { "type": "string" },
        "descriptor_map": {
          "type": "array",
          "items": { "$ref": "#/definitions/descriptor" }
        }
      },
      "required": ["id", "manifest_id", "descriptor_map"],
      "additionalProperties": false
    }
  },
  "definitions": {
    "descriptor": {
      "type": "object",
      "properties": {
        "id": { "type": "string" },
        "path": { "type": "string" },
        "path_nested": {
          "type": "object",
            "$ref": "#/definitions/descriptor"
        },
        "format": {
          "type": "string",
          "enum": ["jwt", "jwt_vc", "jwt_vp", "ldp", "ldp_vc", "ldp_vp"]
        }
      },
      "required": ["id", "path", "format"],
      "additionalProperties": false
    }
  },
  "required": ["credential_fulfillment"],
  "additionalProperties": false
}
```

### Appendix

#### Embed Target Examples

##### Credential Fulfillment

<tab-panels selected-index="0">

<nav>
  <button type="button">Verifiable Presentation</button>
</nav>

<section>

::: example Credential Fulfillment - Verifiable Presentation
```json
{
  "@context": [
    "https://www.w3.org/2018/credentials/v1",
    "https://identity.foundation/credential-manifest/fulfillment/v1"
  ],
  "type": [
    "VerifiablePresentation",
    "CredentialFulfillment"
  ],
  "credential_fulfillment": {
    "id": "a30e3b91-fb77-4d22-95fa-871689c322e2",
    "manifest_id": "32f54163-7166-48f1-93d8-ff217bdb0653",
    "descriptor_map": [
      {
        "id": "banking_output_2",
        "format": "jwt_vc",
        "path": "$.verifiableCredential[0]"
      },
      {
        "id": "employment_output",
        "format": "ldp_vc",
        "path": "$.verifiableCredential[1]"
      },
      {
        "id": "citizenship_output_1",
        "format": "ldp_vc",
        "path": "$.verifiableCredential[2]"
      }
    ]
  },
  "verifiableCredential": [
    {
      "comment": "IN REALWORLD VPs, THIS WILL BE A BIG UGLY OBJECT INSTEAD OF THE DECODED JWT PAYLOAD THAT FOLLOWS",
      "vc": {
        "@context": "https://www.w3.org/2018/credentials/v1",
        "id": "https://eu.com/claims/DriversLicense",
        "type": ["EUDriversLicense"],
        "issuer": "did:example:123",
        "issuanceDate": "2010-01-01T19:73:24Z",
        "credentialSubject": {
          "id": "did:example:ebfeb1f712ebc6f1c276e12ec21",
          "accounts": [
            {
              "id": "1234567890",
              "route": "DE-9876543210"
            },
            {
              "id": "2457913570",
              "route": "DE-0753197542"
            }
          ]
        }
      }
    },
    {
      "@context": "https://www.w3.org/2018/credentials/v1",
      "id": "https://business-standards.org/schemas/employment-history.json",
      "type": ["VerifiableCredential", "GenericEmploymentCredential"],
      "issuer": "did:foo:123",
      "issuanceDate": "2010-01-01T19:73:24Z",
      "credentialSubject": {
        "id": "did:example:ebfeb1f712ebc6f1c276e12ec21",
        "active": true
      },
      "proof": {
        "type": "EcdsaSecp256k1VerificationKey2019",
        "created": "2017-06-18T21:19:10Z",
        "proofPurpose": "assertionMethod",
        "verificationMethod": "https://example.edu/issuers/keys/1",
        "jws": "..."
      }
    },
    {
      "@context": "https://www.w3.org/2018/credentials/v1",
      "id": "https://eu.com/claims/DriversLicense",
      "type": ["EUDriversLicense"],
      "issuer": "did:foo:123",
      "issuanceDate": "2010-01-01T19:73:24Z",
      "credentialSubject": {
        "id": "did:example:ebfeb1f712ebc6f1c276e12ec21",
        "license": {
          "number": "34DGE352",
          "dob": "07/13/80"
        }
      },
      "proof": {
        "type": "RsaSignature2018",
        "created": "2017-06-18T21:19:10Z",
        "proofPurpose": "assertionMethod",
        "verificationMethod": "https://example.edu/issuers/keys/1",
        "jws": "..."
      }
    }
  ],
  "proof": {
    "type": "RsaSignature2018",
    "created": "2018-09-14T21:19:10Z",
    "proofPurpose": "authentication",
    "verificationMethod": "did:example:ebfeb1f712ebc6f1c276e12ec21#keys-1",
    "challenge": "1f44d55f-f161-4938-a659-f8026467f126",
    "domain": "4jt78h47fh47",
    "jws": "..."
  }
}
```
:::

</section>
