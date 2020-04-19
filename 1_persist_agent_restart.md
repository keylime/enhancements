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
# Persist verifier monitoring after agent restarts

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

Should someone restart an agent based server or force an agent offline, the
agent will no longer be monitored by the verifier. Upon starting the agent will
just register with the registrar and IMA monitoring will cease.

This behavior was originally discussed on the [keylime mailing list](https://keylime.groups.io/g/main/topic/q_is_an_agent_s_policy/72856684?p=,,,20,0,0,0::recentpostdate%2Fsticky,,,20,2,0,72856684)

## Motivation

<!--
This section is for explicitly listing the motivation, goals and non-goals of
this enhancement.  Describe why the change is important and the benefits to users.
-->

Its acceptable that someone may want to manually restart a server (or the server
restarts as part of an automated work flow) while retaining the configuration
set up during the adding of the agent to the verifier (`whitelist`, `tpm_policy`)
without needing to remote and add (or update) the verifier ever time.

### Goals

<!--
List the specific goals of the enhancement.  What is it trying to achieve?  How will we
know that this has succeeded?
-->

A user restarts the agent on a target node and when the agent is active again
the verifier proceeds to commence monitoring of delegated measurements again.

### Non-Goals

<!--
What is out of scope for this enhancement?  Listing non-goals helps to focus discussion
and make progress.
-->

Any sort of migration or fault redundancy (although both areas benefit from this
change )

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation.  The "Design Details" section below is for the real
nitty-gritty.
-->

A target machine is rebooted with no change in state. This machine should not
require “re adding” with the keylime tenant again. Once it comes back online,
the verifier should proceed to recommence run time monitoring.

A new tornado web handler is created within the verifier to listen for requests
that an agent will emit when it (re)starts.

Code will be introduced to the agent that will perform a `POST` request to
inform the verifier an agent has been (re)started. This in turn will cause the
verifier to perform an `operational_state query` for the UUID of that agent and
then proceed to perform run time integrity monitoring again.

### User Stories (optional)

<!--
Detail the things that people will be able to do if this enhancement is implemented.
Include as much detail as possible so that people can understand the "how" of
the system.  The goal here is to make this feel real for users without getting
bogged down.
-->

For any given reason my server reboots. Keylime handles this event and provides
trust assurity once the server and agent are back online.

Should the machines state have been tampered with during the offline period,
Keylime will immediate fail the target node accordingly (or likewise show the
machine is still in the expected trust state according to the delegated
measurements)

### Risks and Mitigations

<!--
What are the risks of this proposal and how do we mitigate.  Think broadly.
For example, consider both security and how this will impact the larger
enhancement ecosystem.

How will security be reviewed and by whom?
-->

We should be sure we do not introduce securty risks and be mindful of future
enhancements such as multi tenancy, auth and migration.

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable.  This may include API specs (though not always
required) or even code snippets.  If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->

Verifier Changes
----------------

A new tornado web handler is created within the verifier to listen for requests
that an agent will emit when it starts. We will call this `/nudge` for now with
a more suitable now agreed within this review.

A new `operational_state` named `OFFLINE` will be created for when a machine
becomes unreachable during a `GET_QUOTE` `operational_state`

Agent Changes
-------------

Code will be introduced to the agent that will perform a `POST` request
`/nudge` to inform the verifier an agent has been (re)started. This in turn will
instruct the verifier to perform an `operational_state` query for the `UUID` of
the concerned agent, should the `operational_state` be `OFFLINE`, it will
change `operational_state` to `GET_QUOTE` and proceed to (re)start continuous
monitoring of the node with the previous set measurements (`whitelist`,`tpm_policy`)

Registrar Changes
------------------

No immediate changes come to mind, but we should be mindful of this as the
design evolves.

Keylime TPM coms changes
------------------------

We will need to assess changes required within our tpm communications. For
example the Agent calls `tpm_startup -c` and takes ownership of the tpm
every time it starts. The AK handle is also flushed.

We may need to consider having some sort of flag the agent queries to establish
its already associated with a verifier.

Rather than bootstrapping itself as a fresh agent, it instead retains its TPM
set up and instead just instantiates its web service to allow rest API
interactions with the verifier. These interactions will be a continuum of the
previous quote `GET` requests from the verifier while retaining the root of
trust already set up by the registrar.

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

Functional tests will be needed to play out the user case of restarting a
agent, persisting state and reestablishing measurements upon its restart.

Unit tests will be needed to test the new `nudge` API functionality.

### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this enhancement
-->

All changes will be code based. May need to consider impact of upgrading with
an agent offline and then the new TPM code changes interacting with the TPM
setup from the previous release.

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

We evolve the retry handler in the verifier to wait for indefinite periods
instead of having a wake up API - this is hazardous as we risk bottle necks
and need to consider managing more state (for example a node goes offline to
never return).

## Infrastructure Needed (optional)

<!--
Use this section if you need things infrastructure related specific to your enhancement.  Examples include a
new subproject, repos requested, github webhook, changes to CI (travis).
-->

Some changes may be needed to travis CI, but not expected currently.

No new repos required.
