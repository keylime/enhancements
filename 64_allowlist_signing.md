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
# enhancement-64: Allowlist signing and verification improvements

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

This enhancement proposes improvements to Keylime’s [allowlist](https://keylime-docs.readthedocs.io/en/latest/user_guide/runtime_ima.html#keylime-ima-allowlists) verification mechanisms, including the use of more modern key types (such as x509), addition of allowlist signature verification on the Keylime verifier, and direct integration with a transparency log (for example, the [Sigstore](https://www.sigstore.dev/) project) to improve transparency and verifiability of allowlists.

## Motivation

<!--
This section is for explicitly listing the motivation, goals and non-goals of
this enhancement.  Describe why the change is important and the benefits to users.
-->

This feature introduces more modern key algorithms (starting with ECDSA) and introduces the use of a transparency log for allowlist inclusion proof. In turn, a transparency log allows monitoring for key compromise and protects against a forwarded attack.

### Goals

<!--
List the specific goals of the enhancement.  What is it trying to achieve?  How will we
know that this has succeeded?
-->

Users should be able to sign allowlists using new, modern key algorithms (such as elliptic curves) in addition to GPG. The user should also have an option to ask the tenant or verifier to perform additional verification of allowlist metadata against a transparency log.

### Non-Goals

<!--
What is out of scope for this enhancement?  Listing non-goals helps to focus discussion
and make progress.
-->

- Integrated signing of Keylime allowlists

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation.  The "Design Details" section below is for the real
nitty-gritty.
-->

Currently, the Keylime tenant performs a signature verification step whenever an allowlist is loaded and a signature file and public key are provided as input. This proposal suggests that:

- a new option should be added to this verification step that would also check the provided signature and public key for inclusion in a transparency log
- more modern key types (such as ECDSA) should be accepted during the signature verification step
- The Keylime verifier should have support for verifying user-provided signatures and keys against allowlists stored on the verifier. (There seems to be some community support for this; see [here](https://github.com/keylime/enhancements/pull/65#issuecomment-1063129850).)


### User Stories (optional)

<!--
Detail the things that people will be able to do if this enhancement is implemented.
Include as much detail as possible so that people can understand the "how" of
the system.  The goal here is to make this feel real for users without getting
bogged down.
-->

#### Story 1

- A Keylime deployment with access available to a transparency log (can be deployed privately or a public instance is available and run by the Linux Foundation).
- User provides the tenant with paths to an allowlist, a signature, and a public key using a modern algorithm (e.g. ECDSA)
- Keylime verifies the inputs are consistent, then checks Sigstore for an inclusion proof of those artifacts, before loading them into Keylime

#### Story 2

- A Keylime deployment with access available to a transparency log, which can be deployed privately or as a public instance and run by the Linux Foundation.
- User provides the tenant with paths to an allowlist, a signature, and a public key.
- The tenant performs a POST request against the verifier, uploading the signature, public key, and allowlist to the verifier.
- The verifier uses the inputs to check the allowlist's consistency with the signature and, if desired, for inclusion proof in the specified transparency log.


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

This proposal suggests the [Sigstore](https://www.sigstore.dev/) project for use as a transparency log.

Key terminology: “Rekor” is the transparency log Sigstore uses to store artifacts.

### Allowlist loading

Currently, all allowlist verification happens on the tenant when loading an allowlist file from disk. If the user specifies a signature file and key file using the `--allowlist-sig` and `--allowlist-sig-key` files, the tenant will first verify that the signature matches the provided allowlist file using the public key.

Three fundamental changes/additions should be made to this process.

#### More key types

Right now, the tenant assumes that all signing is done using GPG keys. Instead, the tenant should inspect the provided key file to determine which algorithm is being used (for example, it could be an ECDSA key) and behave accordingly. This will allow for future extensibility around new key algorithms, and will not require the user to specify what kind of key they are providing.

#### Verifier-side signature validation

Verifier-side signature validation should be added in addition to the existing tenant-side signature validation.

To indicate whether verifier-side validation should be performed, there should be a new key in `keylime.conf` under the `cloud_verifier` section called `enforce_allow_list_signatures`, set to `False` by default for now. This functionality shouldn’t require any new flags; specifically, it would use the already existing `--allowlist-sig`, `allowlist-sig-key`, and `--allowlist-name` flags to gather inputs.

When set to `True`, the tenant will upload signing materials alongside an allowlist after it has passed local signature verification, after which the verifier can perform its own validation and return the results as part of the response to the tenant.

In terms of specific implementation:

- The tenant command will collect the following inputs using command flags: allowlist signature, associated signing key, and allowlist name.
- After validating the signing materials locally, the tenant command will perform a POST request against the verifier API, uploading the signature and public key.
- The verifier will use the uploaded signing materials to perform its own signature verification.


#### Optional verification step against a transparency log

Currently, all signature validation happens within `read_allowlist()` in [ima.py](https://github.com/keylime/keylime/blob/master/keylime/ima.py#L408). This proposal suggests an additional, optional verification step that would check provided artifacts for inclusion proof in a transparency log. This assumes the user has already uploaded those artifacts to that transparency log using a separate tool (for example, `rekor-cli`).

To support this new functionality, the tenant command should have two new flags:

- `--allowlist-verify-tlog`: to indicate the user would like to verify against a transparency log. Since both a signature and public key are necessary, existing flags`--allowlist-sig` and `--allowlist-sig-key` should also be set when using this new flag.
- `--tlog-url`: to provide the server address of a compatible transparency log. Defaults to `rekor.sigstore.dev`

In terms of specific implementation:

- When loading an allowlist from disk, if the `--allowlist-verify-transparency-log` flag is set, the tenant will check the transparency log specified by `--transparency-log-url` for an entry that matches the provided artifacts.
- Then, it will verify the inclusion proof using the results returned from the transparency log. If valid, allowlist loading continues; if not, it fails with an error.

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

- A new test should be written to ensure that the allowlist loading process checks for inclusion proof in a transparency log.
- A new test will be written to ensure that verifier-side signature verification and transparency log inclusion works as expected.


### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this enhancement
-->

Since this feature will take the form of a new option in Keylime’s tenant command, there should be no adverse impacts to upgrading or downgrading.

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

N/A.

## Drawbacks

<!--
Why should this enhancement _not_ be implemented?
-->

No drawbacks are known.

## Alternatives

<!--
What other approaches did you consider and why did you rule them out?  These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->

At present, Sigstore is the best known suitale transparency log for the proposed application. While a public good instance is available and maintained by the Linux Foundation, users may also self deploy their own instance - this proposal considers that option as part of the design. Additionally, the proposed design does not preclude the use of other transparency logs in the future.

## Infrastructure Needed (optional)

<!--
Use this section if you need things infrastructure related specific to your enhancement.  Examples include a
new subproject, repos requested, github webhook, changes to CI (travis).
-->

N/A
