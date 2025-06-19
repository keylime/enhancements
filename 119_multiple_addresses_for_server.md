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
# Enhancement-119: Multiple Addresses for Server

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
- [Enhancement-119: Multiple Addresses for Server](#enhancement-119-multiple-addresses-for-server)
  - [Release Signoff Checklist](#release-signoff-checklist)
  - [Summary](#summary)
  - [Motivation](#motivation)
    - [Goals](#goals)
      - [Availability:](#availability)
      - [Security:](#security)
    - [Non-Goals](#non-goals)
  - [Proposal](#proposal)
    - [User Stories (optional)](#user-stories-optional)
      - [Story 1](#story-1)
      - [Story 2](#story-2)
    - [Notes/Constraints/Caveats (optional)](#notesconstraintscaveats-optional)
    - [Risks and Mitigations](#risks-and-mitigations)
  - [Design Details](#design-details)
        - [1. Pull mode, 1 group, 1 server per group:](#1-pull-mode-1-group-1-server-per-group)
        - [2. Push mode, 1 group, 1 server per group:](#2-push-mode-1-group-1-server-per-group)
          - [3. Push mode, 1 group, multiple servers per group:](#3-push-mode-1-group-multiple-servers-per-group)
          - [4. Push mode, multiple groups, multiple servers per group:](#4-push-mode-multiple-groups-multiple-servers-per-group)
    - [Test Plan](#test-plan)
    - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
    - [Dependencie requirements](#dependencie-requirements)
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

This will add support for allowing users to specify multiple Registrar addresses in the pull model, and multiple Registrar and Verifier addresses in the push model. When an agent communicates with the Registrar, it will randomly select a Registrar address from the list provided by the user. If the connection fails, it will reselect until a successful connection is made. In the push model, where the agent must initiate the verification, the agent will use the same process for choosing a verifier.

This will be an iterative set of enhancements, split into 3 changes:
1. Enable a list of Registrars and Verifiers to be given in the config, with the agent randomly selecting from the Registrars, and in the push model the Verifiers as well, and using that address, re-selecting after connection failures.
2. Add a config option to specify the number of Registrars and Verifiers to communicate with.
3. Allow Registrars and Verifiers to be specified in a list form with each list item being a block of Registrar and Verifier addresses that share a backend, and require a number or Registrar/Verifier blocks to attest to.

## Motivation

<!--
This section is for explicitly listing the motivation, goals and non-goals of
this enhancement.  Describe why the change is important and the benefits to users.
-->

### Goals

<!--
List the specific goals of the enhancement.  What is it trying to achieve?  How will we
know that this has succeeded?
-->
There are two main goals for this enhancement.
#### Availability: 
By allowing lists of Registrars and Verifiers we can increase availiability by enabling the agent to fallback to alternative addresses in the case that a Registrar or Verifier cannot be contacted.

#### Security:
By adding the option of requiring a certain amount of Registrar/Verifier blocks to attest to, and enabling attestation to platforms with separate backends we remove the single point of failure that is currently present and so increase security.

### Non-Goals

<!--
What is out of scope for this enhancement?  Listing non-goals helps to focus discussion
and make progress.
-->

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
---WIP---

The agent config will need to be adapted. 
A new config option to replace `registrar_ip` and `registrar_port` will be added, called `servers` (?). Each address till be specified as `"[scheme://]host[:port]"` with scheme and port optional. If just host is given, eg. `n.n.n.n` then default Keylime ports and protocols will be used, if host and port is provided, eg. `n.n.n.n:pppp` then just the default protocol will be used.

This will accept the following configurations:
##### 1. Pull mode, 1 group, 1 server per group:

        [[servers]]
        registrars = [ "reg1.example.com" ]

##### 2. Push mode, 1 group, 1 server per group:

        [[servers]]
        registrars = [ "reg1.example.com" ]
        verifiers = [ "ver1.example.com" ]

###### 3. Push mode, 1 group, multiple servers per group:

        [[servers]]
        registrars = [ "reg1.example.com", "reg2.example.com" ]
        verifiers = [ "ver1.example.com", "ver2.example.com" ]

###### 4. Push mode, multiple groups, multiple servers per group:

        [[servers]]
        registrars = [ "reg1.example.com", "reg2.example.com" ]
        verifiers = [ "ver1.example.com", "ver2.example.com" ]

        [[servers]]
        registrars = [ "reg3.example.com", "reg4.example.com" ]
        verifiers = [ "ver3.example.com", "ver4.example.com" ]

        [[servers]]
        registrars = [ "reg6.example.com" ]
        verifiers = [ "ver6.example.com" ]

If Pull mode is being used but verifier addresses are provided, a config warning will be emitted noting that the specified verifier addresses will not be used.

If Push mode is being used and no verifier addresses are given, an error specifying that no verifier address has been provided will be given.

A new config option `verification_duplication = 1`(?) will be added, which will allow a user to specify how many of the blocks to register and verify with. This would require option 4 for `servers` config option so that there are at least 2 backends, and the maximum value would be equal to the number of backends, requiring registration and verification to all backends. The default will be 1, and changing this value will result in a (warning/error) in the situation where only one block is given, or the config is like 1-3.

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
We will need to ensure the "blocks" of addresses are respected and that addresses aren't mixed. We should also ensure that various formats in the config file are compatible, for example allowing protocol to be specified (https://n.n.n.n:pppp), ports to be recognized, and defaults applied if they are missing (n.n.n.n should default to standard Keylime ports and protocol).

### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this enhancement
-->

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
