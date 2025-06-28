# enhancement-126: `/verify/evidence` API JWT Response

<!--
A table of contents is helpful for quickly jumping to sections of a enhancement and for
highlighting any additional information provided beyond the standard enhancement
template.
-->

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories (optional)](#user-stories-optional)
    - [Story 1](#story-1)
  - [Notes/Constraints/Caveats (optional)](#notesconstraintscaveats-optional)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed (optional)](#infrastructure-needed-optional)
<!-- /toc -->

## Release Signoff Checklist

- [ ] Enhancement issue in release milestone, which links to pull request in [keylime/enhancements]
- [ ] Core members have approved the issue with the label `implementable`
- [ ] Design details are appropriately documented
- [ ] Test plan is in place
- [ ] User-facing documentation has been created in [keylime/keylime-docs]

## Summary

The `/verify/evidence` API allows for a third party to gather attestation
evidence on its own and send it to Keylime for a one-time validation.
Enhancement proposal [#121](https://github.com/keylime/enhancements/blob/master/121-verify-evidence-api.md#enhancement-121-verification-api) allows for Trusted Execution Environment (TEE) evidence to be
evaluated along with TPM evidence to support confidential computing use-cases.

## Motivation

In many confidential computing deployments, there exists another third party
which evaluates claims from an attestation service and determines if a secret or
confidential data should be released. One such example is the [Trustee Key
Broker Server (KBS)] (https://github.com/confidential-containers/trustee/blob/main/kbs/docs/kbs_attestation_protocol.md). The method in which the KBS uses to evaluate such claims is through a [JSON Web Token (JWT)] (https://jwt.io/introduction) evaluation.

At present, the `/verify/evidence` API returns a valid/invalid JSON message
based on the evaluation of the given evidence. This is sufficient for clients
looking to evaluate their evidence for validity, but is insufficient for third
parties (such as the KBS) needing to verify claims from a client that indicates
to have attested their evidence with Keylime.

To support this use-case, a JWT response can be returned by the
`/verify/evidence` API upon successful attestation. A client could then use this
JWT to validate it was attested by the Keylime Verifier.

### Goals

1. Return a signed JWT response from the Keylime Verifier upon successful attestation with the `/verify/evidence` API.

### Non-goals

1. Tie TEE attestation with Keylime to the Trustee KBS or any other third party project.

## Proposal

To support this use-case, a JWT response can be returned by the `/verify/evidence` API on successful attestation. Rather than a "valid" message like returned now:

```
{
    "valid": 1
}
```

A JWT response including the parsed claims from the `/verify/evidence` API can be included like so:
```
{
    "valid": 1,
    "jwt": $ENCODED_JWT
}
```

With this, a third party such as the KBS can parse and validate claims from the `/verify/evidence` API and release secrets to a client in a known valid state with confidence that the Keylime Verifier has validated the evidence.

Note that this does not effect the failure message from the `/verify/evidence` API. If the `/verify/evidence` API does not successfully attest the claims of a client, a JSON error message indicating which claims have failed is sufficient to return, like exemplified in the original enhancement proposal:

```
{
    "valid": 0,
    "failures": [
        {
            "type": "ima.validation.ima-ng.not_in_allowlist",
            "context": {
                "message": "File not found in allowlist: /root/evil.sh"
            }
        }
    ]
}
```

Since validation fialed, no JWT was released. Therefore, no claims can be validated by a third party (as intended).

### User Stories (optional)


#### Story 1

Initially, this API intends to support TEE attestation with SVSM (in-tandem with KBS), but can serve as a general method for third parties to verify that the Keylime Verifier attested a client's claims, whether they be TPM or TEE claims. As for the SVSM attestation use-case, the following user story illustrates how a JWT response would be used.

- Client gathers its TEE evidence, sending it to the `/verify/evidence` API.
- Keylime Verifier attests the TEE evidence from the client. If successful, will produce a JWT with the claims verified by the Verifier. Verifier will return JSON response with JWT back to client.
- Client will produce JWT to KBS.
- KBS will verify that the Keylime Verifier signed the JWT to determine legitimacy. If legitimate, KBS will release a secret to the client.
- Client will use secret released from KBS to unlock some state within the TEE.

### Test Plan

Add new unit tests to verify the JWT response from the `/verify/evidence` API is provisioned correctly.

Modify the end-to-end test for the API.

### Upgrade / Downgrade Strategy

The `/verify/evidence` API is marked as experimental in a new API version. Thus this feature shouldn't need any upgrade/downgrade strategy.

### Dependency requirements

This feature shouldn't require any new dependencies.

## Infrastructure Needed (optional)

This shouldn't have any new infrastructure needs.
