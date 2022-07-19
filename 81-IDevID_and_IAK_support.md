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
This enhancement proposes leveraging the IDevID and IAK keys/certificates to register each device at the registrar service and enabling the generation and use of LDevID and LAKs. IDevID is an industry standard identity which is issued by manufacturers when the device is built.

## Motivation

<!--
This section is for explicitly listing the motivation, goals and non-goals of
this enhancement.  Describe why the change is important and the benefits to users.
-->
The IEEE 802.1AR Standard [1] defines a secure device identifier (DevID) as "a cryptographic identity that is bound to a device and used to assert the device’s identity".  The initially installed identity is defined as an IDevID/IAK ("I" for initial) and is installed in the factory, where the server, switch, or other device is built, before shipping. The IDevID credential is intended to be usable for the life of the product. The IDevID/IAK is expected to be created at device manufacturing time.

The IEEE 802.1AR Standard can be used together with TPM-based keys and certificates as described in [2]. It applies the IEEE Standard 802.1AR device identity module definition and formatting to keys protected by a TPM 2.0. It addresses ways to incorporate TPM 2.0 created keys into solutions that protect device identities and helps prevent a "malicious endpoint". TPM is a secure Root of Trust for Storage (RTS) that provides protection for private keys, preventing use of keys from one device on another device or with another TPM. The security of the DevID keys provisioning is anchored in the Endorsement Key (EK) and its certificate (issued by the TPM manufacturer).

The IDevID and IAK keys are generated by the TPM at the OEM factory and certified by the OEM CA. The CA enforces the TCG-defined provisioning protocol which uses the EK similarly to how Keylime currently validates colocation of the EK and AK. The IDevID certificates include serial number information, so you can identify the specific device. This is used to bootstrap onboarding of devices for services, including attestation services. Using IDevID, provides important security guarantees to users. It assures them the device is genuine and one that they own (via the serial number in the IDevID). In many cases the devices will be geographically distributed in remote or branch offices, for example. So securely identifying the specific device is important.  

 [1] IEEE Standard for Local and Metropolitan Area Networks - Secure Device Identity at https://standards.ieee.org/standard/802_1AR-2018.html  
 [2] TPM 2.0 Keys for Device Identity and Attestation at https://trustedcomputinggroup.org/wp-content/uploads/TPM-2p0-Keys-for-Device-Identity-and-Attestation_v1_r12_pub10082021.pdf

### Goals

<!--
List the specific goals of the enhancement.  What is it trying to achieve?  How will we
know that this has succeeded?
-->
 * Use IDevID and IAK keys/certificates to register each device at the registrar service
 * Use LDevID and LAK keys/certificates to register each device at the registrar service
 * Provisioning of LDevID and LAKs based on the IAK. The provisioning is done outside of Keylime, or at least as a separate tool.

### Non-Goals

<!--
What is out of scope for this enhancement?  Listing non-goals helps to focus discussion
and make progress.
--> 
 * mTLS usage scenario 

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation.  The "Design Details" section below is for the real
nitty-gritty.
-->
We propose to optionally leverage the IDevID and IAK keys/certificates to register each device at the registrar service and generate the LDevID and LAK keys/certificates. This will enable: 1) further checking of the device based on IDevID information and CA trustchain ; 2) use of IAK for generation of LAK and LDevID ; 3) attestation based on LAK instead of an ephemeral AK.

For compatibility reasons and taking into account different scenarios, the proposal extends to the ones detailed below regarding the possible keys. When EAK is mentioned it means an ephemeral Attestation Key that can be generated at the TPM with access to exporting it's public and private parts. Generally the private part of the EAK is exportable encrypted, so that it can be reimported into the TPM but not used anywhere else, while the private part of IDevID and IAK keys cannot be exported due to TCG defined template restrictions. 

 * EK, EK certificate and EAK: this is the current configuration used in Keylime
 * EK, EK certificate, IDevID, IAK and EAK: this configuration can be used when an IDevID and IAK exists, but the user does not want to maintain a CA for issuing LDevIDs or LAKs.
 * EK, EK certificate, LDevID and LAK: this configuration can be used when a long-lived AK is required, but no IDevID/IAK is available. The proof of co-residency is done between the LDevID/LAK and the EK.
 * EK, EK certificate, IDevID, IAK, LDevID and LAK: this configuration enables issuing LDevID and LAKs based on the IAK (from the OEM) and allows the user/client to issue LAK certificates from their own CA.

Adding support for IDevID and IAK as an option to the Keylime registrar, to the RUST agent, and verifier services, allows our users to take advantage of IDevID and IAK when using Keylime. It will promote use of IDevID for switches and servers, which will improve security for all users.
Using the IAK and IDevID credentials would mean the OEM had already exercised the proof of residency pre-requisites for generating the credentials, making it possible to simplify the registering process to a single exchange from the agent to the registrar service, by skipping the DevID provisioning in the field.

A challenge (nonce) will be sent to the node agent to prove it owns the TPM protected private EAK key, avoiding registering a node that is invalid (does not have access to the private key related to the certificate provided). Explicit proof of possession of the private IAK key is not required as the TPM needs this when using TPM2_Certify in the process, which can use the nonce, and that is used for the proof of co-residency between the IAK and LDevID/LAK/EAK, so proof of possession is implicit

Either the EAK or the LAK can be used for attestation, and in case when there is no IDevID and IAK keys then the EAK is verified using the EK and the Make and Activate credential process. Furthermore, the idea is not to use the IAK for attestation of the device after it is registered, so it's necessary to enroll an EAK. The IAK and IDevID certificates and their associated keys don't expire, as they are intended to last the life of the product, so several years at least. To preserve the cryptographic strength of the keys it is recommended to minimize their use.

Regarding the validation process on how the IAK and IDevID certificates will be accepted: the registrar service will need to be configured with a set of trusted CA certificates for validating IAK and IDevID certificates in order to support a multi-OEM environments. More precisely, when used they will be validated through: 
1) Checking the validity of the certificate chain from the OEM; 
2) Checking the device information (as Serial number) present at the IDevID cert. This is optional.
3) Executing a challenge/response with the agent to ensure it has the private keys. The challenge is not necessary in case TPM2_Certify is used in the process, as previously mentioned.

Regarding the item 1 when checking of the certificate chain, the OEM should supply the IDevID and IAK signed certificates along with the necessary intermediaty certificates used within the signing process. Once the Keylime server is deployed the Root CA Cert of the OEMs should be configured within the registrar service. This way the registrar service will be able to fully validate the trust chain. 

In case the Root CA Cert of the OEM is not available and/or the intermediate certs are not provided, this will block the validation of item 1, but this should be configurable. This will allow using the IDevID and IAK even if the certificates are not available (item 1).

Regarding the validation of LAK and LDevID: the LAK should be verified to be in the same TPM as the IAK provided by the OEM and the IAK certificate shows in which device this key resides on. The LDevID key should be verified to be in the same TPM as the LAK.

### User Stories (optional)

<!--
Detail the things that people will be able to do if this enhancement is implemented.
Include as much detail as possible so that people can understand the "how" of
the system.  The goal here is to make this feel real for users without getting
bogged down.
-->

#### Story 1
<!-- TODO: add IDevID scenario with Keylime using it to make sure the device is authentic. -->
The IDevID plays an important role to securely identify that a device is genuine from a given OEM. Within the registration process, along with information like the EK and EAK, the IDevID key and certificates will be provided.
At the registrar service the OEM Root CA will be configured and the IDevID trust chain will be checked, ensuring the certificates are signed by the OEM CAs. 
In addition to the certificates the signed IDevID certificate Subject will contain the device's SN(Serial Number) and the registrar service will check it against the information obtained from the device.

#### Story 2
<!-- TODO: add LDevID scenario with the user creating its own identity. -->
With the IAK provided by the OEM, the client can issue it's own LDevID signed by it's own CAs. Instead of a long lived certificate as the IDevID from the OEM, the client can renew this certificate on a more frequent basis and securely verify its binding to the IAK and the devices TPM. 
Along with the LDevID, it can create LAKs for different services or workloads, enabling a secure execution of services issuing short lived LAKs according to the demand in place.

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
The risk involved at this proposal is increasing the configuration complexity and by enabling using different chains of trust for the IDs.

For mitigation it is necessary to properly document what is trusted during registration and what is not, besides providing examples on how to configure it correctly.

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable.  This may include API specs (though not always
required) or even code snippets.  If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->
A potential workflow is presented below for the scenario using EK, EK certificate, IDevID, IAK, LDevID and LAK. The other scenarios will include most parts of this workflow. It also worth mentioning that when using an IAK the EK is not needed to enroll another AK (being either an EAK or an LAK). Instead of using make/activate credential, which requires a decryption key, TPM2_Certify is used to prove that the EAK/LAK (including the sensitive area, e.g., the plaintext private key) is co-resident in the same TPM as the IAK.

### Registration from the agent perspective when interacting with the registrar service.
1. Create session (done at get_tpm2_ctx)   
2. Get endorsement certificate from TPM NV index and EK public key (done at create_ek)
3. Regeneration of IAK and IDevID possible info (create create_iak and create_idevid routines). Depending on the configuration, the IDevID/IAK may also require a session.
4. Load the IDevID and IAK and their certificates from a given local path and send them along with the rest of the attestation/measured data to the registrar service
5. Create keys for LAK and LDevID based on the IAK (if possible, load them if already created). When using the TPM2_Certify the nonce should be used here.
6. Encode the necessary data and send it to the registrar service (modify the method do_register_agent to support the new parameters regarding IDevID, LDevID, IAK and LAK keys). 
7. Receive the challenge to prove that is actually has the IDevID, LDevID, IAK and LAK keys. 
8. Sends challenge response back to the registrar service.
9. The agent gets its UUID in case of a successful registering process.

### Registration from the registrar service perspective when interacting with the agent
<!-- TODO: add the specific API changes here./ -->
1. The main endpoints here are do_register_agent() and do_activate_agent() invoked by the agent. The registration will affect the /v{api_version}/agents/{agent_id} exchange, while the activation will affect the /v{api_version}/agents/{agent_id}/activate endpoint. The specific changes might not happen at the API endpoints, but on the data exchanged and the api_version to support the new information.
2. Decode data and parse certificates. This implies modifying the method doRegisterAgent to support new parameters regarding IAK and IDevID. This method exercises a POST to /v{api_version}/agents/{agent_id} at registrar_common.py do_POST method.  
3. Verify IDevID certificate chain of trust + IAK certificate chain of trust
4. Create a challenge to send to the agent for prove the possession of the private keys. It's important to mention that the challenge will change from using Make/ActivateCredential to use TPM2_Certify.
5. Receive the challenge response and process
6. Load and check the certificates for LAK and LDevID keys with the owner/company's CAs. The certificates should be generated using a separate process.
7. Commit data to the local Keylime database, the necessary fields regarding keys and certificates to be included there.  
8. Activate the agent. This implies modifying the method do_activate_agent at keylime\keylime\registrar_client.py to support new parameters regarding IAK and IDevID. This method exercises a PUT to /v{api_version}/agents/{agent_id}/activate at registrar_common.py do_PUT method.  

### Attestation from the verifier perspective after the registration process
The work from the verifier perspective can keep using the ephemeral AK or switch between all the possible scenarios mentioned. 
This is being thought as a multi-step plan and these scenarios will be implemented gradually. 

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
<!-- TODO: Add unit tests where possible and otherwise instructions on how to integrate it into our e2e test suite. -->

### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this enhancement
-->
In case of downgrade: don't use that feature as it will not be available.
In case of upgrade: it will be required to use the new API version for this feature.

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
