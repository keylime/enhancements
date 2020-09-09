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
# enhancement-7: Multi Tenancy

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

Keylime is at present monolithic in that there is no concept of multi tenancy in
the form of groups or users.

This enhancement sets the foundation for scoping Keylime into a mutli tenant
capable system.

## Motivation

<!--
This section is for explicitly listing the motivation, goals and non-goals of
this enhancement.  Describe why the change is important and the benefits to users.
-->

Keylime works in the context of a single tenant. Any agents registered within
Keylime can be seen by all (who have access to the system).

Should an owner of a Keylime deployment want to host multiple tenants, there will
be no isolation on interaction of view and management rights of a registered agent.

An owner of a Keylime deployment may want to provide services to multiple tenants
(or users) and allow them to manage and create their own users , groups and
administrators while isolating views of agents to just members of that group.

### Goals

<!--
List the specific goals of the enhancement.  What is it trying to achieve?  How will we
know that this has succeeded?
-->

Implement an a rudimentary multi tenancy model within Keylime to allow the creation
of users, groups and administrators of said groups.

Implement a root admin who has an overall level or privilege to create and manage
groups and group administrators.

Implement a means to fix an agent to a group and user.

Provide a scalable authentication system to arbiter access of keylimes APIs.

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

This proposal will seek to implement the following features:

Implement database amendments to introduce groups, users and roles.

Implement a method for a user or administrator to provide credentials to prove
their identity to Keylime. In turn these accounts can be disabled, enforce
password rotation and have view rights limited to those granted to them via group
administrators.

Implement authentication in the form of JSON Web Tokens to allow federation over
multiple instances of Keylime.

Implement a means of associating an `agent_id` to a `group_id` and a means for
administrators to re-associate the `agent_id` with a different `group_id`
("change group").

### Risks and Mitigations

<!--
What are the risks of this proposal and how do we mitigate.  Think broadly.
For example, consider both security and how this will impact the larger
enhancement ecosystem.

How will security be reviewed and by whom?
-->

Multi tenancy introduces its own set of security risks in the form of information
leakage and unauthorised access.

This change will seek to use existing security modules such as PyJWT and will
build upon the existing access controls already present in Keylime (mTLS).

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable.  This may include API specs (though not always
required) or even code snippets.  If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->

Central to the design, will be the introduction of "users" to the system. Users
are accounts which are available for machines / humans to interact with the
RestFUL API's available within the verifier.

Alongside users, will be the introductions of "groups". "users" will belong to
"groups" and will only be able to operate within the context of the group(s) to
which they belong. Agents (`agent_id`) will also belong to groups. This will
then restrict view rights of an agent, to group in which the agent is registered.

A root admin will be created within the system at deploy time and will be
unremovable. Creation of new groups will only be possible using the root admin
account. When a new group is created, an admin role for that group will be
automatically created.


### JWTauth

JSON Web Tokens will be introduced as a means for users to authorise themselves
and access / interact with the verifiers APIs.

JWT is proposed as it affords us a means to federate access over multiple
verifiers.

To enable JWT, the [PyJWT](https://pyjwt.readthedocs.io/en/latest/) module
will be imported into Keylimes code and become Keylime dependency.

Users will then be required to authorise themselves and upon successful
authorization, be provided with a time based scoped JWT token.

To deliver this feature it will require a `@JWTauth` python decorator which can
easily be set over each handler that requires authorised access controls.

### Database changes

There will also be new tables introduced to the verifiers database

 `users`: a table store user information
 `groups`: a table to store group information
 `roles`: a table to store role information

Declarative mappings will be made from `agent_id` in the `verifiermain` table
to a new `groups` table. This will allow isolation of agent viewing rights to
only users within the same group.

Declarative mappings will also be made from `users` to `groups` and `roles`

#### users table

| row name | row properties                                |
|----------|-----------------------------------------------|
| user_id  | Integer (primary_key)                         |
| username | String                                        |
| password | String                                        |
| email    | String                                        |
| group_id | Integer (declarative mapping to groups table) |
| role_id  | Integer (declarative mapping to roles table)  |

#### groups table

| row name   | row properties                                                          |
|------------|-------------------------------------------------------------------------|
| group_id   | Integer (primary_key) (declarative mapping to users table)              |
| group_name | String                                                                  |
| role_id    | String (declarative mapping to the roles table)                         |

#### roles table

| row name  | row properties                                                      |
|-----------|---------------------------------------------------------------------|
| role_id   | Integer (primary_key) (declarative mapping to groups & users table) |
| role_name | String                                                              |
| group_id  | String (declarative mapping to the groups table)                    |

The `werkzeug` library will be used for secure password salting of database
stored user passwords.

### New handlers

JWT handlers will be introduced to provide authorization protect access of the
`AgentsHandler` tornado handler.

Two new handlers will be introduced:

#### UsersHandler

`UsersHandler` - will provide the following HTTP methods:

`GET`: Get a list of users or details of a specific user from the system (requires
  the caller to be the admin of the group in which the user resides. A call for
  a specific user can only be made by the admin or the user themselves.

`POST`: Limited to only root admin or group admin. Allows creation of a specific
    user. The following params should be populated:
      `username`: Username of the new user
      `password`: Password of the new user
      `email`: New users email address
      `group_id`: The group_id of the users primary group

A `role_id` will be auto generated (incremented from last value).

`PATCH`: Patch will be used to change user passwords or amend the group(s) of an
  existing user.

 `DELETE`: Limited to only root admin or group admin. Allows deletion of a user.

#### GroupsHandler

`GroupsHandler` - will provide the following HTTP methods:

All calls to the `GroupsHandler` require a root admin account, with the exception
of the `GET` method.

`GET`: Get a list of groups or details of a specific group from the system. Requires
  the caller to be the admin of the group in which the user resides or the root admin.

`POST`: Allows creation of a new group. The following params should be populated:
      `group_name`: Username of the new user
      `group_desc`: Password of the new user
=
`PATCH`: Patch will be used to change groups name.

`DELETE`: Allows deletion of a group. This operation will cascade remove all users
 and agents and admins that exist within the group.

#### AuthHandler

`AuthHandler` - will extract the username and password from a `GET` request to
 an `auth` uri. The `werkzeug.security` module will then be used to perform
 a database lookup using `check_password_hash`

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

Unit tests will be provided to test the database classes and JWT. Tests will also
be amended to require that a JWT token be scoped.

### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this enhancement
-->

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

No alternatives are known of.
