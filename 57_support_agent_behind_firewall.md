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
# enhancement-57: Support Agent behind firewall

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

This enhancement describes the changes necessary to support agents behind firewalls and 
networkproxies, at the far edge.

## Motivation
With the current design, the verifier initiates the communication to the agent. 
In far edge deployments, agents (remote edge devices) are usually located behind
firewalls. Thus, they can not be reached by the verifier. Opening firewall ports 

### Goals

We want to achieve the following goals:
* Attestation of devices behind firewalls is possible.

### Non-Goals

The following are not goals for this enhancement:
* ?

## Proposal
Provide a way that the agent initiates the communication to the verifier via firewall and network proxy friendly protocol (probably HTTPS)

### User Stories

#### Story 1

As operator of a fleet of edge devices at the far edge or customer locations, I want to be able to atest my devices even if they are behind firewalls 
and/or connect via network proxies, so that I can still establish trust. 

### Notes/Constraints/Caveats

### Risks and Mitigations

## Design Details

### Test Plan

### Upgrade / Downgrade Strategy

### Dependency requirements

## Drawbacks

## Alternatives

