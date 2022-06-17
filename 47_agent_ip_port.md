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
# enhancement-47: Let the agent provide contact IP and port for verifier

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
The agents gains the ability to provide the registrar with a contact ip and port
for the verifier.

## Motivation
The verifier needs a port and ip adress to contact a agent. 
Currently this information is manually provided to the tenant.

This information can also be provided by the agent in most cases. It makes
Keylime more user friendly in environments where no external configuration
management is used.

Allowing configuration with environment variables allows other tools to set this
information easily before the agent is started.
### Goals
 * Add support for the agent to specify a contact ip and port
  * Static configuration in config file or specified via environment variable
 * Extending registrar API to allow specifying contact ip and port 
 * Add support for using the specified ip and port to the tenant

### Non-Goals
 * Moving Keylime to model to a push only model
 * Allow IP changes after registration
 * Auto detection of IP and port

## Proposal
This enhancement proposal adds that the agent can provide a contact IP and port
for the verifier to the registrar. The tenant does not have to ask the user
manually for this data anymore and tries to retrieve it from the registrar. 

### User Stories

#### Story 1
 * Agent has contact ip and port configured
 * Agent registers itself with the registrar
 * User adds agent with `keylime_tenant -c add -u AGENT_ID`
 * Tenant retrieves the ip and port information from the registrar
 * Agent is added to the verifier

#### Story 2
 * Agent has contact ip and port not configured
 * Agent registers itself with the registrar
 * User adds agent with `keylime_tenant -c add -t 127.0.0.1 -u AGENT_ID`
   (`127.0.0.1` is the IP of the agent)
 * Agent is added to the verifier

#### Story 3
 * Agent has contact ip and port not configured
 * Agent registers itself with the registrar
 * User adds agent with `keylime_tenant -c add -u AGENT_ID`
 * Tenant fails because the IP cannot be retrieved from the registrar

### Risks and Mitigations
The input by the agent is generally not trusted and must be validated.
Is is done in the registrar. 

## Design Details
To the registrar database table `registrarmain` two new columns are added: `ip`
and `port`. Those entries can be NULL.

The fields `ip` and `port` can be optionally specified in the registrar API when
a agents tries to register. Simple input validation for those fields is added in
the registrar.

The agent configuration gains two new optional fields `agent_contact_ip`
and `agent_contact_port`. Those options can also be specified as
environment variables with `KEYLIME_AGENT_CONTACT_IP` and
`KEYLIME_AGENT_CONTACT_PORT`. Environment variables have a higher precedence.

The `--targethost` option for the tenant is made optional and if not specified
the tenant tries to retrieve the data from the registrar automatically. The same
goes for the port. The tenant can assume that this data is validated. 
The precedence is first command line argument, values from the registrar and
then default value from config (last one only applies to port).


### Test Plan
 * Extending the `test_registrar_db.py` test to test the new fields
 * Extending the `test_restful.py` tests to check for the registration with an
   IP and port and without.

### Upgrade / Downgrade Strategy
To the `registrarmain` table two new columns are added `ip` and `port`. Those
fields can be NULL. The registrar API gains two new fields `ip` and `port` but
those can be empty such that old agents still can connect.

A minor update to the API should be done to indicate that the registrar supports
the new fields.

A downgrade of the agents should possible without any changes. If the registrar
is downgraded the columns can be removed and the tenant now needs to specify
those fields explicitly again.

### Dependency requirements
No additional dependencies should be required.

## Drawbacks
 * Additional data is stored which in some cases could also be retrieved from
   tools outside of Keylime. 

## Alternatives
 * In some enhancements the contact IP and port can also be retrieved via third
   party configuration tools.
 * Moving Keylime to a model where the agent polls the registrar and pushes the
   data to the verifier periodically. This would eliminate the need for the
   verifier or tenant to contact the agent directly but requires heavy changes
   to Keylime.
