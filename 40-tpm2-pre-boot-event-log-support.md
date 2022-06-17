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
# enhancement-#40: TPM 2.0 Pre-Boot Event log support

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

Provide Keylime with the ability to extract and process the TPM 2.0 Pre-Boot
Event Log Keylime. Event logs are highly useful in attestation given that these
can be used to reconstruct the values of any TPM PCR. With the proposed enhancement,
Keylime `agents` will to access the contents of the event log (via `securityfs`), ship it
to a `verifier`, and have it properly processed there. This framework is very similar
to what is already done currently for IMA, with the added benefit of this being
applicable to the "reconstruction" of values of PCRs other than the #10.

## Motivation

<!--
This section is for explicitly listing the motivation, goals and non-goals of
this enhancement.  Describe why the change is important and the benefits to users.
-->

In any modern datacenter, the usual scenario is one with literally hundreds of different types of nodes, 
and simply "getting the contents of the boot aggregate (PCR 10) from a list of 
well-known values" has shown itself to be unfeasible from an operational standpoint 
(just to give some perspective: when nodes are netbooted, the value of every PCR1 
is different as it encodes the MAC address of the NIC). Our experiments with the 
"TPM 2.0 Pre-Boot Event log" demonstrated that, with a "recent enough" Kernel (e.g. 5.4.X), 
and a "recent enough" version of tpm2-tools, we can access the contents of the aforementioned 
log to re-create (or debug) the values of PCRs [0-9],14 without requiring a 
pre-measurement (i.e., a "golden value").

### Goals

<!--
List the specific goals of the enhancement.  What is it trying to achieve?  How will we
know that this has succeeded?
-->

- Add support for the `agents` to (conditionally) read and ship the TPM 2.0 event log
- Add support on the `verifier` to replay the event log and reconstruct the values of PCR [0-9],14
- Add support on the `tenant` CLI to specify an "event log desired state", which can then be 
consumed by the `verifier` to attest a given event log

### Non-Goals

<!--
What is out of scope for this enhancement?  Listing non-goals helps to focus discussion
and make progress.
-->

- The "policy", which will take a given event log, a given "event log desired state" 
and issue a pass/fail on the former based on the latter is outside of the scope of this
enhacement. We will provide a way for a `verifier` to execute a generic policy, but the
actual policy code will be provided by Keylime deployers/users.


## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation.  The "Design Details" section below is for the real
nitty-gritty.
-->

- The `tenant` CLI will provide a new option to indicate an "event log desired state"
- The `agent`, upon being notified that a new "event log desired state" exists,
will read the TPM2.0 pre-boot event log from /sys/kernel/security/tpm0/binary_bios_measurements
and ship it back to the `verifier`
- Whenever the `agent` reads the TPM2.0 pre-boot event log, it will also
read (and ship to the `verifier`) the contents of PCRs [0-9], 14.
- The verifier will be extended to support the replaying of the event log, 
making sure it matches the contents of the aforementioned PCRs.
- The verifier will also be extended to support the execution of generic
python code, in the form of a "policy", which will read the contents of 
the TPM2.0 Pre-Boot Event Log extracted by the agent, the contents of the 
"event log desired state" (supplied via `tenant` CLI and stored on the database) 
and then apply a pass/fail attestation test to the event log.
- Test cases will be written that a very simple "policy", which will simply
read the contents of the event log and attest it as legitimate, will be available
by default. This means that the TPM2.0 Pre-Boot Event Log will be simply read on its
entirety and that is can properly reconstruct PCRs [0-9],14. 

### Notes/Constraints/Caveats (optional)

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above.
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->

- More complex policies will have to provided by the user, since these are
very site-specific. For instance, we have an internal policy to check for 
firmware versions of all hardware devices, an information which is recorded 
on the TPM2.0 Pre-Boot Event Log, and thus we have special custom "desired state"
policy which will not be made available as open-source.

### Risks and Mitigations

<!--
What are the risks of this proposal and how do we mitigate.  Think broadly.
For example, consider both security and how this will impact the larger
enhancement ecosystem.

How will security be reviewed and by whom?
-->

- What is being proposed here is not dissimilar to what is already done with IMA 
in the sense that:
       - it is an optional capability, which should not interfere with the "regular" operation of KL 
       - adds, to each interaction verifier<->agent interaction, no more than tens of KB (which is the total size of the "TPM2 Pre-Boot Event log"). 
- While we do expect very little impact on KeyLime's scalability by adding this capability, it is 
important to remember that we are constantly testing KL in a configuration with 5K nodes (with IMA), and can provide experimental evidence to back this hypothesis.

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable.  This may include API specs (though not always
required) or even code snippets.  If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->

- The "event log desired state" will be provided, via `tenant` CLI, in JSON format, and it is up
to the specific policy code to read, interpret and apply it to the received
TPM2.0 Pre-Boot Event Log

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

- The simplest "always pass" code for the policy would allows us to make sure that TPM2.0 Pre-Boot Event Log
can be extracted by the `agent`, sent to the `verifier` and that the replaying of such log will match the
values on PCRs [0-9],14.

### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this enhancement
-->

- This is an optional feature, and thoroughly backward compatible with current Keylime deployments.

## Drawbacks

<!--
Why should this enhancement _not_ be implemented?
-->

- No known drawbacks.

## Alternatives

<!--
What other approaches did you consider and why did you rule them out?  These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->

- No known alternatives.

## Infrastructure Needed (optional)

<!--
Use this section if you need things infrastructure related specific to your enhancement.  Examples include a
new subproject, repos requested, github webhook, changes to CI (travis).
-->

- A relatively recent linux kernel (5.4.X) is needed to access the contents of the TPM2.0 Pre-Boot Event Log via securityfs
