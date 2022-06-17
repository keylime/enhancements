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
# enhancement-53: Agent-local Revocation Notification Mechanism

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

- [x] Enhancement issue in release milestone, which links to pull request in [keylime/enhancements]
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

Keylime agent currently receives revocation notifications through
ZeroMQ over TCP.  The agent connects to the revocation notifier
service, typically run by the verifier, and waits for any revocation
being sent.

While this setup works well in practice, it is not ideal in certain
deployment scenarios such as IoT.  This enhancement provides an
alternative revocation notification mechanism and the configuration
options.

## Motivation

<!--
This section is for explicitly listing the motivation, goals and non-goals of
this enhancement.  Describe why the change is important and the benefits to users.
-->

### Dependencies of ZeroMQ implementations

The reference implemention of ZeroMQ ([libzmq]) has quite a few
run-time dependencies including libsodium (for cryptography) and
openpgm (for multicast), which are not always available in restricted
environment.

While revocation notification can be turned off at run-time, this is
particularly problematic for Rust agent, because whether the ZeroMQ
feature is supported needs to be determined at build time.  Although
there is a pure-Rust rewrite of ZeroMQ ([zmq.rs]) which can be
configured without such dependencies, it depends on a newer async
runtime (tokio 1.x) incompatible with the one used by the Rust agent.

### Goals

<!--
List the specific goals of the enhancement.  What is it trying to achieve?  How will we
know that this has succeeded?
-->
 * Add support for the agent to listen on revocation notification without ZeroMQ transport
 * Add support for the verifier to send out revocation notification without ZeroMQ transport

### Non-Goals

<!--
What is out of scope for this enhancement?  Listing non-goals helps to focus discussion
and make progress.
-->
 * Secure execution of revocation actions by the agent (while it is desired)
 * Moving Keylime to a push-only model

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation.  The "Design Details" section below is for the real
nitty-gritty.
-->

This enhancement proposal adds a new revocation notification mechanism
natively integrated into the agent protocol, as well as the supporting
configuration options to allow the verifier to use multiple revocation
notification mechanisms together.

### User Stories (optional)

<!--
Detail the things that people will be able to do if this enhancement is implemented.
Include as much detail as possible so that people can understand the "how" of
the system.  The goal here is to make this feel real for users without getting
bogged down.
-->

#### Story 1
 * Verifier has `revocation_notifiers` with the value "agent" configured
 * Agent registers itself with the registrar
 * User adds agent with `keylime_tenant -c add -u AGENT_ID`
 * Agent is added to the verifier
 * Verifier sends revocation notification to the agent through the REST API
 * Agent executes revocation actions

### Notes/Constraints/Caveats (optional)

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above.
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->
 * The API version needs to be bumped to a new minor version, as this change falls under the "Adding a new API endpoint would be a minor version bump as previous clients would not be using it" [criteria](https://github.com/keylime/enhancements/blob/master/45_api_versioning.md#design-details).

### Risks and Mitigations

<!--
What are the risks of this proposal and how do we mitigate.  Think broadly.
For example, consider both security and how this will impact the larger
enhancement ecosystem.

How will security be reviewed and by whom?
-->

The attack surface on the agent protocol slightly increases as the new
REST resource (`/v1.0/notifications/revocation`) is added, though the
messages are digitally signed and cannot be exploited as long as the
agent is properly implemented/configured.

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable.  This may include API specs (though not always
required) or even code snippets.  If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->

The REST API of the agent will export a new resource at
`/v1.0/notifications/revocation`, which can be accessed with a POST
method.  The request body is a JSON object with the following properties:

- `msg` (_string_) - argument passed to the revocation actions
- `signature` (_string_) - signature calculated over `msg`, using RSA-PSS with SHA-256 as the hash algorithm and the maximum salt length for the RSA key, in base64 format

At the agent side, the method and the invocation of the revocation
actions should be implemented as idempotent, so that it should not
matter how many times the method is called.

In the `cloud_verifier` section of the configuration file, a new
option `revocation_notifiers` is added to select notification
mechanisms that the verifier uses.  The option takes a list of strings
and the possible values are `zeromq`, `webhook`, and `agent`.

The former two have the same effect as the current
`revocation_notifier` and `revocation_notifier_webhook`, while the
latter (`agent`) enables push notification to the agent, based on the
REST API described above.

The `revocation_notifier` and `revocation_notifier_webhook` options
are deprecated and mutually exclusive with `revocation_notifiers`. The
new code supporting this enhancement should put a warning in the log
that the options are deprecated and suggest the required changes.

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

 * Extend the `test_restful.py` tests to check for the new protocol.

### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this enhancement
-->
To upgrade/downgrade, the configuration file needs modification if it
makes use of the new configuration option (`revocation_notifiers`). A
helper script could be provided to ease the migration

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
No additional dependencies should be required.

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
 * It is possible to use other asynchronous messaging technology or
   design custom protocol without imposing dependencies.  However, the
   agent is already listening on a REST endpoint, reusing the agent
   protocol would be straightforward.

## Infrastructure Needed (optional)

<!--
Use this section if you need things infrastructure related specific to your enhancement.  Examples include a
new subproject, repos requested, github webhook, changes to CI (travis).
-->
No infrastructure changes needed.

[libzmq]: https://github.com/zeromq/libzmq/
[zmq.rs]: https://github.com/zeromq/zmq.rs/
