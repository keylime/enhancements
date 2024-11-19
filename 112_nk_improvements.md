<!--
**Note:** When your enhancement is complete, all of these comment blocks should be removed.

To get started with this template:

- [ ] **Create an issue in keylime/enhancements**
  When filing an enhancement tracking issue, please ensure to complete all
  fields in that template.  One of the fields asks for a link to the enhancement.  You
  can leave that blank until this enhancement is made a pull request, and then
  go back to the enhancement and add the link.
- [ ] **Make a copy of this template.**
 name it `NNNN-short-descriptive-title`, where `NNNN` is the issue number (with no
  leading-zero padding) assigned to your enhancement above.
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
# enhancement-112: Improvements around the Transport Key (NK)

<!--
This is the title of your enhancement.  Keep it short, simple, and descriptive.  A good
title can help communicate what the enhancement is and should be considered as part of
any review.
-->

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

This enhancement proposal removes partly the need for the transport key and removes the usage of PCR16 to bind the NK to the TPM. 

## Motivation

The initial Keylime architecture did not use mTLS for agent and verifier/tenant communication and required a separate key to ensure confidentiality. 
We no longer need this when mTLS is enabled, further this scheme only works with RSA and not with EC keys.


### Goals

- Remove the usage of PCR16 to bind data to a TPM quote by using TPM2_Certify instead
- Remove need to separately encrypt payload with NK when mTLS is enabled
- Link AK and NK together using TPM2_Certify at registration time

### Non-Goals

- Move the U/V key split to standard key exchange protocol
- make EC keys work when mTLS is disabled

## Proposal

See design details for now.

### User Stories (optional)

#### Story 1 - Sending payload

From a user perspective nothing should change, except that by default no identity quote is happening.

#### Story 2 - Getting identity quote

If the /quote/identity is called, no longer a full quote will be reported, instead the public data of the NK loaded on the TPM and the attestation data and signature output from TPM2_Certify

### Risks and Mitigations

We no longer check the freshness of the binding every time the payload is sent, as we only do this check once during registration.
Further we put trust in the registrar to validate this correctly. This does not change anything in our threat model, as we already rely on the registrar to make sure that the EK and AK belong together. 

## Design Details

In the registrar we introduce a new field called `transport_key`. This is only used when mTLS is not used by the agent, as otherwise the key used for the mTLS server side is equal to the NK.

# When mTLS is enabled
The registration process now looks like this:

1. agent sends the usual information: AK, EK, (EK cert) and mTLS certificate
2. the registrar responds with MakeCredential Challenge and nonce for TPM2_Certify
3. agent does ActivateCredential and loads mTLS key pair as NK onto TPM, then runs TPM2_Certify on NK loaded in TPM
4. agent sends HMAC with the secret and UUID, TPM2_PUBLIC of the NK and the attestation and signature data from TPM2_Certify to registrar
5. registrar validates HMAC checks that the TPM2_PUBLIC part matches the key in mTLS certificate and checks if the nonce is correct.
6. agent is now active in registrar

When adding agent to attestation:

1. tenant gets agent information from registrar
2. tenant uses mTLS certificate to establish a connection with the agent and sends payload and U key
3. tenant adds agent to verifier
4. verifier uses mTLS certificate to establish a connection with the agent and sends K key on successful attestation


# When mTLS is disables
The registration process now looks like this:

1. agent sends the usual information: AK, EK, (EK cert) and NK in `transport_key` field (PEM encoded)
2. the registrar responds with MakeCredential Challenge and nonce for TPM2_Certify
3. agent does ActivateCredential and loads NK onto TPM, then runs TPM2_Certify on NK loaded in TPM
4. agent sends HMAC with the secret and UUID, TPM2_PUBLIC of the NK and the attestation and signature data from TPM2_Certify to registrar
5. registrar validates HMAC checks that the TPM2_PUBLIC part matches the key specified as `transport_key` and checks if the nonce is correct.
6. agent is now active in registrar

When adding agent to attestation:

1. tenant gets agent information from registrar
2. tenant encrypts payload and U key with NK and sends it to the agent
3. tenant adds agent to verifier including the NK
4. verifier uses the NK to encrypt and then send the K key on successful attestation

### Test Plan

- Will be mostly already covered by the existing end-to-end tests
- New tests for validating objects via TPM2_Certify will be added where necessary

### Upgrade / Downgrade Strategy

The identity quote endpoint will be replaced by the output as described in the user story. Otherwise this change is transparent for the user. 

### Dependency requirements

No new dependencies are required.

## Drawbacks

None.

## Alternatives

- Eliminate to NK fully (would require for us to force enable mTLS when using payloads)

