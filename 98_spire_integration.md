<!--
**Note:** When your enhancement is complete, all of these comment blocks should be removed.

To get started with this template:

- [ ] **Fill out this file as best you can.**
  At minimum, you should fill in the "Summary", and "Motivation" sections.
  These should be easy if you've preflighted the idea of the enhancement with the
  appropriate SIG(s).
- [ ] **Merge early and iterate.**
  Avoid getting hung up on specific details and instead aim to get the goals of
  the enhancement clarified and merged quickly.  The best way to do this is to just
  start with the high-level sections and fill out details incrementally in
  subsequent PRs.
-->
# enhancement-98: SPIRE Integration

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
    - [Story 2](#story-2)
  - [Notes/Constraints/Caveats (optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed (optional)](#infrastructure-needed-optional)
<!-- /toc -->

## Release Signoff Checklist

<!--
**ACTION REQUIRED:** In order to merge code into a release, there must be an
issue in [keylime/enhancements] referencing this enhancement and targeting a release**.

For enhancements that make changes to code or processes/procedures in core
Keylime i.e., [keylime/keylime], we require the following Release
Signoff checklist to be completed.

Check these off as they are completed for the Release Team to track. These
checklist items _must_ be updated for the enhancement to be released.
-->

- [ ] Enhancement issue in release milestone, which links to pull request in [keylime/enhancements]
- [ ] Core members have approved the issue with the label `implementable`
- [ ] Design details are appropriately documented
- [ ] Test plan is in place
- [ ] User-facing documentation has been created in [keylime/keylime-docs]

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

## Summary

SPIFFE/SPIRE is an elegant solution to workload identity that is pluggable
in it's node and workload attestors. Keylime would be a perfect candidate
for node attestation if it had a few extra APIs that would allow the
SPIRE agent and SPIRE server to be able to verify the following:

* Is the node where the SPIRE agent being attested by Keylime?
* Is the attestation passing?

We propose new APIs on the agent and verifier to allow SPIRE to verify
these things. But since SPIRE will also need plugins (agent and server)
for the node attestation we have some flexibility in how these APIs
appear. If we keep them generic enough, they could theoretically be used
by any system that wants to independently verify the state of given node
in Keylime.

## Motivation

* To expand Keylime's usefulness and reach in the cloud-native landscape.
* To have a better hardware root-of-trust for software identity
* To have a more complete Zero Trust solution

### Goals

When complete, this proposal will allow SPIRE plugins to be written to
target Keylime as an attestor and provide useful properties in keylime
as selectors in SPIRE. This will allow a user to craft authentication
and authorization policy that takes into account a machine's boot and
file integration attestation state.

### Non-Goals

Although these APIs will be generic, no direct effort will be made to
support other non-SPIRE entities.

## Proposal


### User Stories (optional)

#### Story 1

A developer will be able to develop a SPIRE agent and server plugin
that communitcates with the Keylime agent and verifier to be able to
independenly prove that the agent in question is on the same node as the
SPIRE agent and also that the agent is passing it's attestation policies
in Keylime.

This integration will also pull in various properties of the Keylime setup
(agent configuration, policy, etc) to use as selectors for SPIRE.

### Notes/Constraints/Caveats (optional)

None


### Risks and Mitigations

Care will need to be taken so that we don't leak any sensitive data in
these APIs and that our verification/signing process is secure and leads
to the guarantees we are making (that the SPIRE and Keylime agents are
on the same node).

The security of the information flows has been reviewed by several
members of the Keylime development team as well as SPIRE participants. The
implementation will need thorough review as well.


## Design Details

The following flow is anticipated for the full Keylime SPIRE plugins:

```
┌───────────────────────────────────────────────┐                    ┌───────────────┐
│                                               │                    │               │
│                     Node              #3      │                    │    SPIRE      │
│           ┌───────────────────────────────────┼────────────────────►    SERVER     │
│  ┌────────┴────┐                              │                    │               │
│  │  SPIRE      │       #1                     │                    │               │
│  │  Agent      ◄────────────────┐             │                    └─────┬─────────┘
│  │             │                │             │                          │
│  └─────────┬───┘                │             │                          │ #4
│            │#2             ┌────▼─────┐       │                          │
│         ┌──▼──────┐        │ Keylime  │       │                    ┌─────▼─────────┐
│         │  TPM    │        │  Agent   │       │                    │               │
│         │         │        │          │       │                    │    Keylime    │
│         └─────────┘        └──────────┘       │                    │   Verifier    │
│                                               │                    │               │
└───────────────────────────────────────────────┘                    └───────────────┘
```

This flow has the following steps:

1. SPIRE Agent queries node-local /info API on keylime agent to get information like the Keylime UUID
2. SPIRE Agent creates a nonce that is sent to the TPM’s AK (keylime created) for signing
3. SPIRE Agent sends the information to the Spire Server
4. SPIRE Server queries Keylime Verifier about the agent. Does it exist? Is it passing attestation? If so, can you unencrypt (verify signature) of this nonce? If all are true, then SPIRE attestation passed and identity is issued.


In order to accomodate this flow, this enhancement will consist of the following:

1. A new node-local, non-TLS API on the keylime agent responding the the `/info` path. It will return information about the keylime agent which will be used to not only identity the agent, but also be used to perform a signature verification. A 3rd party can use the credential created by the agent in the TPM to sign a nonce which can then be verified by the verifier. The new API will return the following information:

    * agent_uuid
    * tpm_hash_alg
    * tpm_encryption_alg
    * tpm_signing_alg
    * ek_handle

2. A new API on the verifier that can take a signed payload from a TPM and given agent's UUID verify that it came from a TPM associated with that agent. This will be used to independently verify that the Keylime agent resides on a node with that TPM.

3. An expansion of the existing `/agents` GET API on the verifier to return enough information for use as selectors in SPIRE.


### Test Plan

The individual new APIs will have tests written for them. And the new
SPIRE plugins written to use those APIs will also have their own CI/CD
tests/pipelines to test against those APIs, targetting specific versions
of the Keylime agent and verifier.

### Upgrade / Downgrade Strategy

These will be net-new APIs and will require a minor bump in the Keylime
API version number. It's not believed that they will require database
schema changes, nor any upgrade migrations. As such, there doesn't need
to be a downgrade strategy.

### Dependency requirements

It is not believed that we will require any new dependencies for these
APIs as they will just re-use existing libraries for any cryptographic
signing or verification of those signatures.

## Drawbacks

It's possible that these APIs won't be useful outside of the SPIRE
integration, but it's our belief they will be generic enough to be evolved
for any 3rd party that wants to do deep verification of an node's status
in Keylime.

## Alternatives

It's already possible to create an integration of Keylime with SPIRE
by using the x509pop plugin, but there are several limitations with
this approach:

    * You need to have a key management solution for those certs
    * It's very automatic and requires a lot of setup/configuration for the users
    * It relies on using the payload delivery mechanism of the keylime tenant which some users turn off for security
    * It doesn't propagate any information about the Keylime setup into SPIRE properties for use in auth policy.
    * There's no way to revoke the certificate if attestation fails in the future

This enhancement should allow for full Keylime/SPIRE plugins to fix all
of those problems and make it really easy and convenient for users.

## Infrastructure Needed (optional)

This enhancement shouldn't need any additional infrastructure.
