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
# enhancement-81: Adding support for IDevID and IAK

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

<!--
This section is incredibly important for producing high quality user-focused
documentation such as release notes or a development roadmap.  It should be
possible to collect this information before implementation begins in order to
avoid requiring implementers to split their attention between writing release
notes and implementing the feature itself. Reviewers
should help to ensure that the tone and content of the `Summary` section is
useful for a wide audience.

A good summary is probably at least a paragraph in length.
-->
This enhancement proposes leveraging the IDevID and IAK keys/certificates to register each device at the registrar service and enable attestation based on the IAK instead of using an ephemeral AK (Attestation key). IDevID is an industry
standard identity which is issued by manufacturers when the device is built.

## Motivation

<!--
This section is for explicitly listing the motivation, goals and non-goals of
this enhancement.  Describe why the change is important and the benefits to users.
-->
The IEEE 802.1AR Standard [1] defines a secure device identifier (DevID) as "a cryptographic identity that is bound to a device and used to assert the device’s identity".  The initially installed identity is defined as an IDevID/IAK (“I” for initial) and is installed in the factory, where the server, switch, or other device is built, before shipping. The IDevID credential is intended to be usable for the life of the product. The IDevID/IAK is expected to be created at device manufacturing time.  

The IEEE 802.1AR Standard can be used together with TPM-based keys and certificates as described in [2]. It applies the IEEE Standard 802.1AR device identity module definition and formatting to keys protected by a TPM 2.0. It addresses ways to incorporate TPM 2 created keys into solutions that protect device identities and helps prevent a “malicious endpoint”. TPM is a secure Root of Trust for Storage (RTS) that provides protection for private keys, preventing use of keys from one device on another device or with another TPM. The security of the DevID keys provisioning is anchored in the Endorsement Key (EK) and its certificate (issued by the TPM manufacturer).

The keys are generated by the TPM at the factory and certified by an OEM CA. The CA enforces the TCG-defined provisioning protocol which uses the EK similarly to how Keylime validates colocation of the EK and AK. The IDevID certificates include serial number information, so you can identify the specific device. This is used to bootstrap onboarding of devices for services, including attestation services. Using IDevID, provides important security guarantees to users. It assures them the device is genuine and one that they own (via the serial number in the IDevID). In many cases the devices will be geographically distributed in remote or branch offices, for example. So securely identifying the specific device is important.  

### Goals

<!--
List the specific goals of the enhancement.  What is it trying to achieve?  How will we
know that this has succeeded?
-->
 * Use IDevID and IAK keys/certificates to register each device at the registrar service
 * Enable attestation based on the IAK instead of using an ephemeral AK (Attestation key)

### Non-Goals

<!--
What is out of scope for this enhancement?  Listing non-goals helps to focus discussion
and make progress.
-->
 At least at this first step:
 * mTLS usage scenario (at first)
 * Add support for LDevID and LAKs

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation.  The "Design Details" section below is for the real
nitty-gritty.
-->
At a first step, instead of using an ephemeral AK (Attestation key), we propose to optionally leverage the IDevID and IAK keys/certificates to register each device at the registrar service. This would enable attestation based on the IAK.  

Adding support for IDevID and IAK as an option to the Keylime registrar and verifier services, as for the RUST agent, will allow our users to take advantage of IDevID and IAK when using Keylime. It will promote use of IDevID for switches and servers, which will improve security for all users.
Using the IAK and IDevID credentials would mean the OEM had already exercised the proof of residency pre-requisites for generating the credentials, making it possible to simplify the registering process to a single exchange from the agent to the registrar service, by skipping the DevID provisioning in the field.

If required, it would be possible to send a challenge (nonce) to a node agent to prove it owns the TPM protected private IAK key. A nonce challenge to make sure that the node can access the IAK private key is not mandatory but would be good just for avoiding registering a node that is invalid (does not have access to the private key related to the certificate provided). Even without this, the node would not be able to attest later.


### User Stories (optional)

<!--
Detail the things that people will be able to do if this enhancement is implemented.
Include as much detail as possible so that people can understand the "how" of
the system.  The goal here is to make this feel real for users without getting
bogged down.
-->

#### Story 1

#### Story 2

### Notes/Constraints/Caveats (optional)

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above.
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->

### Risks and Mitigations

<!--
What are the risks of this proposal and how do we mitigate.  Think broadly.
For example, consider both security and how this will impact the larger
enhancement ecosystem.

How will security be reviewed and by whom?
-->


## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable.  This may include API specs (though not always
required) or even code snippets.  If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->
A potential workflow is presented below:

### Registration from the agent perspective when interacting with the registrar service
1. Create session (done at get_tpm2_ctx)   
2. Get endorsement certificate from TPM NV index and EK public key (done at create_ek)
3. Regeneration of IAK and IDevID possible info (create create_iak and create_idevid routines)
4. Load the IDevID and IAK and IDevID certificates from a given local path and send them along with the rest of the attestation/measured data to the registrar service.
5. Encode attestation data and send it to the registrar service (modify the method do_register_agent to support the new parameters regarding IAK and IDevID).  
6. Receive the confirmation from registrar service.

### Registration from the registrar service perspective when interacting with the agent
1. The main endpoints here are do_register_agent() and do_activate_agent() invoked by the agent.
2. Decode data and parse certificates. This implies modifying the method doRegisterAgent to support new parameters regarding IAK and IDevID. This method exercises a POST to /v{api_version}/agents/{agent_id} at registrar_common.py do_POST method.  
3. Verify IDevID certificate chain of trust + IAK certificate chain of trust
4. Commit data to the local Keylime database, the necessary fields regarding IAK and IDevID needs to be included there.  
5. Activate the agent. This implies on modifying the method do_activate_agent at keylime\keylime\registrar_client.py to support new parameters regarding IAK and IDevID. This method exercises a PUT to /v{api_version}/agents/{agent_id}/activate at registrar_common.py do_PUT method.  
6. Send confirmation to the agent

### Attestation from the verifier perspective after the registration process
In general lines what was being executed with the ephemeral AK can now follow with the IAK. This point will have more insights soon.

### Test Plan

<!--
**Note:** *Not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy.  Anything
that would count as tricky in the implementation and anything particularly
challenging to test should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations).
-->
TBD

### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this enhancement
-->
TBD

### Dependencie requirements

<!--
If your new change requires new dependencies, please outline and demonstrate that your selected dependency
is well maintained and packaged in Keylime's supported Operating Systems (currently Debian Stable
and as of time writing Fedora 32/33).

During code implementation you will also be expected to add the package to CI , the keylime ansible role and
keylimes main installer (`keylime/installers.sh`).

If the package is not available in the supported Operated systems, the PR will not be merged into master.

Adding the package in `requirements.txt` is not sufficent for master which is where we tag releases from.

You may however be able to work within an experimental branch until a package is made available. If this is
the case, please outline it in this enhancement.

-->
It is a feature that will be optional to use, but will require a device with IDevID and IAK keys issued by the OEM at its factory.

## Drawbacks

<!--
Why should this enhancement _not_ be implemented?
-->

## Alternatives

<!--
What other approaches did you consider and why did you rule them out?  These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->

## Infrastructure Needed (optional)

<!--
Use this section if you need things infrastructure related specific to your enhancement.  Examples include a
new subproject, repos requested, github webhook, changes to CI (travis).
-->
