# Keylime Enhancement Tracking and Backlog

The enhancement process has been in implemented to provide a way to review and assess the impact(s) of significant changes to Keylime. This in turn eases the management of our roadmap and release planning.

The Keylime Enhancement Tracking and Backlog process is loosely based on the [Kubernetes Enhance Proposal system](https://github.com/kubernetes/enhancements/tree/master/keps)

This repo contains issues that map to enhancements that drive change requests targeted for Keylime and it's ecosystem. These enhancements are umbrellas for new features to be added to Keylime. An enhancement may take multiple releases to complete and be planned accordingly.

To view accepted enhancments under current development view with the [in-progress label](https://github.com/keylime/enhancements/issues?q=is%3Aissue+is%3Aopen+label%3Ain-progress)

To view enhancements under review, view with the [backlog label](https://github.com/keylime/enhancements/issues?q=is%3Aissue+is%3Aopen+label%3Abacklog)

## Is My Thing an Enhancement?

We are trying to figure out the exact shape of an enhancement. Until then here are a few rough heuristics.

An enhancement is anything that:

- requires significant effort or changes to Keylime in a significant way
- impacts the operation of Keylime substantially such that engineers using Keylime will need retraining
- users will notice and come to rely on it.

It is unlikely an enhancement if it is:
- fixing a test / bug
- refactoring code
- performance / security improvements
- adding error messages or events

If you are not sure, please ask within a keylime issue, the gitter channel or
the mailing list.

## When to Create a New Enhancement

Create an issue here once you:
- have circulated your idea to see if there is interest (for example on the
  keylime mailing list or gitter community channel)
- (optional) have done a prototype in your own fork
- are ready to be the project-manager for the enhancement

## Why are Enhancements Tracked

Once users adopt an enhancement, they expect to use it for an extended period of time. Therefore, we hold new enhancements to a high standard of conceptual integrity and require consistency with other parts of the system, thorough testing, and complete
documentation. As the project grows no single person can track whether all those requirements are met. The development of an enhancement often spans three stages: Alpha, Beta, and Stable; Enhancement Tracking Issues provide a checklist that allows for different approver's for different aspects, and ensures that nothing is forgotten across the development lifetime of an enhancement.

## When to Comment on an Enhancement Issue

Please comment on the enhancement issue to:
- request a review or clarification on the process
- update status of the enhancement effort
- link to relevant issues in other repos

Please do not comment on the enhancement issue to:
- discuss a detail of the design, code or docs. Use the pull request for that.

## How to create an enhancement

Create an issue in keylime/enhancements and then make a pull request to keylime/enhancements (using the available template). The issue is then used to map to the pull request, track the enhancement in the project board and map it to a planned milestone release. The pull request to keylime/enhancements is to allow community review and approve the enhancement.

New enhancements are labelled as 'backlog'

Once the pull request is approved and then merged it is labelled as 'in-progress'
and work is expected to start (this is not to say you cannot pre stage work into
your own fork first).

When you then make pull requests to any given Keylime repository, please reference
the enhancement issue in your commit message (e.g. `#353`, you can also reference
the issue number when discussing a topic in any of the keylime repositories). For example
someone may make a change which is going to have a possible impact upon an enhancement
labelled as in 'backlog' or 'in-progress'.

Once the feature is delivered (the "Release Signoff Checklist" is completed) and all
pull requests are merged, with a planned and agreed release (TODO) then the 'complete'
label can be applied.

## I want to raise an issue around the enhancements process

Sure, go ahead. Please use `[Admin]` in the subject field of the issue and ensure it will then be tagged as `house-keeping`.
