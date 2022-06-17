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
# enhancement-33: Automate assignees on enhancement proposal PRs

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
Keylime's `enhancements` process uses Github's Pull Requests feature for proposing changes, allowing collaborators to make use of the robust review tools to suggest improvements or facilitate discussion. The process does not, however, make use of Github's assignee or review request features. This presents an opportunity to apply some generalized automation work done for the Enarx project around making PR assignees more useful than a manual field -- instead, automating it to directly indicate who is responsible for pushing a PR towards merge. This work could also eventually apply to other Keylime repos as well.

## Motivation

<!--
This section is for explicitly listing the motivation, goals and non-goals of
this enhancement.  Describe why the change is important and the benefits to users.
-->
Github's PR assignee feature is currently underutilized. At best, it's usually used to designate someone as a point of contact for that PR.

The Enarx project has developed some automation tools that seek to change that, however -- in particular, making a PR's current assignee reflect immediate responsibility for pushing a PR forward. Effectively, whenever something is assigned to someone, that assignment means they, specifically, are needed -- a nice usability benefit for developers and reviewers.


### Goals

<!--
List the specific goals of the enhancement.  What is it trying to achieve?  How will we
know that this has succeeded?
-->
Enable automation from Enarx on `keylime/enhancements`

### Non-Goals

<!--
What is out of scope for this enhancement?  Listing non-goals helps to focus discussion
and make progress.
-->
TBD

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation.  The "Design Details" section below is for the real
nitty-gritty.
-->
The proposed automation was developed and refined for the Enarx project and available through [enarx/bot](https://github.com/enarx/bot), or the Enarxbot. The particular component of interest for this proposal has been split out and made generally available in [actions-automation/pr-workflow](https://github.com/actions-automation/pr-workflow), which contains a standalone Github Action with relevant logic.

Once this action has been included (using a workflow `.yml` present in `.github/workflows`) and other requisite setup steps taken, the bot should:

- Automatically request a specified number of reviewers on new pull requests -- in addition to code owners (requested by Github), the bot will request a suggested reviewer, in addition to however many people are needed to hit a user-input target number of reviewers from a user-defined Github team;
- Automatically change PR assignees based on the current state of the PR and its reviews.

Here's an example of a typical timeline of events on a PR:
- Alice opens a PR. Bob, Charlie, and Dora are requested for review.
- Bob, Charlie, and Dora are assigned to the PR, and Alice is not (since she is still awaiting a review, she can't do anything else to push it forward).
- Bob submits a review requesting changes. Bob is unassigned from the PR, and Alice is reassigned, since she must address Bob's review.
- Alice submits a patch to the PR addressing Bob's changes, and requests a re-review. Alice is once again unassigned from the PR, while Bob is reassigned.
- Bob thinks the changes look good, and approves the PR. Bob is unassigned, since he's approved it.
- Alice pushes another patch, dismissing Bob's approving review. Bob is assigned again to approve the updated PR.

This cycle repeats until a satisfactory number of approvals has been reached and there are no outstanding Changes Requested reviews.

This cycle is excellent for indicating responsibility, and, on the Enarx project, has been positively received.

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
TBD

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable.  This may include API specs (though not always
required) or even code snippets.  If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->
TBD

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

### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this enhancement
-->
Removing the appropriate Actions workflow should disable automation cleanly.

## Drawbacks

<!--
Why should this enhancement _not_ be implemented?
-->
TBD

## Alternatives

<!--
What other approaches did you consider and why did you rule them out?  These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->
No direct alternatives.

## Infrastructure Needed (optional)

<!--
Use this section if you need things infrastructure related specific to your enhancement.  Examples include a
new subproject, repos requested, github webhook, changes to CI (travis).
-->
In order to fully do its magic, the bot needs several things:

- A separate user account, scoped to have write access to the Keylime organization (in this case, write access to repos it interacts with should be enough)
- a personal access token with the `public_repo` scope stored as a [shared organization secret](https://docs.github.com/en/free-pro-team@latest/actions/reference/encrypted-secrets), named `BOT_TOKEN`.
- a Github Team in the parent organization that contains people that may be requested to review pull requests
- (optional) a set of code owners specified in a `CODEOWNERS` file
