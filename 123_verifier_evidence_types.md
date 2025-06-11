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
# enhancement-x: Verifier Evicende Types

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories)](#user-stories)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
<!-- /toc -->

## Release Signoff Checklist

- [ ] Enhancement issue in release milestone, which links to pull request in [keylime/enhancements]
- [ ] Core members have approved the issue with the label `implementable`
- [ ] Design details are appropriately documented
- [ ] Test plan is in place
- [ ] User-facing documentation has been created in [keylime/keylime-docs]

## Summary

This enhancement builds on the [Verification API enhancement](https://github.com/keylime/enhancements/blob/master/121-verification-api.md) to also allow for TEE evidence types to be evaluated with the Verification API. This will allow the Keylime Verifier the ability to {de}serialize TEE evidence as well as TPM evidence.

## Motivation

With the desire to allow for Keylime to also attest TEE evidence, there will be a requirement for different evidence types to be interpreted by the Verifier API. Under TEE protections, an agent or third-party *could* send one or more of the following evidence types for evaluation by the Verifier:

- TPM attestation data + CA certificate chain
- SEV-SNP attestation report + CA certificate chain
- Intel TDX attestation report + CA certificate chain
- Confidential device (GPU, hardware RNG, etc) attestation data
- etc...

This is the first step to allow the Keylime Verifier to be able to attest TEE evidence.

### Goals

We want to achieve the following goals:

* Allow TEE evidence to be interpreted by the Verifier alongside TPM evidence.
* Prepare the Keylime Agent and Verifier for the addition of TEE evaluation plugins.

### Non-Goals

The following are not goals for this enhancement:

* Give Keylime full RATS capabilities complete with secret storage, etc. We only intend for Keylime to be able to attest TEE evidence and report results, just as it does with TPM evidence currently.

## Proposal

The Verification API defines the following payload for TPM evidence (at present, the only supported evidence type). This JSON payload will be sent to the `/verify` API:

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

This is strictly for TPM evidence, as the Keylime Verifier cannot interpret any other evidence formats at present. This proposal intends to extend this payload to also include TEE evidence types interpreted by the verifier. This can be done by specifying the evidence type as a header, and parsing the payload data accordingly. For this previous example, the TPM evidence format would be specified by the following:

    {
        "type": "tpm", // TPM evidence type.
        "data": {
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
    }

The verifier would read the `type` parameter to be `tpm` (indicating TPM evidence) and parse the data accordingly. This could be further extended to also include SEV-SNP attestation evidence like so:

    {
        "type": "sev_snp", // SEV-SNP evidence type.
        "data": {
            "attestation_report": attestation_report,
            "vcek": vcek,
            "ca_chain": ca_chain,
        }
    }

This payload will include the following data fields:

* attestation_report: SEV-SNP attestation report (required)
* vcek: SEV-SNP Versioned Chip Endorsement Key (optional)
* ca_chain: SEV-SNP processor CA chain (optional)

We would also intend for Intel TDX and ARM CCA evidence types (among others) as well. We have provided example evidence format for the example.

Users may also like to send *multiple* unique evidences to the Verifier for evaluation at one time. For example, to monitor both the TPM data as well as the SEV-SNP launch environment (for live migration scenarios, etc). To accommodate this, evidence being sent can also be modified to be a *list* of separate evidence formats like so:

    [
        {
            "type": "tpm", // TPM evidence type.
            "data": {
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
        },
        {
            "type": "sev-snp", // SEV-SNP evidence type.
            "data": {
                "attestation_report": attestation_report,
                "vcek": vcek,
                "ca_chain": ca_chain,
            }
        }
    ]

### User Stories

#### Story 1

A SEV-SNP encrypted guest requires evidence verification to unlock some vTPM state and begin running its guest OS. The VM's firmware (running SVSM) will send its evidence to the Verification API and get a result back based on evaluation. Upon a successful result, SVSM will present the signed results from the Keylime Verifier to a key broker service (not part of Keylime) and get a secret to unlock its vTPM state.

#### Story 2

A SEV-SNP encrypted guest has already booted and is running the Keylime Agent. To continously monitor/verify its TEE environment in case of live migrations, it will continuously send its TEE attestation evidence alongside its TPM evidence.

### Notes/Constraints/Caveats

* This proposal only defines the subsequent header and message types that will determine the type of format in which to interpret different evidence types. It does not include the Verifier changes to actually perform the validation of TEE evidence. This modification will be a separate proposal to add Verification "plugins" for the actual evaluation of these messages at a later point.
* We intend for these changes to initially apply to the `/verify` API. However, once that is complete, we intend for the evidence formatting changes to also be included in the standard Agent-Verifier communication as well.

## Design Details

* The Keylime Agent and Verifier will require specific types and {de}serialization methods to {un}marshall the specific evidence they send and receive to/from each other.

### Test Plan

* Tests including a combination of TPM and TEE evidence JSON payloads to ensure correct {de}serialization of these evidence types can be added.
* Backwards compatibility tests to ensure that previous versions of evidence JSON payloads still behave correctly can also be added.
