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
# enhancement-45: API Versioning

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories)](#user-stories)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
<!-- /toc -->

## Release Signoff Checklist

- [ ] Enhancement issue in release milestone, which links to pull request in [keylime/enhancements]
- [ ] Core members have approved the issue with the label `implementable`
- [ ] Design details are appropriately documented
- [ ] Test plan is in place
- [ ] User-facing documentation has been created in [keylime/keylime-docs]

## Summary

This enhancement describes the changes necessary for a consistent, global
strategy for versioning the various APIs inside of keylime. This will
make it easier to have consistent guarantees between various components as
different versions of each exist in real-world deployments.

## Motivation

With the upcoming addition of the new Rust-based keylime agent there
will need to be more explicit guarantees and control about changes in
the APIs between various keylime components. We can no longer rely on
releasing the client and server-side components together in a single
package. For this reason (and others) we need to formalize our API
versioning strategy and guarantees between different versions.

### Goals

We want to achieve the following goals:

* Allow older versions of agents to talk to newer versions of server-side components (within reason)
* Allow minor upgrades to APIs without requiring a new version (new fields with reasonable defaults, etc)
* Allow minor API version upgrades to server-side components to be transparent to existing agents without requiring their upgrade.
* Implement this in a way that doesn't require lots of copy/pasted code or near duplicate code for different versions of the API

### Non-Goals

The following are not goals for this enhancement:

* Support for previous versions of the API structure. This one time can be a clean break.
* Perpetual support for all versions of the API.

## Proposal

Versioning will exist for all components and will follow a `$MAJOR.$MINOR` numbering scheme similar to [Semantic Versioning](https://semver.org/). Backwards compatibility will be supported for at least 2 major versions. Forward compatability will be supported within the same major version family.

### User Stories

#### Story 1

An organization has a fleet of machines deployed with a `keylime-agent` which knows about a specific version of the keylime API (v1.2). The fleet's operators want to upgrade the keylime server side components to a newer version of keylime which is running a newer version (v2.3). When the upgrade is complete on the server side, the existing agents in the fleet should be able to continue their attestations as before without any problem communicating with the server.

#### Story 2

An organization is in the process of changing from a python-based `keylime-agent` to a rust-based one. The python-based agent was using the v1.1 version of the API and the new rust-based agent is using v1.2. The keylime server-side components are using the v1.3. Machines that have their agent replaced should continue to be able to perform attestations.

#### Story 3

An organization has upgraded their machine images to a new version of the `keylime-agent` which uses the v1.5 version of the API. Their existing machines use a previous version of the agent which uses the v1.2 version. Their server-side components are currently supporting v1.4 of the API. They first upgrade their server-side components to the latest version which is now at v2.1 of the API. Then new machines with the v1.5 client can communicate with the server side and existing clients running v1.4 will also continue to operate without problems. The fleet should be able to operate in this "mixed" mode of different client versions without issue.

### Notes/Constraints/Caveats

* The current API will be pegged at `v1.0` and when the feature is released, all components will be updated together. This will be the last time that this is required as future versions will allow the client and server-side components to upgrade independently. But we need this clean break from previous URL paths to give us this foundation for the future.
* It's difficult to predict the future, so it may not be possible to restrict the project to supporting just the latest 2 major versions, but that's a good place to start.
* This proposal does not include any API version negotiation. It should be sufficient for the client code to use a single version as long as it's in a major version family that the server-side supports.
* This proposal does not guarantee the ability for a newer client (using say `v3.4`) to communicate with an older server-side component (using say `v3.3`). Backwards compatible changes are within scope, not future-compatible ones. But it also doesn't explicitly disallow it. In some cases it might work out in practice for scenarios like the following: The only difference between `v3.4` and `v3.3` is the addition of a new optional field. The server side would just ignore this new unknown field. But any organization doing this would have no guarantees and would need to do their own extensive vetting.

### Risks and Mitigations

There is always the risk that this proposal is not flexible enough and that we would need to revist in the future. Hopefully this can be done without needing another "clean break" in API URL versioning and can just be built on top of this one with new strictures and guarantees.

It's possible that there are security vulnerabilities discovered in a previous version that is still supported where those vulnerabilities are inherent in how the API is structured. If such a vulnerability could not be fixed without breaking the API then we would need to review as a project how to deprecate support for that particular version without breaking the whole major version family if possible. If that is not possible, some other mitigation strategies would need to be decided on a case-by-case basis.

## Design Details

The design of the versioned API consists of the following pieces (numbered for convenience not to denote implementation order):

1. Versioning will exist for all APIs on all components
2. The APIs that currently exist will be versioned `1.0` and will have the URL prefix `v1.0`.
3. There will exist an alias for each major version that is aliased to the latest minor version. For instance, if the latest version in the v1 family is 1.3, then the `v1` alias will act as if it were `v1.3`. This allows clients to peg their implementations to a major version without having to worry about every minor version upgrade, but also allows each version of the API to be cataloged and documented with each change. For example, version `v2.3` will always be exactly the same no matter what happens in the future, even if the `v2` alias could point to a new `v2.4` in the future. This alias could be implemented in the code or via an HTTP redirect response.
4. The format of `v$MAJOR.$MINOR` will be similar to semantic versioning. Backwards incompatible changes will require a bump in the `$MAJOR` number and changes that are backwards compatible will require a bump in the `$MINOR` number. Changes such as bug fixes, performance enhancements, etc that don't change the API won't require any changes to the API version. For instance:
    * Adding a new optional field would require a reasonable default for the field's value if missing and would require a bump in the minor number.
    * Marking a field as *deprecated* would require a bump in the minor version, but as long as the code simply ignores the field it does not require a major version bump.
    * Adding a new API endpoint would be a minor version bump as previous clients would not be using it
    * Removing a field completely where the presence of that field would cause an error requires a major version bump. This should be very rare as the API implementations should usually just ignore fields they don't understand.
    * Changing the semantics of a field's value requires a major version bump
    * Changing a field from optional to required requires a major version bump
    * Removing an API endpoint requires a major version bump.
5. All components that have API endpoints will need to support the current major version (and all it's minor versions) and the most recent previous major version (and all it's minor versions). For example: a `keylime_registrar` with the latest API version of 3.4 will need to support all previous 3.x versions as well as all 2.x versions. This will effectively mean as a project, Keylime will only support the current and most recent major versions of the API.
6. Tests should be written to exercise all supported versions of the APIs. Any new features which introduce major or minor version bumps will need to include tests for the new APIs as well as the old.
7. When a new major API version is released, no new minor versions for the previous major version will be released. For instance, if the current API version is 3.4 and a new 4.0 is released, there will not be a 3.5 version. Bugs can be fixed in previous versions, but no enhancements made.
8. Re-organize the code such that API version specific handling can be handled as early as possible and the body of code that does the actual work can be shared between versions. If a new version is drastically different in the way it implements something, it's fine for it to use new code. But code re-use should be a high priority. This will help us to not have to fix bugs in multiple locations for different versions. This of course will evolve and likely need refactoring as we move into the future and have more versions to support in real-world deployments.

### Test Plan

All current tests would need to be updated to point to the `v1.0` base API URL. Any new major or minor versions to the API would need accompanying tests before they could be merged. We are making a guarantee of support for previous versions as a project, so that guarantee needs to be backed up by tests.

### Upgrade / Downgrade Strategy

The initial upgrade to keylime code that supports the versioned APIs will have to be as a single unit. Client (agent and tenant) and server (registrar and verifier) will need to be upgraded from the same release. This isn't really a problem since that's how keylime upgrades currently work.

Downgrading to versions before the versioned APIs should work as any other keylime downgrade (all pieces need to be downgraded together).

Going forward, upgrading and downgrading client and server components should be independent as long as it's done within the supported version constraints.

### Dependency requirements

We shouldn't need any new 3rd party dependencies for this work.

## Drawbacks

Adding multiple supported versions adds complexity to just about everything. Troubleshooting, bug reporting, new features and bug fixes will all be more complicated when we have to deal with multiple supported versions talking to each other. Unfortunately, if we want to support real life deployments with mixed versions we'll have to tackle this and accept the extra burden.

## Alternatives

We could have gone with a simpler versioning structure where we just used major version labels and anything that was backwards compatible just went into the same major version without anything noting a change in the API. But this approach can be confusing if for example you look at a version of the documentation tomorrow that has fields you didn't see yesterday. Using a `v$MAJOR.$MINOR` schema allows implementors to follow a version of the docs they know won't change underneath them, but still allows for backwards compatible future enhancements.

