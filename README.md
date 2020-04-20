# Keylime Enhancement Tracking and Backlog

The Keylime Enhancement Tracking and Backlog process is loosely based on the [Kubernetes Enhance Proposal system](https://github.com/kubernetes/enhancements/tree/master/keps)

This repo contains issues that map to enhancements that drive change requests targeted for Keylime and it's ecosystem. These enhancements are umbrellas for new features to be added to Keylime. An enhancement may take multiple releases to complete and be planned accordingly. And an enhancement can be tracked as backlog items before work begins. An enhancement may be filed once there is consensus with the Keylime community (TBD)

## Is My Thing an Enhancement?

We are trying to figure out the exact shape of an enhancement. Until then here are a few rough heuristics.

An enhancement is anything that:

- needs significant effort or changes to Keylime in a significant way
- impacts the operation of Keylime substantially such that engineers using Keylime will need retraining
- users will notice and come to rely on it.

It is unlikely an enhancement if it is:
- fixing a test / bug
- refactoring code
- performance / security improvements
- adding error messages or events

If you are not sure, please ask within a keylime issue.

## When to Create a New Enhancement

Create an issue here once you:
- have circulated your idea to see if there is interest (for example on the
  keylime mailing list or gitter community channel)
- (optional) have done a prototype in your own fork
- are ready to be the project-manager for the enhancement

## Why are Enhancements Tracked

Once users adopt an enhancement, they expect to use it for an extended period of time. Therefore, we hold new enhancements to a
high standard of conceptual integrity and require consistency with other parts of the system, thorough testing, and complete
documentation. As the project grows no single person can track whether all those requirements are met. The development
of an enhancement often spans three stages: Alpha, Beta, and Stable; Enhancement Tracking Issues provide a
checklist that allows for different approvers for different aspects, and ensures that nothing is forgotten across the
development lifetime of an enhancement.

## When to Comment on an Enhancement Issue

Please comment on the enhancement issue to:
- request a review or clarification on the process
- update status of the enhancement effort
- link to relevant issues in other repos

Please do not comment on the enhancement issue to:
- discuss a detail of the design, code or docs. Use the pull request for that.

## How to create an enhancement

Create an issue in keylime/enhancements and then make a pull request to keylime/enhancements (using the available template). The issue is then used to map to the pull request, track the enhancement in the project board and map it to a planned milestone release. The pull request to keylime/enhancements is to allow community review and approve the enhancement.

Once the pull request is approved and then merged, the issue can be moved from "Enhancements" to "Backlog" and work can begin (this is not to say you can pre stage work into your own fork).
