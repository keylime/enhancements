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
# enhancement-16: Remote Allow-List Retrieval

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

## Summary

This enhancement proposes a way to allow Keylime to automatically
import IMA Allow-Lists from external sources. These allow-lists will
follow a prescribed JSON format that allows the `keylime_tenant` to
download, cryptographically verify and then upload these lists to the
verifier. This will be done in versioned manner to allow upgrades and
extensions in the future.

## Motivation

Keylime currently places the burden of creating and managing whitelists
upon the user.  Currently the way to generate these allow-lists is to
air-gap the machine after it’s been installed and configured and then
run a script which crawls the filesystem and records SHA256 hashes into
a file. This file is then loaded into keylime with the `keylime_tenant`
command at which point the machine can be put back into circulation and
keylime will continually verify it’s state.

This extra process is a burden when provisioning new resources for a
cluster. It also means adding keylime to an existing cluster requires
reducing the capacity of that cluster as batches of machines are pulled
out for allow-list generation. Another downside is that the allow-lists
generated are unique per-machine and managing large per-machine
allow-lists is another administrative task on cluster operations.

We want to make this process simpler and more automated for more use
cases. This will not only help Keylime users scale up their usage, but
also increase adoption and make Keylime more attractive for downstream
projects.


### Goals

* Create a simple, versioned JSON format spec for allow-lists
* Make it easy and secure for keylime users to point `keylime_tenant` at remote allow-lists
* Allow users to cryptographically verify the signature or checksum (or both) of the allow-lists
* Document everything and provide at least one real world integration example (Fedora CoreOS)

### Non-Goals

This will be the starting point for many future integrations with keylime, but for now we will not:

* Integrate with other secure supply chain initiatives like in-toto, Tekton/chains or Co-Swids
* Include non-files in the allow-lists (like SELinux Labels, boot arguments, etc)

## Proposal

These are the proposed changes:

* Document the first version of the allow-list JSON spec
* Add the following arguments to `keylime_tenant`

  * `allowlist-url`
    An HTTP URL that points to the remote JSON data file. This URL must
    be secure (HTTPS), and non-redirecting (no 30X HTTP codes) to reduce
    the chance of any MITM attacks. If this option is specified then
    you can't also have the `--whitelist` option present.

  * `allowlist-sig-url`
    An HTTP URL that points to the cryptographic signature of the JSON
    data file. Again, This URL must be secure (HTTPS), and non-redirecting
    (no 30X HTTP codes) to reduce the chance of any MITM attacks. This
    option cannot be specified if `allowlist-url` isn't also present. And
    you must include either this option or `allowlist-sig` or `allowlist-checksum`

  * `allowlist-sig`
    The cryptographic signature of the remote JSON data file. This
    option This option cannot be specified if `allowlist-url`
    isn't also present. And you must include either this option or
    `allowlist-sig-url` or `allowlist-checksum`. This option gives more
    flexibility to the user if the `allowlist-sig-url` doesn't quite
    fit their workflow.

  * `allowlist-key`
    The key used to verify the `allowlist-sig`. This option is required if
    `allowlist-sig` or `allowlist-sig-url` is used.

  * `allowlist-checksum`
    The SHA256 checksum of the allow-list. This can be used to verify
    that the allow-list hasn't been tampered. You must either verify
    the checksum or signature (or both) of the allow-list.

* Expand the way allow-lists are passed between the tenant and the
verifier to use JSON instead of the simple text file that exists now. This
makes the contract more explicit and allows for expansion in the future.

### User Stories

#### Story 1

As a Fedora CoreOS user, I already know every package that could be
present on my system based on my ostree version. I want to just tell
Keylime where to find the list of file hashes for my specific version
and let it do the rest

#### Story 2

I have a secure build process that can produce allow-lists for the
software my team is building. I can store these allow-lists in an external
repository (say an OCI registry). As part of my build/release process I
have the cryptographic signatures of these allow-lists. I would like to
tell keylime about these lists when deploying and provide the signature
to keylime to verify it pulled the expected lists.

### Risks and Mitigations

* We will be updating the internal API between the tenant and the verifier
to use the new JSON spec. Care will be taken to make sure this is done in
a backwards compatible way so existing instances of keylime can continue
to use their existing whitelists.

* We will need to thoroughly review our handling of the external HTTP
connections to make sure they are as secure as possible and don't do
things like accept invalid TLS certificates.

* We will need to thoroughly review our handling of the cryptographic
signatures and checksums

## Design Details

Propsed JSON spec:

{
    "meta": {
        "version":  "",    # Hash format version to allow for future changes if needed
        "generator": ""   # What generated the hash payload
        "timestamp": ""  # When the hash payload was generated
    },
    "release": "",        # Release Identifier. Specifics here may be different for different compound components
    "hashes": {
        "/filesystem/path": [
            {"sha256": "HASH"},  # an object so other hashes could go side by side if needed
            {"sha256": "HASH"},
    ]
}

### Test Plan

TBD

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

### Upgrade / Downgrade Strategy

TBD

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this enhancement
-->

## Drawbacks

If we are concerned about not using existing data formats (like in-toto) we should not implement this design.

## Alternatives

Instead of baking the URL retrieval and Hash verification into
`keylime_tenant` itself, we could create an additional utility that
would perform those actions and then translate the JSON allow-lists
into the current format. Then `keylime_tenant` could be called on the
new resulting whitelist.

I believe this is a less desireable approach because it means we have yet
another script that users would have to run in order to get the desired
integration. While one more step isn't drastic, it does allow more room
for error and creates more friction.


## Infrastructure Needed (optional)

For the keylime side there is no additional infrastructure needed. For
the end-to-end documented example using Fedore CoreOS we will need
to coordinate with that team to make sure the URLs are in place for
allow-list retrieval and verification.
