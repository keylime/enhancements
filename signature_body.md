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
# 77-signature_body: Attached signatures

Utilise signature body in allow list for attached signatures. 

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
- [ ] Core members have approved the issue with the label `in-progress`
- [ ] Design details are appropriately documented
- [ ] Test plan is in place
- [ ] User-facing documentation has been created in [keylime/keylime-docs]

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

## Summary

This proposal introduces a method to sign over the body of an allow-list
and then attach a signature and public key ID or X509 certificate.

It uses the same manifest layout as supply chain projects [intoto](https://in-toto.io/)
and [The update framework](https://theupdateframework.io/).
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

## Motivation

The current implementation leverages seperate keys to the list itself. While
suitable for some cases, it is also useful to have a single package where the
list and the signature travel in the same payload.

<!--
This section is for explicitly listing the motivation, goals and non-goals of
this enhancement.  Describe why the change is important and the benefits to users.
-->

### Proposal

User produces a cryptographic signature by 'passing over' the body of an allow/exclude
list. The signature and public key id (sha256 hash of pubKey), or an X509 cert (base64)
are then attached to the allowlist.

The verifier upon discovery of a `"signed"` body and `"signatures"` keys, will validate
the `signed` section.

This proposal borrows from the existing signing conventions
established by the [intoto project](https://github.com/in-toto/docs/blob/master/in-toto-spec.md#4-document-formats).

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
        "version": 872
        },
        "release": 774.75,
        "hashes": {
        ",'Sr": []
        },
        "excludes": [],
        "keyrings": {
        "}": "xxx"
        },
        "ima": {
        "ignored_keyrings": [],
        "log_hash_alg": "xxx"
        },
        "ima-buf": {
        "": "xxx"
        },
        "verification-keys": []
    }
}
```
#### signatures

The `signatures`` section lists there json keys:

##### keyid

* keyid: The keyid can be one of two. 
1. A sha256 hash of a public key (therefore acting as a map). This allows an out-of-band public key
2. A base64 encoded X509 v3 certificate in PEM format. This is in-band and can be chained to CA for identity.

For the first method (sha256 hash) the public key will be stored within the verifier at:

`/var/lib/keylime/signing_keys/{hash-of-pubkey}.pkey`

For the second method a store is not required, the public key can be extracted from the X509 certificate

For a later stage, we can look to have a chain verification from the the signing cert to a CA.

##### keytype

keytype is a string denoting a public key signature system.

We define three key types at present: "rsa", "ed25519", and "ecdsa".

##### sig

The `sig` section is a base64 encoded signature (see signed below for details around signature generation)

#### signed

The signed body contains all content that is to be signed.

The signature functionality, extracts all characters (including whitespace) and encodes it to a base64 string

A signature is then generated by signing the base64 string. This is then placed into `sig` section within the
`signatures` body.

### verification

The verification flow, will play out as follows

1. The verifier will check to see if the signature and signed bodies exist
2. The verifier will either decode the x509 certificate and extract the public key, or will capture
   the sha256 digest and load the mapped key from `/var/lib/keylime/signing_keys/{hash-of-pubkey}.pkey`
3. The verifier will then perform a verification of the signature against the `signed` body
4. If any of the steps above fail, then a verifier API will return a failure code.

### Non-Goals

<!--
What is out of scope for this enhancement?  Listing non-goals helps to focus discussion
and make progress.
-->
No immediate plan to implement signing into the tenant and no extra flags are required,
the verifier will check for the signature and signed body and only attempt to validate
if they exist.

### User Stories (optional)

<!--
Detail the things that people will be able to do if this enhancement is implemented.
Include as much detail as possible so that people can understand the "how" of
the system.  The goal here is to make this feel real for users without getting
bogged down.
-->

I as a user would like to have attached signatures.

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

Unit test will be provided to verify serialization of json input and 
signature verification.

### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this enhancement
-->

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

No new dependencies will be required. 

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
