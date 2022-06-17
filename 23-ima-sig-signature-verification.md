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
# enhancement-NNNN: Your short, descriptive title

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

Keylime currently only supports verification of IMA measurement lists against
an allowlist (whitelist) and exclude list. This proposal suggests to extend
Keylime to support IMA signature verification by supporting the 'ima-sig'
template.

## Motivation

<!--
This section is for explicitly listing the motivation, goals and non-goals of
this enhancement.  Describe why the change is important and the benefits to users.
-->

The allowlist that is used to verify the IMA measurements received from a
system may be a huge list with multiple thousand entries. The size of
the allowlist primarily depends on the number of software packages installed
on a system and thus the expected number of possible measurements coming from
that system. An alternative to allowlist reconciliation is to verify file
signatures using a few keys that were used to sign immutable files on such a
system. A combination of both is also possible.

Having an exclude list to skip over measurements on mutable files will still be
necessary since those cannot be signed.

### Goals

<!--
List the specific goals of the enhancement.  What is it trying to achieve?  How will we
know that this has succeeded?
-->

The goal of this extension involves the following:
- Support for parsing 'ima-sig' template entries in the IMA measurement list
- Support for signature verification of each signature found in an 'ima-sig'
  template using per-system public keys
- Registration of per-system public keys to be used for signature verification

### Non-Goals

<!--
What is out of scope for this enhancement?  Listing non-goals helps to focus discussion
and make progress.
-->

The following is a list of non-goals:
- Addition of support for other IMA templates besides the proposed 'ima-sig'
  template; others can be supported later through separate enhancements


## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation.  The "Design Details" section below is for the real
nitty-gritty.
-->

This proposal will develop and deliver the following features:
- The verifier will be extended to support parsing the 'ima-sig' template
- The verifier will be extended to support signature verification on
  signatures found in a 'ima-sig' template entry using per-system
  registered public keys (RSA, DSA, EC, etc)
- The verifier will require that each IMA measurement list entry will
  either be verifiable with a key or can be reconciled against
  a allowlist or exclude list; every IMA measurement list entry must
  be 'covered'
- The tenant client tool will be extended to support registering
  per-system public keys to be used for signature verification
- Test cases will be written that read in the example keys and
  IMA measuement lists and perform signature verification along
  with alllwlist and exclude list reconciliation

### Notes/Constraints/Caveats (optional)

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above.
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->

- Signature verification may be more CPU intensive than verification
  against a small allowlist of measurements; on the other hand
  the advantage of signature verification is that it only needs a few
  keys rather than a possibly huge allowlist
- Every IMA measurement entry should be covered either using a
  measurement allowlist reconciliation, be part of an exclude list, or
  the signature verification has to succeed; the complexity on
  the user side is to setup a system by signing some immutable files
  while leaving others to allowlist and putting mutable files
  into an exclude list
- A prototype of this extension is being built here:
  https://github.com/stefanberger/keylime/commits/ima_sig

### Risks and Mitigations

<!--
What are the risks of this proposal and how do we mitigate.  Think broadly.
For example, consider both security and how this will impact the larger
enhancement ecosystem.

How will security be reviewed and by whom?
-->

TBD

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable.  This may include API specs (though not always
required) or even code snippets.  If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->

To make it easy for a client, the tenant tool will be able to read
public key files in various format, such as PEM or DER formatted
public keys, x509 certificates, or even private keys. Only the public
key parts will be stored in the Keylime database.

The existing exclude list will be used for skipping of mutable files.
If no allowlist is passed via the tenant tool, only signatures Will be
verified. All IMA measurement list entries that have signatures must be verifiable
with one of the provided public keys, otherwise the file must be in the excluded
list.
If an allowlist, exclude list, and keys for signature verification are provided
then each measurement list entry that is not skipped due to the exclude list
must pass both the signature verification and be in the allowlist. Another
model would be to require signature validation for entries that show a signature
and be in the allowlist otherwise. If such a model is valid, then this could be
enabled with some sort of evaluation policy (a simple flag). Different behavior
on a per measurement list entry basis may be too complicated to setup for a user.

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

For testing we will use sample IMA measurement lists that have the 'ima-sig'
template and file signatures, along with measurement whitelists and excluded
lists and with various formats of keys for usage in signature verification.
Test cases will verify the successes and intended failures to verify 'ima-sig'
measurement lists.

### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this enhancement
-->

TBD

## Drawbacks

<!--
Why should this enhancement _not_ be implemented?
-->

No known drawbacks.

## Alternatives

<!--
What other approaches did you consider and why did you rule them out?  These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->

No known alternatives.

## Infrastructure Needed (optional)

<!--
Use this section if you need things infrastructure related specific to your enhancement.  Examples include a
new subproject, repos requested, github webhook, changes to CI (travis).
-->
