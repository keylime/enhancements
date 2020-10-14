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
# enhancement-23: Use a dedicated Keylime user account

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

Currently, Keylime runs as the root user, providing it essentially all control
over a host on which Keylime runs. This enhancement suggests changing this to
running Keylime (specifically, its different components) under a dedicated user
account to limit the level of damage that could happen should a vulnerability in
Keylime be exploited.

We will analyse Keylime Verifier, Keylime Registrar and Keylime Agent to decide
which do not need root access to run properly, and then make necessary changes
to run them under their own dedicated user, `keylime`.

## Motivation

Running everything as root is bad practice: any piece of software may have a
vulnerability, and if a bug in a piece of software running as root is exploited,
the negative consequences to the host system can be dramatic, essentially
allowing the malicious party exploiting the bug to have full control over the
system (There are some exceptions to this, which we will not consider
specifically here as they are system-dependent, such as running software in
TEEs)).

A simple and well-known defense against this is privilege separation,
specifically the least privileged model, where each piece of software is given
the rights necessary to run properly, but no more. As such, we want to
make sure that, in as many cases as possible, Keylime components do not run as
the root user but as a specific user, whose rights on a host are more
specifically scoped out, minimising the risk of damage should an exploitable
vulnerability be found in Keylime.

### Goals

The goals of the enhancement is to move as many components of Keylime as
possible from running as the root user to running as a dedicated user, called
`keylime`.

### Non-Goals

This enhancement does not aim to solve the problem of Keylime dependencies
running as root, such as tpm2-tools.

## Proposal

In practice, this enhancement will happen in stages.

First, evaluate the different Keylime components to see which ones are being run
as the root user by convenience, and which ones may genuinely need to do so to
be able to operate properly.

Once this is done, we will look at the best option to run each of these
components under the Keylime user. This may be as simple as modifying the
systemd unit file to specify a user under which to run Keylime.

Other files that will need to be modified to take this change into account are
`installer.sh`, `run_tests.sh` as well as the Ansible role for Keylime. There
may be some work to do on the SElinux labels too. 

Finally, the documentation will have to be updated as well.

### User Stories (optional)

#### Story 1

As a system administrator in an organisation with strong security rules
regarding the software that may be run, and how, i would like to use Keylime for
the benefits it provides, but cannot as only a very limited subset of software
from a pre-approved list is allowed to run as root on the organisation's hosts,
and Keylime is not one of them.

Now that Keylime uses its own dedicated user account, this aligns Keylime with my
organisation's policy and i can deploy Keylime.


### Notes/Constraints/Caveats (optional)

TBD
<!--
What are the caveats to the proposal?
What are some important details that didn't come across above.
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->

### Risks and Mitigations

TBD
<!--
What are the risks of this proposal and how do we mitigate.  Think broadly.
For example, consider both security and how this will impact the larger
enhancement ecosystem.

How will security be reviewed and by whom?
-->

## Design Details

TBD
<!--
This section should contain enough information that the specifics of your
change are understandable.  This may include API specs (though not always
required) or even code snippets.  If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->

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

It makes the overall design of Keylime a bit more complicated. However, we
believe that benefits provided by improving privilege separation vastly
outweigh this drawback.

## Alternatives

Privilege separation (and more generally, defense in depth) can be done at
various levels, however at the level we are discussing here (users on a
UNIX-based system) there are no alternatives to my knowledge.  
Mandatory Access Control (MAC) systems, such as SElinux, are complementary 
rather than alternatives to user privilege separation.

## Infrastructure Needed (optional)

N/A
