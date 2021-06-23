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
This enhancement proposal adds tagging of revocation events and the capability
differentiate and classify revocation events by adding severity levels.

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
   part in Keylime that causes a potential revocation.
   * Tagging all parts that cause revocations in Keylime with an unique id.
   * Provide the option for Keylime to add context to revocation events.
    This is useful if for example Eventlog parsing gains support for dynamic
    policies.
   * Extend the verifier API and tenant to add those rules to Keylime.
 * Providing all the necessary information about an event for future revocation
   mechanisms.
### Non-Goals
 * The classification of revocation events is highly environment and
   configuration specific. It is not a goal to provide a default configuration
   other than all events are classified with the highest severity.
 * Changing the Keylime databases to keep which events were already sent. This
   would break the assumption that the databases are "non historical".
 * Implementing a new way for adding revocation mechanisms.


## Proposal
This proposal consists of two parts. Part A can be implemented without part B.

### Part A - Tagging of all parts of Keylime that can cause a revocation
In the current model every check that fails causes that the agent is not
verified anymore and no further information what exactly failed is recorded.
Checks that are still outstanding are not executed after the first failure
occurs.

Part A changes that by tagging each component in Keylime that might cause a
revocation event and trying to evaluate all checks instead of aborting if one
check failed.

### Part B - Extending the revocation events to support severity levels
This part uses the gained capabilities from part A and adds severity level
functionality to the current revocation mechanism.

Currently revocation messages are either send if something with the
communication to the agent went wrong, the quote is not valid or one of the PCR
based checks fail. This part changes that behavior by introducing the concept of
severity levels. Now messages are send if a event if a higher than the currently
recorded severity level occurs.

Instead of stop polling an agent if a failure occurred the agent is added back
for checking if the failure is recoverable. If it is not a revocation event with
the highest severity is generated.

The user can specify for an event a severity level on a agent by agent basis.


### User Stories (optional)
User stories belong to part B of this proposal. Part A only provides internal
design changes for future enhancements.

#### Story 1
* User specifies that all `ima` events have a severity level of `warning` and
  all a failure of `pcr_validation` has a severity level of `err`.
* Agent B removes Agents from a system if any revocation message with the
  severity level `err` is generated.
* Agent A has a file that fails the ima check and the failure object contains an
  event with the event id `ima.ng-sig.hashfailed`.
* Failure object is evaluated and the highest severity level is `warning`.
* Agent A hadn't had a failure before so a revocation message is sent with the
  severity level `warning` and for agent A the `severity_level` is set to
  `warning`.
* Agent B ignores the revocation message based on the level.

#### Story 2
* Setup is the same as the end of Story 1.
* Agent A has still a file that fails the ima check and the failure object
  contains an event with the event id `ima.ng-sig.hashfailed`.
* Failure object is evaluated and the highest severity level is `warning`.
* The `severity_level` of agent A is `warning`so no revocation message gets send.

#### Story 3
* Setup is the same as the end of Story 1.
* Agents A pcr10 has a now a wrong value and the failure object contains now an
  event with an event with the event id `ima.ng-sig.hashfailed` and also one
  with `pcr_validation.pcr1`.
* Failure object is evaluated and the highest severity level is `err`.
* `err` is a higher severity level than `warning` and revocation message with
  the severity level of `err` is sent and for agent A the `severity_level` is
  set to `err`.
* Agent B removes agent A from a system.

### Notes/Constraints/Caveats
Part B of this proposal is limited by the database design of Keylime. More
flexible solutions can be implemented outside of Keylime when an API for
external revocation mechanisms gets implemented.

### Risks and Mitigations
The option to classify the revocation events opens the possibility that events
are not handled. To mitigate this all revocation events that are not explicitly
from the user classified are assigned the highest severity level.

The same concept applies for revocation events that are caused by failures that
make the agent irrecoverable. Those also always need to be treated with the
highest severity by default because otherwise some events won't be caught if a
irrecoverable failure is triggered before.

## Design Details

### Part A
#### Event tagging
Events are tagged with an event id. Which has the following schema:
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

Events can have a specified context. Those can be either a static string or an
JSON object that contains more information about that event. This is optional
and it is gernerally assumed that two events from the same agent with the same
event id have the same severity and are the same event.

### Collecting events
Instead of returning early if one check in a component fails a new failure
object will be introduced to collect the events. All parts of the validation
process such as (`check_qoute` and `check_pcrs`) will append their generated
events to that object instead of returning false. 

If the validation process cannot advance without a step that failed the failure
object must be marked as irrecoverable and only then a function can return
without validating any further. This will be the case if e.g. quote validation
failed.

Recoverable in this context means that the validation can continue without that
check needing to succeed, not that if the check succeeds in the future
again the agent can get back into a verified state.

Without the implementation of part B a revocation event will be sent if any
check failed and the polling of the agent will be stopped.

### Part B
#### Severity levels and user rules
The severity level is described by a label. By default following labels are
available: crit, err, warning, notice, info and debug. They are strictly ordered
from hightest to lowest severity. The labels are configurable in the
`keylime.conf` to allow finer granularity if necessary.

The user can specify severity level for event ids. 
One rule contains the following attributes:
 * `event_id`: The event id to match or a regex rule for it.
 * `severity_level`: The severity level that the matched events have.

To make the rules future proof with dynamic policies it is possible to specify a
regular expression for that matching. This introduces more complexity on the
parsing and matching side, but allows for flexible rules. Rules are parsed in a
top down order and the first matching rule is used and are supplied as a JSON
Array string to the verifier API (or the tenant).

Rules are added to the agent similar on how it is done currently for
`mb_refstate`. A new attribute `revocation_rules` is added to the agent data in
the verifier to hold that information. The rules can be specified on a per agent
basis when the agent is added to the verifier. 

The tenant is extended to support that functionality. 

#### Changes to the state machine and revocation events
We keep the current model of the states, but modify the behavior of the failure
states. If we are in a failure state that is recoverable the polling of
the agent is stopped otherwise the agent is still added back for normal polling.

If the failure object is marked as irrecoverable the state of the agent after
that should be `QUOTE_FAILED_IRRECOVERABLE`, a revocation event with the highest
severity level gets generated and the agent gets removed from polling.

To the agent table in the verifier a new column called `severity_level` is added. 
It contains the highest severity level that a generated event had.

To the revocation message a new field called `severity_level` is added which
contains the highest severity level that was generated by an event that caused
the message to be sent.

The `process_agent` function gets a new argument that can contain a failure
object. If the status is `QUOTE_FAILED` the events from failure object are
evaluated against the user specified rules and if the highest severity level is
higher than the saved in `severity_level` a revocation message is sent and
`severity_level` is updated to the new highest severity level.

This checking against `severity_level` is done to prevent spamming the agents
with messages that don't contain new information.

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
A new column `severity_level` to the table `verifiermain` gets added.
Otherwise by default Keylime will still operate in the old binary state.

New fields in the API are introduced so an API version update is needed to
indicate that change.

### Dependency requirements
No additional dependencies should be required.

## Drawbacks
 * This will add additional complexity for features that cause revocation events.
 * Part B changes the current binary state of the agents verification status.

## Alternatives
 * Only implement the tagging (part A) and completely redesign the revocation mechanism.

## Infrastructure Needed (optional)

<!--
Use this section if you need things infrastructure related specific to your enhancement.  Examples include a
new subproject, repos requested, github webhook, changes to CI (travis).
-->
