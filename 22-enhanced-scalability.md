# enhancement-22: Enhanced Scalability and High Availability for Cloud Verifier and Registrar

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
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Test Plan](#test-plan)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed (optional)](#infrastructure-needed-optional)
<!-- /toc -->

## Release Signoff Checklist

- [ ] Enhancement issue in release milestone, which links to pull request in [keylime/enhancements]
- [ ] Core members have approved the issue with the label `implementable`
- [ ] Design details are appropriately documented
- [ ] Test plan is in place
- [ ] User-facing documentation has been created in [keylime/keylime-docs]

## Summary

Enhance the scalability of the Registrar and Verifier so that we can
monitor very large clusters of target nodes (10k+) each with their own
agent. Since the design of a good scalable system has a lot of overlap
with High-Availability, this enhancement will also take into account HA
principles so there are no single-points-of-failure. This will require
changes to the registrar, the verifier, the agents, and the tenant
command. We want to achieve near linear scaling for full continuous
attestation including IMA events.

## Motivation

Currently the default operating mode for the Registrar and Verifier is
to be run as a single instance. These instances can be run on the same
machine or on different machines and the backing data can be stored
in either a local SQLite database or a full RDMS system (MySQL or
Postgres). Agents and Tenants run API requests against these processes
where ever they are running. But each agent is tied to a single verifier
and a single registrar. If one of those goes down continuous attestation
is left hanging until someone can manually restart the dead processes
and re-add the agent to either the registrar or verifier.

In addition, there is a finite capacity to how many agents a single
verifier can handle. This threshold varies by setup (depending on how
much is being monitored and how busy those events are) but it caps out
at a few hundred. This proposal aims to give near linear scalability
such that adding more verifier and registrar nodes is simple, reliable
and well documented.

### Goals

* Allow Registrars and Verifiers to be load-balanced. This will
support both client-side (agent's choosing different targets based
on configuration) and server-side (adding a load-balancer in between
the agent and the target) or a combination of the two, allowing for
flexibility, fault tolerance and multi-zone deployments.
* Move in-memory state out of the Registrars and Verifiers into a shared
backing store. This could leverage the existing RDMS store for more
key/value type data, or introducing a new store (eg, etcd) to maintain
this state.
* Detailed documentation about how to configure and manage a cluster
for Scalability and HA.
* A virtualized and containerized developer environment for HA/Scalability
testing and development.

### Non-Goals

* Keylime should not be in the distributed database business. This means:
    * Keylime will not ship with an embedded data store
    * We will not produce documentation on how to scale the datastore itself
    * Hopefully we can continue to an RDMS store and keep it
    abstracted. If our testing does point toward performance
    problems/benefits with a particular solution we can document that.

## Proposal

The following are proposed:

* Identify the data that is stored in-memory in the registrar and
verifier. Determine a serialization strategy for that data (protobuf,
avro, bson, etc) and then change the code to deserialize on each request
and put any changes serialized back into the data store.
* Determine if we can continue with the RDMS abstracted datastore that
we currently use for other things for this data.
* Implement client-side load balancing in the agents and the tenant. For
instance, change `keylime.conf` to allow the `cloudverifier_ip` to
optionally be a comma-separate list of IPs along with a hash function so
each agent has a different order of those IPs. The agent would start with
the first in their list and fall back to the others as ordered for them.
* Test and verify that server-side load-balancing works in between the
agents/tenant and the registrar/verifier.
* Create detailed documentation about how to setup keylime for an
HA/Scalable deployment
* Create a virtualized and containerized environment with multiple agents,
registrars, verifiers and backing store nodes. This will not only provide
a detailed demonstration of how all the pieces fit together, but also
provide a nice development environment for improving things going forward.

### User Stories (optional)

#### Story 1

As a keylime operator I want to be able to use Keylime for a large cluster
(10k+) of machines with agents installed. I need it to be responsive
to scaling up and easy to scale down. I want to be able to do real-time
boot attestation as well as run-time attestation of IMA events.

#### Story 2

As a keylime operator I want to be able to setup Keylime to run with
high durability. If multiple keylime infrastructure components (registrar,
verifier, data-store) nodes go down I want the cluster to continue to
operate and provide attestation as long as possible given my redundancy.

I also want it to be easy and straight forward to add more redundant
capacity to my keylime cluster.

### Risks and Mitigations

* Since there are not currently any large scale Keylime deployments that
we know of (aside from some experiments) any changes here should be
relatively risk-free. Care will need to be taken that existing setups
(non-scalable, non-HA) continue to perform as expected. If we need to
sacrifice some individual node performance for scalability we will need
to verify that single instance use-cases are not drastically impacted.

* Because we are adding more data to the backing store care will need to
be taken so this data is treated securely. If there is sensitive data,
or data we need to protect against tampering, it will need cryptographic
protections.

## Design Details

Since there is some ambiguity in how some goals will be implemented,
most of this section is TBD. It will be flushed out as work progresses.

### Test Plan

TBD

### Upgrade / Downgrade Strategy

TBD

## Drawbacks

* Any time something becomes more scalable it becomes more complicated. Is
that complexity worth it? Does the complexity potentially compromise
the security?
* It's possible that moving the in-memory state out of the verifier and
registrar adds noticeable performance penalties to API requests from
the agent and the tenant. While the overall scalability will be enhanced
the per-node performance might suffer.

## Alternatives

It might be possible to achieve a subset of these goals with just
client-side load-balancing, automatic restart of failed verifiers and
registrars, and automatic re-registration and re-tenanting of agents. But
this would likely involve an outside orchestrator, limit the deployment
scenarios, and have windows where attestation would not be current.

## Infrastructure Needed (optional)

We shouldn't need any extra infrastructure. Although, we'll probably
be using several test labs setup by different developers' employers for
large scale testing.
