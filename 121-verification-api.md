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
# enhancement-121: Verification API

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

- [ ] Enhancement issue in release milestone, which links to pull request in [keylime/enhancements]
- [ ] Core members have approved the issue with the label `implementable`
- [ ] Design details are appropriately documented
- [ ] Test plan is in place
- [ ] User-facing documentation has been created in [keylime/keylime-docs]

## Summary

Create new API(s) and tools to allow hosts to check their potential
attestation evidence (PCRs, TPM quote, IMA measurement list, Measured
Boot log, etc) against policy.  This would allow a third party (or simple
tool script shipped with Keylime) to gather the attestation evidence
data on it's own and send it to Keylime for a one-time validation.

No continuous attestation, no trust in the origins of the attestation
evidence, no trust established through key derivation protocol, etc. But
we do answer the "what if" about the attestation evidence would pass
the given policy

Because this request happens out of band and is not part of the regular
registration, tenant-establish-trust and agent-pull cycle of Keylime
it can be considered as a light form of the agent-driven attestation
model. Not useful for fully agent-driven workloads, but would be useful
for one-off enrollment style workloads. It is also useful for CI/CD
pipelines that are testing new versions of host configuration against
new versions of policy before rolling out to production systems.

## Motivation

There are several scenarios where we would like to check some attestation
data to see if they would pass attestation against a given policy without
having the full trust model and continous attestation cycle of a fully
registered and monitored node in Keylime. Maybe it's part of a pre-check
enrollment process, or maybe part of some 3rd party tool that manages
the target node.

These scenarios have different threat and trust models, so as long as
we're explicit about the lack of trust chain in validating the attestation
data against the given policy, it's a useful feature to expose Keylime's
full featured IMA and Measured Boot validation cababilities.

### Goals

1. Create a new API in the verifier to allow for verification of evidence against policy.

2. Create a new one-shot attestation script that would collect data on
a host and verify it using the above API.

### Non-Goals

We are not changing Keylime's core trust model or verifier-driven
attestation cycle.

We are not implementing the proposed agent-driven attestation model.

## Proposal

Create a new API on the verifier at `/verify` that will take the
following example JSON data payload:

    {
        "quote": quote,
        "nonce": nonce,
        "hash_alg": hash_alg,
        "tpm_ek": ek,
        "tpm_ak": ak,
        "tpm_policy": tpm_policy,
        "runtime_policy": runtime_policy,
        "mb_policy": mb_policy,
        "ima_measurement_list": ima_measurement_list,
        "mb_log": mb_log,
    }

The payload will take the following data fields:

* quote - a TPM quote (required)
* nonce - a client create nonce (required)
* hash_alg - the hashing algorithm used for the TPM. Must be standard algorithm supported by Keylime (required)
* tpm_ek - The public EK certificate of the TPM (required)
* tpm_ak - The public AK certificate of the TPM (required)
* tpm_policy - A JSON TPM policy in a format supported by Keylime (optional)
* runtime_policy - A JSON Runtime policy in a format supported by Keylime (optional)
* mb_policy - A Measured Boot policy in a format supported by Keylime (optional)
* ima_measurement_list - The ASCII contents of the IMA measurement file (optional)
* mb_log - The contents of the boot log (optional)

The new API would validate all of this material and then run the
various optional attestation artifacts through Keylime's validation
routines. Those routines would need to be adjusted to handle the case
where there is no known agent. Keylime would not trust the various
artifacts, but instead report whether they would have passed attestation
if they had come from a trusted source.

In the case of attestation failures, we will provide as detailed a
response as we can for the failure(s).

The API will return a response like the following:

    {
        "success": 0,
        "failures": [
            {
                "type": "ima.validation.ima-ng.not_in_allowlist",
                "context": {
                    "message": "File not found in allowlist: /root/evil.sh"
                }
            }
        ]
    }

Separately a new tool named `keylime_oneshot_attestation` will be created
as an example of using this API. This script will gather the information
about the host and take user-supplied attestation policy and send them
to the API for validation. It will serve as a sample client of the new
API as well as being a useful tool for people to use on target hosts.


### User Stories (optional)

<!--
Detail the things that people will be able to do if this enhancement is implemented.
Include as much detail as possible so that people can understand the "how" of
the system.  The goal here is to make this feel real for users without getting
bogged down.
-->

#### Story 1

I have a 3rd party system that handles enrollment of devices and release
of secrets to those devices upon enrollment. I want to use Keylime to
make sure the device hasn't been tampered with since it was shipped
by validating the boot log and measured files on the device. My 3rd
party tool knows the TPM information of the device and will be doing
the collection of the boot and IMA logs. It will also craft custom
Keylime policies based on the device's version. It will then send this
information to a Keylime verifier and receive the one-shot attestation
results. If they pass, it will continue with the device enrollment and
handle the rest of the device's lifecycle on it's own.

#### Story 2

I have a large production system running keylime. All changes to
this system go through a CI/CD pipeline before being pushed out. It is
currently very hard to test changes to keylime policy in this pipeline. It
would be nice to have a simple API on an existing keylime system that
could be used as an integration point to check whether changes to the
system violate existing keylime policies and/or if changes to keylime
policy will continue to pass on existing systems.


### Risks and Mitigations

It will be important to stress in the documentation of this feature that
this is a fundamentally different trust model than Keylime normally has
since there has been no registration with Keylime of this target, nor a
key exchange to establish agent trust. Keylime can't vouch for the trust
worthiness of the data in question, just whether it would pass attestation
if it were trusted. The trust issue needs to be handled by a 3rd party.

This also means that Keylime's normal trust and attestation model is
not changed for normally registered nodes.

### Test Plan

Add new unit tests for the new methods and adjust unit tests for any
methods that need to be altered.

Create a new end-to-end test for the API.

### Upgrade / Downgrade Strategy

This new feature should not require any schema changes. The API will
be net-new and marked as experimental in a new API version. Thus this
feature shouldn't need any upgrade/downgrade strategy.

### Dependency requirements

This feature shouldn't require any new dependencies.

## Drawbacks

This is a brand new idea for Keylime to allow interaction of data from
non-registered hosts. This shouldn't be a problem, but it should be
noted as new model.

## Alternatives

Instead of having a new API we could explore trying to make these
attestation validation procedures available as library functions that
others could embed in their code. This would only really work for Python
based projects though and limit it's usefulness.

## Infrastructure Needed (optional)

This shouldn't have any new infrastructure needs.
