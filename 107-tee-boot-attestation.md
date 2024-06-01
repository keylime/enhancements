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

TEEs are for protecting applications/users on untrusted platforms. In fact, the
confidential computing threat model assumes that the host in which a CVM is
running on top of is actively attempting to tamper or spy on guest memory. With
this, users must *ensure* that a host has launched their guest with all proper
TEE protections and didn't perform any other nefarious actions to tamper with
a CVM. That is, CVMs must *attest* their boot environment and prove the
*integrity* of their workload before any sensitive operations can be performed
within a CVM.

This enhancement will introduce the notion of workloads using TEE secure
processors as their root-of-trust, rather than vTPM devices. With this, initial
vTPM state will be derived from successful TEE attestation reports, and certain
vTPM PCRs will be extended to reflect successful TEE attestation.

## Motivation

<!--
This section is for explicitly listing the motivation, goals and non-goals of
this enhancement.  Describe why the change is important and the benefits to users.
-->

As CVMs are deployed on untrusted systems, it is reasonable to assume that a
CVM user would also like to take advantage of system integrity monitoring
provided by Keylime *in addition to* TEE technology. That is, it is reasonable
to expect that Keylime agents will be deployed on CVMs. With that, TEE
attestation support for environments in which Keylime will already be running
for system integrity monitoring purposes will likely be desired.

### Goals

<!--
List the specific goals of the enhancement.  What is it trying to achieve?  How will we
know that this has succeeded?
-->

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

With TEE secure processors acting as the ultimate root-of-trust, much of the
registration and attestation process's behavior should be identical to that as
if vTPM devices were the root of trust. However, some registration steps will be
handled before the a Keylime agent is initialized.

It is critical to establish trust in a CVM as early as possible. As such,
typical CVMs encrypt their root disks, and require a key stored on an
attestation server to unlock the root disk and begin running any operating
systems or applications on the CVM. To get this key from the server, a CVM would
have to fetch its TEE evidence from the secure processor and send it as input to
the server. If the server is able to validate the TEE evidence, it will then
release the key. At this point, CVMs could decrypt their root disks and begin
running their OS and applications trusting that they are in-fact running
confidentially.

[SVSM] is a paravisor that is the first thing to run in a confidential VM. It
can provide trusted services to the guest that are protected from both the host
OS, and from the guest firmware & OS. Under AMD SEV-SNP, SVSM will run at
VMPL-0, while the guest firmware & OS will run at VMPL-3. One of the services
SVSM can provide is a virtual TPM (vTPM). By performing a TEE attestation, SVSM
can establish trust in the confidential hardware environment and link the CVM
attestation report to the virtual TPM identity, enabling the guest OS to then
trust the vTPM.

A vTPM will be provided by the paravisor (SVSM) to ensure the integrity of
software running on a CVM (i.e. runtime attestation of CVMs). The intention is
to enable support for encrypted disks, whose keys are sealed using a key derived
from the vTPM.

A vTPM may either be ephemeral in which case it is effectively re-manufactured
on every boot (fresh primary seeds & empty NVRAM), or  persistent, in which case
it has state (primary seeds & NVRAM) maintained across (re)boots.

With a stateful vTPM, the CVM paravisor initially has access to some storage
that contains the encrypted vTPM state. The state is encrypted with a vTPM State
Key, which is in turn encrypted with a key stored on a TEE attestation server
(known as the TEE Attestation Key). The vTPM State Key sent to the attestation
server should only be decrypted once the attestation server has successfully
validated the CVM hardware attestation report. Once the vTPM State Key is
decrypted, the vTPM state can be decrypted and initialized, the VM rootfs can be
unsealed, and the application/OS can begin running.

TEE attestation in-itself is a "blocking operation". That is, if TEE attestation
fails, the vTPM state cannot be decrypted. Subsequently, since the vTPM state
cannot be decrypted, the VM's root disk cannot be unsealed. Without a successful
TEE attestation, the application, OS, etc cannot run. The keylime registrar and
verifier do _not_ block, but merely report data about running applications back
to the tenant. With that, we will introduce another component to the Keylime
tenant, that of a TEE broker (so-far known as [limebroker]). The TEE broker will
essentially have two interfaces to begin: providing the public components of the
randomly-generated TEE Attestation Key to encrypt vTPM State Keys, as well as
attesting TEE evidence from clients and decrypting secrets with the TEE
Attestation Key if successful.

There are two user stories that will be discussed in detail, that being pre-boot
attestation with SVSM and post-boot attestation from the keylime agent on an
untrusted system.

#### Story 1: Pre-boot attestation

As previously mentioned, SVSM is a paravisor with an initial responsibility to
attest its TEE evidence, initialize some vTPM state, and use the vTPM to unlock
the root disk and begin running.

However, each component (LUKS disk sealed w/ vTPM, vTPM state encrypted
with a vTPM State Key, and vTPM State Key encrypted with a TEE Attestation Key
stored on an attestation server) must be provisioned correctly.

##### Pre-boot VM Provisioning

As mentioned before, CVM root disks are encrypted, with decryption reliant upon
successful TEE attestation. In the VM provisioning phase, a user provisions an
encrypted VM image sealed with its vTPM state, which is in-turn encrypted with a
vTPM State Key, and which is then encrypted with a key within the TEE broker
(known as the TEE Attestation Key). This step involves a TEE broker endpoint to
fetch the public components of the TEE Attestation Key and encrypting the vTPM
State Key with it.

- User creates an encrypted disk image instance from a plain text image
  template.
- User manufactures initial vTPM state for the CVM.
- User LUKS-encrypts rootfs with randomly-generated passphrase.
- User seals rootfs passphrase to vTPM.
- User creates "vTPM State Key", encrypts vTPM state with vTPM State Key.
- User makes POST request to Keylime TEE broker endpoint to fetch TEE
  Attestation Key. Keylime TEE broker replies with public key.
- User encrypts vTPM State Key with public TEE Attestation Key.
- User stores vTPM state (encrypted with vTPM State Key) and vTPM State Key
  (encrypted with TEE Attestation Key) along with encrypted root disk.

A diagram showing the VM provisioning process:
```
                                      ┌────────────────┐
                                      │                │
                                      │ ┌────────────┐ │
                                      │ │VM rootfs   │ │
                                      │ │Sealed w/   │ │
┌──────┐                              │ │key derived │ │
│ VM   │    ┌──────────────┐          │ │from vTPM   │ │
│ Disk ├───►│ Provisioning ├─────────►│ └────────────┘ │
│ Image│    │     Tool     │          │                │
└──────┘    └───┬─────▲────┘          │ ┌────────────┐ │
                │     │               │ │vTPM State  │ │
         GET    │     │    TEE        │ │encrypted   │ │
         TEE    │     │Attestation    │ │w/ vTPM     │ │
     Attestation│     │Public Key     │ │State Key   │ │
     Public Key │     │               │ └────────────┘ │
              ┌─▼─────┴─┐             │                │
              │  TEE    │             │ ┌────────────┐ │
              │ Broker  │             │ │vTPM State  │ │
              └─────────┘             │ │Key         │ │
                                      │ │encrypted   │ │
                                      │ │w/ TEE      │ │
                                      │ │Attestation │ │
                                      │ │Key         │ │
                                      │ └────────────┘ │
                                      │   Encrypted    │
                                      │     Image      │
                                      └────────────────┘
```
2. Pre-boot VM TEE Attestation

With the image created and the CVM now running, SVSM is able to attest the CVM's
TEE evidence, decrypt its vTPM State Key, and unlock its root disk. SVSM will
interact with the Keylime TEE Broker's attestation interface to send its
evidence and decrypt its vTPM State Key.

- Since SVSM relies on the untrusted host for networking, all incoming secrets
  sent upon a successful attestation must be encrypted with a key that only the
  CVM controls. Therefore, SVSM generates a "Secret Integrity Key" that the TEE
  broker must use to encrypt all secrets sent back to SVSM. If attestation is
  successful, then the server can be confident that the private Secret Integrity
  Key is stored in encrypted VM memory and cannot be reached from the untrusted
  host. Therefore, by encrypting each secret sent to SVSM with the Secret
  Integrity Key, the server can be confident that the untrusted host is unable
  to decrypt any secrets meant for the confidential guest.
- To ensure freshness of attestation reports, SVSM will request a nonce hash
  from Keylime's TEE broker. The broker will reply with a randomly-generated
  32-byte hash.
- SVSM will produce a hash including the public components of the Secret
  Integrity Key and nonce from TEE broker. This hash will be included in the TEE
  evidence/attestation report.
- SVSM will POST the TEE evidence, Secret Integrity Key public components, and
  encrypted vTPM State Key to the attestation server.

- TEE broker will attest the report (with TEE certificates) and report a
  pass/fail.

UPON SUCCESSFUL ATTESTATION
- TEE broker will decrypt the vTPM State Key that was previously encrypted
  with its TEE Attestation Key.
- As mentioned before, SVSM provides a Secret Integrity Key to prevent the
  untrusted host from spying on decrypted secrets. The vTPM State Key is
  encrypted with this key and sent back to SVSM to decrypt.

```
                    3. Decrypt      4. Encrypt        
                       vTPM State      vTPM State Key 
                  ┌──► Key w/ TEE  ──► w/             
                  │    Attestation     Secret Integrity
                  │    Key             Key            
             ┌────┴────┐                   │
             │Broker   │                   │
             └─┬─────▲─┘                   │          
               │     │                     │          
1. Attestation │     │2. Pass/Fail         │          
   Report      │     │                     │    ┌────┐
        +      │     │                     └───►│SVSM│
   Certificates│     │                          └────┘
               │     │                                
            ┌──▼─────┴──┐                             
            │TEE        │                             
            │Attestation│                             
            │Verifier   │                             
            └───────────┘                             
```

- SVSM will decrypt its vTPM State Key with its Secret Integrity Key.
- SVSM will decrypt vTPM state with vTPM State Key.
- SVSM will derive key from vTPM state and use to unseal rootfs.
- SVSM will begin running OS under decrypted vTPM.

Post-boot Attestation
---------------------

With the CVM's OS booted and Keylime agent started, no functional changes are
requried by the agent. The agent has a view of the vTPM and will begin the
Keylime registration/activation/enrollment/runtime attestation phases as normal.

However, there is a desire for the Keylime verifier to be able to perform TEE
attestation with evidence provided by an agent upon enrollment. To support this
use-case, we will also offer HTTP endpoints to the TEE verifiers within the TEE
broker. With this, the Keylime verifier will be able to access the TEE verifiers
to prove legitimacy of TEE evidence provided by agents.

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

Keylime will store a TEE Attestation Key used to encrypt vTPM State Keys for
all encrypted guests attesting with the server. If this key is leaked, all of
the images could potentially be decrypted and secrets can be leaked.

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable.  This may include API specs (though not always
required) or even code snippets.  If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->


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
   service. However, using trustee would require Keylime and trustee to work
   in-tandem and communicate with each other. It would also require two separate
   servers to be run for attesting one confidential VM (trustee for boot
   attestation, and Keylime for runtime attestation and integrity monitoring).
   Keylime already offers the sufficient runtime services needed for integrity
   monitoring, so it is reasonable for it to include extensions for boot
   attestation rather than require another solution to be deployed alongside it.
   This would also make Keylime a one-stop solution for running sensitive CVM
   workloads on the cloud or edge.

   However, we have provided support in the SVSM attestation proxy for other
   "backends" (i.e. a server capable of attesting TEE evidence) in addition to
   the Keylime TEE broker. It is entirely within the capabilities of the SVSM
   proxy to implement support for the Trustee server.

## Infrastructure Needed (optional)

<!--
Use this section if you need things infrastructure related specific to your enhancement.  Examples include a
new subproject, repos requested, github webhook, changes to CI (travis).
-->
No infrastructure changes needed.

[SVSM]: https://github.com/coconut-svsm/svsm/blob/main/README.md
[trustee]: https://github.com/confidential-containers/trustee/blob/main/README.md
[limebroker]: https://github.com/tylerfanelli/limebroker.git
