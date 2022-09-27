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
# enhancement-86: Additional IMA policy hash metadata

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

This enhancement proposes a new field within Keylime's IMA policy format allowing for additional metadata to be specified for particular file hashes. This metadata could be used to enforce more stringent checks for particular files on the Keylime verifier or agent.

## Motivation

<!--
This section is for explicitly listing the motivation, goals and non-goals of
this enhancement.  Describe why the change is important and the benefits to users.
-->

Currently, Keylime utilizes IMA policies (otherwise known as "allow/exclude lists") as the source of truth for what is allowed to run on agent systems. These policies rely on a simple allow/deny pattern: the file at path X must have digest Y, or it won't be allowed to run. While functional and workable, this system is also rather primitive - Keylime does not know or care about where an individual file came from, for example, or how it got to the end-user system. Provenance information can still be incredibly useful, however, and Keylime would be able to provide additional, stronger security guarantees if it could more thoroughly investigate and verify file provenance (among other verification checks).

A number of systems can be built to facilitate these additional checks, but underpinning all of them is the need for more direct metadata about how to perform them, and on which files. Not every file will have the same level of attestations available, and some may be more important than others. The ability to adjust the desired level of attestation per file depends on the availability of additional policy information, which this proposal attempts to tackle via an extension to the current IMA policy system.

### Goals

<!--
List the specific goals of the enhancement.  What is it trying to achieve?  How will we
know that this has succeeded?
-->

- Allow policies to specify additional information about individual files
- Enable new kinds of more nuanced file verification beyond blanket allow/deny

### Non-Goals

<!--
What is out of scope for this enhancement?  Listing non-goals helps to focus discussion
and make progress.
-->

- Enforce more stringent checks for _all_ files in a Keylime policy
- Implement specific kinds of additional file/hash verification

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation.  The "Design Details" section below is for the real
nitty-gritty.
-->

Keylime's IMA policy format should contain an additional field, which would contain a JSON-formatted dictionary of key-value pairs (with digests as keys) that could later be used by Keylime to implement additional checks on the verifier. These additional values could range in type from boolean all the way to base64-encoded public keys or signatures if needed.

### User Stories (optional)

<!--
Detail the things that people will be able to do if this enhancement is implemented.
Include as much detail as possible so that people can understand the "how" of
the system.  The goal here is to make this feel real for users without getting
bogged down.
-->

#### Story 1

I am a system administrator looking to attest the validity and provenance of a mission-critical application running on a Fedora machine. I already have a vendor-provided policy for Fedora, which attests the validity of the base image, but I'd like to be able to run additional verification checks within Keylime for the application I'm running on top of Fedora. Most of all, I'd like to be able to validate that the binaries running on my agent machine come from a specific developer identity. In order to do this, I attach additional metadata to my Keylime IMA policy, specifying that the binary for my mission-critical application must have

- an inclusion proof within a transparency log
- a valid signature within that transparency log that attests to a specific developer's identity

in order to run on my agent-attested machine. Some future version of Keylime would consume this metadata and carry out the required checks.

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

An example Keylime policy (after the implementation of the IMA policy enhancement proposal, [here](https://github.com/keylime/enhancements/pull/71)), would look something like this:

```json
{
    "signatures": [
        {
          "keyid": <KEYID>,
          "keytype" : <KEYTYPE>,
          "sig": <SIGNATURE>
        }
    ],
    "signed": {
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
}
```

With this enhancement, an additional field called `digest_metadata` would be present in the `signed` body, alongside `digests` and `excludes` etc:

```json
{
    "signatures": [
        {
          "keyid": <KEYID>,
          "keytype" : <KEYTYPE>,
          "sig": <SIGNATURE>
        }
    ],
    "signed": {
        "meta": {
            "version": 0
        },
        "release": 6.5.0,
        "digests": {
            "/root/hello.txt": ["a4dc309f..."]
        },
        "excludes": [],
        "digest_metadata": {},
        "keyrings": {},
        "ima": {
            "ignored_keyrings": [],
            "log_hash_alg": "xxx"
        },
        "ima-buf": {},
        "verification-keys": []
    }
}
```

This could then be populated with key-value pairs of hashes and metadata about them. For example, to specify that the Keylime verifier should perform an inclusion proof against hash `a4dc309f...`:

```json
{
    "signatures": [
        {
          "keyid": <KEYID>,
          "keytype" : <KEYTYPE>,
          "sig": <SIGNATURE>
        }
    ],
    "signed": {
        "meta": {
            "version": 0
        },
        "release": 6.5.0,
        "digests": {
            "/root/hello.txt": ["a4dc309f..."]
        },
        "excludes": [],
        "digest_metadata": {
            "a4dc309f...": {
                "transparency_log_inclusion_proof": true,
            },
        },
        "keyrings": {},
        "ima": {
            "ignored_keyrings": [],
            "log_hash_alg": "xxx"
        },
        "ima-buf": {},
        "verification-keys": []
    }
}
```

These values could vary in complexity according to the checks being implemented.


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

Tests to this extension would be dependent on individual implementations of specific checks. For example, if a check for transparency log inclusion was implemented, it would be up to that implementation to also test for correctly-formatted metadata.

### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this enhancement
-->

Since this is a new field within Keylime's IMA policy schema, upgrades should be trivial, and IMA policies of this new format should still be backwards compatible with older Keylime components.

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

N/A

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

Another possibility might be to refactor the way the `digests` dictionary is formatted, to allow for metadata to live right alongside allow/exclude data. This might be harder to parse, and would be less forward/backwards compatible.

## Infrastructure Needed (optional)

<!--
Use this section if you need things infrastructure related specific to your enhancement.  Examples include a
new subproject, repos requested, github webhook, changes to CI (travis).
-->

N/A
