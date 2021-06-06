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
# enhancement-46: Context and severity levels for revocations

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
This enhancement proposal adds the capability differentiate and classify
revocation events by adding severity levels and user defined or generated
context.

## Motivation
Currently Keylime operates in a binary state. Either a device verified or not.

Not all revocation events have to be handled with an equal severity. 
Depending on the use case a failing PCR caused by a firmware update is handled 
differently than IMA policy failures.

Granular reporting of revocation events allows the user to make more informed 
decisions if failure occurs. The additional provided context also helps with 
reconstructing why a revocation was issued.

This enhancement allows users to use Keylime in new scenarios like exam
environments where a person must make an informed decision if a device is
actually compromised.

### Goals
 * Allow the user to specify a severity level and context information for every
   revocation event that is issued by Keylime.
  * Tag all parts that cause revocations in Keylime with an unique id.
  * Assign context provided by the user to (static) revocation events.
  * Provide the option for Keylime to add context itself to revocation events.
    This is useful if for example Eventlog parsing gains support for dynamic
    policies.
  * Extend the verifier API and tenant to add those rules to Keylime.
 * Provide API entry point for checking which revocations events were triggered
   for a agent.

### Non-Goals
 * The classification of revocation events is highly environment and
   configuration specific. It is not a goal to provide a default configuration
   other than all events are classified with the highest severity.


## Proposal
Currently revocation events are either send if something with the communication
the the agent went wrong, the quote is not valid or one of the PCR based checks
fail.

This proposal changes that behavior and introduces the concept of severity
levels of events and adds the option to provide more context for events.

All parts that cause revocation events are tagged with an unique id and against
those ids user scan specify a severity level and provide some context information.

Instead of stop polling an agent if a failure occurred the agent is added back
for checking if the failure is recoverable. If it is not a revocation event with
the highest severity is generated.

All the send events are visible in the verifier API for other systems to use.

Keylime also gains the ability to specify context for a revocation event.

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
The option to classify the revocation events opens the possibility that events
are not handled. To mitigate this all revocation events that are not explicitly
from the user classified are assigned the highest severity level.

The same concept applies for revocation events that are caused failures that
make the agent irrecoverable. Those also always need to be treated with the
highest severity by default because otherwise some events wont be caught is a
irrecoverable failure is triggered before.

## Design Details
We keep the current model of the states, but modify the behavior of the failure
states. If we are in a failure state that is not irrecoverable the polling of
the agent is stopped otherwise the agent is still added back for normal polling.

A failure object is introduced. Which holds the all the revocation events that
are being produced by the checks and if this any of the failures makes the agent
not recoverable. Irrecoverable events currently are retry timeouts and quote
validation.

All parts of the validation process which is mainly (`check_qoute` and
`check_pcrs`) will append all events to that object instead of returning false.
If the validation process advance without a step that failed the failure object
must be marked as irrecoverable and only then the function can return without
validating further. The state of the agent after that should be
`QUOTE_FAILED_IRRECOVERABLE`.

The highest severity is 0 and the higher the number the number the lower the
severity. This makes it easy to add less sever levels without the need to
increase the level of the highest severity.

A new table `revocation_events` is introduced to save the already send
notifications. It contains the columns:
  * `agent_id`: The id of agent where the event was created.
  * `event_id`: The event ID string
  * `severity_level`: The severity of the event.
  * `context`: String or JSON object that contains more information about that event.

In the `process_agent` function gets a new argument that can contain a failure
object. If the status is `QUOTE_FAILED` the events from failure object are
checked against the database and if there are occurring the first time a message
is send and they are added to the database. Otherwise the event is ignored.

In the verifier API for the agent a new field is exposed that contains the list
of the revocation events, with their respected level and context.

To the send revocation message two new field are added: `severity_level` and
`context`. Such that the other agents can act accordingly.

(To reduce bandwidth all the generated messages could also be bundled into one,
but that would introduce more complexity of parsing the messages for the
recipient.)

If a agent gets removed from the verifier also all entries in
`revocation_events` for the agent will be removed. (An option to clear the old
events from a agent could be introduced, but a reset can already implemented by
removing and then adding the agent to the verifier)

### Events
Events are tagged with an event id. Which has the following schema
`component.[sub_component].event`.

* `component`: Name of the part of Keylime where the event was generated.
  * Current would be: `quote_validation`, `pcr_validation`, `measured_boot`,
    `ima` and `internal`.
* `sub_component`: If it is useful the separate a component into other sub
  components. One example would be IMA checks.
* `event`: The actual event itself.
An example event id for a failing static PCR check would be `pcr_validation.pcr10`.

In most cases the event ids static but for dynamic policies the `sub_component`
and `event` can be generated automatically.

The motivation behind that schema is to allow Keylime to adjust granularity of
the events where needed and give the user an easy way to specify a severity
level to one whole subset of Keylime.

The severity level of an event is either 0 or if it matches one of the user
rules the according to the configuration.

Events can a specified context. Those can be either a static string or an JSON
object that contains more information event. This is optional and it is assumed
that two events from the same agent with the same event id have the same
severity and are the same event.

### User rules
The user can specify severity and context for event ids. 
One rule contains the following attributes:
 * `event_id`: The event id to match or a regex rule for it.
 * `severity_level`: The severity level that the matched events have.
 * `context`: static string for the context of that event.

To make the rules future proof with dynamic policies it is possible to specify a
regular expression for that matching. This introduces more complexity on the
parsing and matching side, but allows for flexible rules. Rules are parsed in a
top down order and the first matching rule is used and are supplied as a JSON
Array string to the verifier API (or the tenant).

Rules are added to the agent similar on how it is done currently for `mb_refstate`.
A new attribute `revocation_rules` is added to the agent to hold that information.

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

### Upgrade / Downgrade Strategy
A new table `revocation_events` needs to be created to store the already send
revocation events. Otherwise by default Keylime will still operate in the old
binary state.

New fields in the API are introduced so an API version update is needed to
indicate that change.

### Dependency requirements
No additional dependencies should be required.

## Drawbacks
 * This will add additional complexity for features that cause revocation events.
 * It creates more data for Keylime to manage.

## Alternatives

## Infrastructure Needed (optional)

<!--
Use this section if you need things infrastructure related specific to your enhancement.  Examples include a
new subproject, repos requested, github webhook, changes to CI (travis).
-->
