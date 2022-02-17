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
# enhancement-55: Revocation Actions without Python Runtime

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

This enhancement proposal allows revocation actions to be any
executable or script, not only a Python module.

## Motivation

<!--
This section is for explicitly listing the motivation, goals and non-goals of
this enhancement.  Describe why the change is important and the benefits to users.
-->

Currently Keylime requires that agent-local actions executed at
revocation notification to be written in Python with a specific
convention: an async `execute` function is globally defined and it
takes a revocation message as the argument.

With the upcoming addtion of the new Rust-based keylime agent there
will need to be a means to execute actions without Python runtime
installation.

### Goals

<!--
List the specific goals of the enhancement.  What is it trying to achieve?  How will we
know that this has succeeded?
-->

 * Arbitrary scripts/executable can be used as a revocation action

### Non-Goals

<!--
What is out of scope for this enhancement?  Listing non-goals helps to focus discussion
and make progress.
-->

 * Secure execution of revocation actions (while it is desired), such as using sandbox and seccomp

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation.  The "Design Details" section below is for the real
nitty-gritty.
-->

The actions defined in the configuration file (or tenant-provided
action list) can be any type of script/executable. The actions names are no
longer required to begin with `local_action_`.

When Keylime agent receives a revocation message, it stores the JSON
payload in a file and invokes revocation actions with the file as a
command-line argument.

### User Stories (optional)

<!--
Detail the things that people will be able to do if this enhancement is implemented.
Include as much detail as possible so that people can understand the "how" of
the system.  The goal here is to make this feel real for users without getting
bogged down.
-->

#### Story 1

* The agent is configured with revocation notification enabled
* The agent is also configured with non-Python actions (pre-installed on the system) set up in the `revocations_actions` configuration option
* The verifier notifies a revocation
* The agent will invoke the actions with the revocation message stored in a file
* If any Python actions are installed alongside non-Python actions, the agent will search and invoke them in the same [convention][secure payload] as used before, after non-Python actions are invoked

#### Story 2

* The agent is configured with revocation notification enabled
* The tenant sends non-Python actions as part of the initial payload, as well as the `action_list` file listing those actions
* The verifier notifies a revocation
* The agent will invoke the actions with the revocation message stored in a file
* If any Python actions are installed alongside non-Python actions, the agent will search and invoke them in the same [convention][secure payload] as used before, after non-Python actions are invoked

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

This mechanism makes it a little easier for the attacker to execute
commands on the system, though the same thing is already possible by
injecting custom Python actions.  To keep the risk minimal, this
proposal suggest mandating that the pre-installed actions are
installed in a fixed/immutable directory, such as `/usr` on modern
Linux distributions rather than `/var/lib` or similar. That way the
action commands will also be subject to attestation.

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable.  This may include API specs (though not always
required) or even code snippets.  If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->

The actions defined with the `revocations_actions` configuration
option (or in the `unzipped/action_list` file provided by the tenant)
can be of the command names of scripts/executables. The names are no longer
required to begin with `local_action_`.

There are two possibilities where the actual action can be found:
pre-installed on the system or sent by the tenant as part of the
initial encrypted payload ([secure payload]).

The following couple of new options are added to the `[cloud_agent]`
section of the configuration file:

- `revocation_actions_dir` (_string_): The location where
  pre-installed actions are found. It is suggested that this points to
  a fixed/immutable location subject to attestation. The default is
  `/usr/libexec/keylime`. See [Risks and
  Mitigations](#risks-and-Mitigations) for details.
- `allow_payload_revocation_actions` (_boolean_): Whether to invoke
  revocation actions sent as part of payload. The default is `True`
  and turning it off allows the agent to limit actions to only
  pre-installed ones for more security.

For backward compatibility, if there is no corresponding
script/executable found on the system nor in the secure payload, the
agent may fall back to the Python-based actions.

The actual lookup procedure of the actions is as follows:
1. Look for the named command on the system
1. Look for the named command in the tenant-provided initial payload
1. If the agent supports Python actions, look for the Python module on the system
1. If the agent supports Python actions, look for the Python module in the tenant-provided initial payload

The command takes a command-line argument: the absolute path to the
file where the revocation message is stored in JSON.  Before being
invoked, the process' working directory will be changed to the secure
mount directory (`/var/lib/keylime/secure`) where any initial payload
is extracted and stored.

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

 * A new unit test should be added to exercise action lookup
 * A new integration test should be written to exercise revocation actions
 
### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this enhancement
-->

When downgrading, non-Python actions will stop working.  Proper log
messages would help diagnose the issues.

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

No dependencies are known of.

## Drawbacks

<!--
Why should this enhancement _not_ be implemented?
-->

No drawbacks are known of.

## Alternatives

<!--
What other approaches did you consider and why did you rule them out?  These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->

 * It would also be an option to make use of an embeddable language runtime, such as Lua and mruby.

## Infrastructure Needed (optional)

<!--
Use this section if you need things infrastructure related specific to your enhancement.  Examples include a
new subproject, repos requested, github webhook, changes to CI (travis).
-->

No infrastructure change needed.

[secure payload]: https://keylime-docs.readthedocs.io/en/latest/user_guide/secure_payload.html
