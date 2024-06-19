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
# enhancement-107: TEE Boot Attestation

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

Trusted Execution Environments (TEEs) are a confidential computing and
virtualization technology that allow for the protection/encryption of guest VM
RAM, cache memory, and CPU registers for running sensitive applications on
potentially untrusted hosts within the cloud/edge. With TEEs, data written to
RAM, caches, or registers are encrypted with keys maintained by secure
processors embedded within a CPU, thus protecting confidential VM guests from
other guests on a host, or even the host system itself. With encryption fully
managed by the hardware, only the confidential VM (CVM) itself would be able to
read/write to its own memory, protecting against buggy/malicious hosts spying
on or tampering with the CVM.

TEEs are for protecting applications/users on untrusted platforms. In-fact, the
confidential computing threat model assumes that the host in which a CVM is
running on top of is actively attempting to tamper or spy on guest memory. With
this, users must *ensure* that a host has launched their guest with all proper
TEE protections and didn't perform any other nefarious actions to tamper with
a CVM. That is, CVMs must *attest* their boot environment and prove the
*integrity* of their workload before any sensitive operations can be performed
within a CVM.

This enhancement will introduce a new handler to the keylime verifier strictly
for performing TEE boot attestation. However, with untrusted hosts controlling
guest memory, certain attacks can take place *before* a guest OS is booted and a
keylime agent has been initialized. Timing of boot attestation is critical,
and thus a TEE boot attestation handler cannot assume that it is communicating
with a keylime agent. Furthermore, the TEE handler will release some secret
(whether that is a public key to extend an initial TPM measurement with, a LUKS
key to unlock some initial TPM state, or a combination of both) upon a
successful boot attestation that will be required for guest OS to boot or a
keylime agent to start running. In-effect, this handler would essentially
bootstrap a keylime agent running in a CVM, as the TEE boot environment would
need to be validated before a keylime agent could begin running.

## Motivation

<!--
This section is for explicitly listing the motivation, goals and non-goals of
this enhancement.  Describe why the change is important and the benefits to users.
-->

As CVMs are deployed on untrusted systems, it is reasonable to assume that a
CVM user would also like to take advantage of system integrity monitoring
provided by keylime *in addition to* TEE technology. That is, it is reasonable
to expect that keylime agents will be deployed on CVMs. With that, it is
reasonable to include TEE attestation mechanisms for environments in which
keylime will already be running for system integrity monitoring purposes.

### Goals

<!--
List the specific goals of the enhancement.  What is it trying to achieve?  How will we
know that this has succeeded?
-->
 * Extend keylime verifier to include support for TEE boot attestation.
 * Extend TPM measurements to account for TEE boot attestation within keylime.
 * Fully support confidential computing systems' need for boot *and* runtime
   attestation within keylime.

### Non-Goals

<!--
What is out of scope for this enhancement?  Listing non-goals helps to focus discussion
and make progress.
-->
 * TODO

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation.  The "Design Details" section below is for the real
nitty-gritty.
-->

### User Stories (optional)

<!--
Detail the things that people will be able to do if this enhancement is implemented.
Include as much detail as possible so that people can understand the "how" of
the system.  The goal here is to make this feel real for users without getting
bogged down.
-->

Initially, the TEE handler will work in-tandem with [SVSM], which runs in
a tenant's firmware and provides privileged operations for CVM guests. SVSM will
handle TEE attestation before a guest OS boots. SVSM will be responsible for
unlocking state (provided by the keylime verifier) needed to begin the guest OS
boot process.

#### Story 1
 * User launches a confidential VM, pointing it to the URL of the keylime
   verifier's TEE handler to communicate with for boot attestation.
 * CVM begins running in SVSM. SVSM gathers all necessary information needed for
   TEE attestation. Information is dependant on the specific TEE architecture
   the guest is running on top of (ARM CCA, Intel TDX, AMD SEV-SNP, etc...).
 * SVSM sends all attestation information to TEE handler.
 * TEE handler uses information to verify the boot environment.
 * If boot attestation is successful, TEE handler reads the agent's TEE secret
   from the registrar DB and sends to CVM. If boot attestation fails, TEE
   handler sends error message.
 * SVSM reads response from TEE handler. If successful, SVSM uses the TEE secret
   to unlock some state and begin booting guest OS, with a keylime agent
   eventually initializing and integrity monitoring continuing as normal.
 * SVSM will extend system's TPM state to account for TEE boot attestation.
   *This is intentially vague at the moment, as the exact design is still under
   ongoing discussion*.
 * Agent initializes and begins running as normal. Relevant additions are made
   to verifier to check if TPM measurement includes the data returned from TEE
   handler. *This is also still under ongoing discussion*.

### Notes/Constraints/Caveats (optional)

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above.
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->
 * TODO

### Risks and Mitigations

<!--
What are the risks of this proposal and how do we mitigate.  Think broadly.
For example, consider both security and how this will impact the larger
enhancement ecosystem.

How will security be reviewed and by whom?
-->

The TEE handler will contain a very sensitive secret for unlocking some state
on the CVM. This secret must be properly protected and should only be made
available to a guest that has successfully attested.

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable.  This may include API specs (though not always
required) or even code snippets.  If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->

The keylime verifier will implement a new handler at `/v_/verify/tee` which can
be accessed with a `POST` method. The request body is a JSON object with the
following properties:

- `tee` (_string_) - TEE architecture that the `evidence` will be formatted in.
- `evidence` (_JSON object_) - TEE evidence. This will be deserialized
  differently depending on the TEE architecture specified in `tee`.
- `pubkey_pem` (_string_) - PEM-encoded public key to encrypt the TEE secret
  with upon a successful attestation.

The RESTful parameters of this handler would also need to include an agent UUID,
as TEE secrets are specific to each agent.

Upon a `POST` request to the TEE handler, the handler will deserialize the
request body from JSON to fetch the parameters listed above. The handler will
read the `tee` parameter and deserialize the `evidence` parameter accordingly.

The handler will attest the TEE `evidence`. If successful, the handler will use
the `agent_uuid` from the RESTful parameters, read the agent's TEE secret from
the registrar, and encrypt the secret with the public key found from
`pubkey_pem` in the request body's JSON object.

SVSM will decrypt the resource and use it to unlock some state within the
firmware of the CVM. With that, it will be able to begin the guest OS boot
process.

*PROBABLE*: SVSM will extend its TPM measurement with some state (be it a public
key or certificate) provided by keylime. This will essentially require and
include the TEE boot attestation to be accounted for within an actual TPM
measurement checked at a later point within the keylime verifier. The design of
this is still under ongoing discussion, and this document will be updated when
the design is complete.


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

 * New tests will need to be written specifically for TEE boot attestation
   extension scenarios.

### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this enhancement
-->

TODO

### Dependency requirements

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
No additional dependencies should be required.

## Drawbacks

<!--
Why should this enhancement _not_ be implemented?
-->
No drawbacks are known of.

## Alternatives

<!--
What other approaches did you consider and why did you rule them out?  These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->
 * There are other TEE attestation servers released now, such as the [trustee]
   service. However, using trustee would require keylime and trustee to work
   in-tandem and communicate with each other. It would also require two separate
   servers to be run for attesting one confidential VM (trustee for boot
   attestation, and keylime for runtime attestation and integrity monitoring).
   Keylime already offers the sufficient runtime services needed for integrity
   monitoring, so it is reasonable for it to include extensions for boot
   attestation rather than require another solution to be deployed alongside it.
   This would also make keylime a one-stop solution for running sensitive CVM
   workloads on the cloud or edge.

## Infrastructure Needed (optional)

<!--
Use this section if you need things infrastructure related specific to your enhancement.  Examples include a
new subproject, repos requested, github webhook, changes to CI (travis).
-->
No infrastructure changes needed.

[SVSM]: https://github.com/coconut-svsm/svsm/blob/main/README.md
[trustee]: https://github.com/confidential-containers/trustee/blob/main/README.md
