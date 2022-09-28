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
# enhancement-88: Encapsulate runtime policies in DSSE-signed envelopes

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

This proposal introduces integrated signatures to runtime policies by encapsulating them as [DSSE](https://github.com/secure-systems-lab/dsse) envelopes. This integrated signing standard is an upstream, centralized replacement for the various attached signature formats present in projects like in-toto and TUF (both of which will be using this standard in the near future), and is also supported by the Sigstore project.

This proposal supersedes and builds on the ideas from [enhancement proposal 77](https://github.com/keylime/enhancements/pull/79), authored by Luke Hinds. That proposal focused on mirroring attached signature schemes from in-toto and TUF; since both are moving to DSSE, Keylime should follow suit.

## Motivation

<!--
This section is for explicitly listing the motivation, goals and non-goals of
this enhancement.  Describe why the change is important and the benefits to users.
-->

Keylime's current policy signing implementation requires that signatures be sent alongside policies themselves, as separate payloads. While suitable for some cases, it is also useful to have a single package where the list and the signature travel in the same payload.

Encapsulation in a standardized signature format would also facilitate easier adoption of Keylime policies into other software supply chain security projects.

### Goals

<!--
List the specific goals of the enhancement.  What is it trying to achieve?  How will we
know that this has succeeded?
-->

- Enable a simplified, more secure signature scheme for Keylime policies

### Non-Goals

<!--
What is out of scope for this enhancement?  Listing non-goals helps to focus discussion
and make progress.
-->

- No immediate plan to implement signing into the tenant and no extra flags are required, the verifier will check for the signature and signed body and only attempt to validate if they exist.

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation.  The "Design Details" section below is for the real
nitty-gritty.
-->

Currently, runtime policies travel as their own payload from tenant to verifier when adding a new agent or creating a new policy using the Runtime Policy API. Signatures travel as separate parts of the payload, and are then verified against the policy by the verifier. Users have to input the policy, the policy's signature, and the key used to create the policy signature.

Instead, the tenant would accept DSSE-formatted envelopes and (optionally) the associated public key as input. Keylime would then be able to perform signature integrity checks through the pipeline, and store the unmodified envelope in its database. This would also enable signature checks after upload time, which is currently not possible.

### User Stories (optional)

<!--
Detail the things that people will be able to do if this enhancement is implemented.
Include as much detail as possible so that people can understand the "how" of
the system.  The goal here is to make this feel real for users without getting
bogged down.
-->

#### Story 1

I am deploying Keylime, and I'd like to sign the runtime policy I want to enforce and validate it on the verifier. Before using the Keylime tenant, I create a DSSE-compliant envelope using a private key I use. It contains the policy I wish to use. Once I pass this envelope to the tenant while creating an agent, the envelope is stored in-place on the Keylime verifier, and I can run signature checks on it at will.

#### Story 2

I am deploying Keylime, and I'd like to store my Keylime policy in a transparency log that supports DSSE envelopes (such as Sigstore). Once I fetch my DSSE-encapsulated policy from Sigstore, I can feed it directly to Keylime with no further processing.

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

After implementation of [proposal 70](https://github.com/keylime/enhancements/pull/71/files), a Keylime runtime policy "plaintext" will look like this:

```json
{
    "meta": {
        "version": 0
    },
    "release": 6.5.0,
    "digests": {
        "/root/hello.txt": ["a4dc309f..."]
    },
    "excludes": [],
    "keyrings": {},
    "ima": {
        "ignored_keyrings": [],
        "log_hash_alg": "xxx"
    },
    "ima-buf": {},
    "verification-keys": []
}
```

This payload is unsigned, and Keylime may continue to accept unsigned runtime policies.

#### Signing

To create a signed policy compatible with Keylime, the user would have to create a DSSE-compliant envelope, as detailed by the [spec](https://github.com/secure-systems-lab/dsse/blob/master/protocol.md). Specifically, the DSSE parameters would be:

- `SERIALIZED_BODY` would be the unmodified Keylime policy.
- `PAYLOAD_TYPE` would be something unique to Keylime, e.g. `application/vnd.keylime+json`. IMPORTANT: This must be a Keylime standard.
- `KEYID` could be one of two possibilities:
  - A sha256 hash of the public key used to verify the signature to be created. This acts as a map, and would allow for an out-of-band public key.
  - A base64 encoded X509 v3 certificate in PEM format. This is in-band and can be chained to CA for identity.

Then, a DSSE signature envelope would be created using a compliant signing implementation. The resulting envelope would look something like this, as detailed in the DSSE JSON envelope [spec](https://github.com/secure-systems-lab/dsse/blob/master/envelope.md):

```json
{
    "payload": "<Base64(SERIALIZED_BODY)>",
    "payloadType": "<PAYLOAD_TYPE>",
    "signatures": [{
        "keyid": "<KEYID>",
        "sig": "<Base64(SIGNATURE)>"
    }]
}
```

This payload can be transferred unmodified over the wire from the tenant to verifier, and then validated according to the spec.

#### `KEYID` details

Keys will be handled differently depending on how the `KEYID` was created.

- If the `KEYID` is a sha256 hash of a pubkey, the associated public key will be stored within the verifier at:

`/var/lib/keylime/signing_keys/{hash-of-pubkey}.pkey`

- If the `KEYID` is an x509 certificate, the public key will be in-band, so a separate keystore is not required. the public key can be extracted from the certificate.

In the future, a chain verification from the the signing cert to a CA may be possible.

#### Verification

On the verifier, the DSSE payload will be validated. The flow should look something like this:

- The verifier will check to see if the runtime policy payload is formatted as a DSSE envelope (if not, it will be treated as an unsigned runtime policy)

- The verifier will either decode the x509 certificate and extract the public key, or will capture the sha256 digest and load the mapped key from `/var/lib/keylime/signing_keys/{hash-of-pubkey}.pkey` (as specified above in #`KEYID` details)

- The verifier will then perform a verification of the DSSE envelope, validating that the signature matches the signed body

- If any of the steps above fail, then a verifier API will return a failure code.

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

Unit test will be provided to verify serialization of json input and signature verification.

### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this enhancement
-->

While unsigned runtime policies will continue to be accepted, separate signatures will not be.

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

It will be necessary to ship a compliant DSSE signing and verification implementation with Keylime in order to work with DSSE envelopes. There may be upstream implementations from `securesystemslib` or other libraries used by other DSSE-forward projects like in-toto, and they may be the best option for out-of-band dependencies.

Alternatively, Keylime could also ship its own signing and verification functions that mirror reference implementations seen on the [DSSE repo](https://github.com/secure-systems-lab/dsse).

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

As detailed in proposal 70, Keylime could opt to craft its own payload format. This would reduce cross-compatibility, however, and would limit additional integrations down the line. DSSE has a great page on the [rationale](https://github.com/secure-systems-lab/dsse/blob/master/background.md) for its existence as a standard.

## Infrastructure Needed (optional)

<!--
Use this section if you need things infrastructure related specific to your enhancement.  Examples include a
new subproject, repos requested, github webhook, changes to CI (travis).
-->
