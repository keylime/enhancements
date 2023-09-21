# [enhancement-103](https://github.com/keylime/enhancements/issues/103): Agent-Driven Attestation

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Proposed Agent Authentication Mechanisms](#proposed-agent-authentication-mechanisms)
  - [Proposed Trust Mechanisms](#proposed-trust-mechanisms)
  - [Proposed Attestation Protocol](#proposed-attestation-protocol)
  - [Proposed Changes to Registration Protocol](#proposed-changes-to-registration-protocol)
  - [Proposed Changes to Enrolment Procedure](#proposed-changes-to-enrolment-procedure)
  - [User Stories](#user-stories)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Mutually Authenticated TLS (mTLS) for Agent Authentication](#mutually-authenticated-tls-mtls-for-agent-authentication)
  - [Challenge–Response Protocol for Agent Authentication](#challengeresponse-protocol-for-agent-authentication)
  - [Certificate Trust Store Mechanism](#certificate-trust-store-mechanism)
  - [Webhook Mechanism for Custom Trust Decisions](#webhook-mechanism-for-custom-trust-decisions)
  - [Agent Lifecycle](#agent-lifecycle)
  - [HTTP Proxy Support](#http-proxy-support)
  - [Test Plan](#test-plan)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Dependency Requirements](#dependency-requirements)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Acknowledgements](#acknowledgements)
<!-- /toc -->

## Release Signoff Checklist

- [x] Enhancement issue in release milestone, which links to pull request in [keylime/enhancements]
- [ ] Core members have approved the issue with the label `implementable`
- [x] Design details are appropriately documented
- [ ] Test plan is in place
- [ ] User-facing documentation has been created in [keylime/keylime-docs]

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

## Summary

This enhancement proposal describes a method of enabling _agent-driven_ attestation in Keylime. It has been adapted from an earlier document, [_Roadmap to Push Model Support in Keylime_](https://gist.github.com/stringlytyped/367dd6cd29141f4f97b142035203b12a), which includes additional context and detail.

In this new operational model, agents send attestations to the server proactively instead of waiting for the server to request them.

There are two supplements to this enhancement proposal which discuss peripheral topics:

- [_802.1AR Secure Device Identity and Agent-Driven Attestation_](https://github.com/stringlytyped/keylime-push-proposal/blob/main/supporting-materials/devids-and-push-model.md) explains how the proposal is compatible with IDevIDs/IAKs[^devid]
- [_Adversarial Model for Agent-Driven Attestation in Keylime_](https://github.com/stringlytyped/keylime-push-proposal/blob/main/supporting-materials/adversarial-model.md) describes the threat model contemplated by this proposal

## Motivation

The Keylime integrity verification system currently operates on a _pull_, or _server-driven_, basis whereby a verifier directs a number of enrolled nodes to attest their state to the server periodically. This model is not appropriate for enterprise environments, as each attested node thereby acts as an HTTP server. The requirement to open additional ports for each node and the associated increase in attack surface is unacceptable from a compliance and risk management perspective.

### Goals

The specific goals of this proposal are as follows:

- Allow Keylime to operate on a fully agent-driven basis.
- Provide a more secure operating model for enterprises and other security-concious users.
- Protect REST APIs from public access by authenticating the agents.
- Ensure attestations are authenticated as originating from the correct TPM for the device.
- Limit the attack surface of the Keylime agent.
- Flexibility to support different deployment modalities.
- The ability to expand the solution to support future cryptographic identities.

These are in service of the security and operational requirements stated below.

#### Security Requirements

The primary driver behind the need for the push model is a desire to limit the attack surface of the agent. The user needs to place significant trust in the agent software, so it should be compact, inspectable and should not unnecessarily affect system state. The user should not be expected to make significant changes to their system/network configuration or deploy additional controls in order to secure a Keylime installation (within reason).

Beyond these high-level concerns, special attention must be afforded to our protocols. See the separate [adversarial model](https://github.com/stringlytyped/keylime-push-proposal/blob/main/supporting-materials/adversarial-model.md) for the threat model we are considering in our protocol designs.

#### Operational Requirements

The environments in which users may deploy Keylime are diverse:

- Users may or may not have an existing public key infrastructure (PKI).
- Users may or may not have a record of their nodes in an inventory management system.
- TPMs may or may not make an EKcert available (physical TPMs usually do, but public cloud vTPMs often do not).
- Attested nodes may be issued with IDevID and IAK certificates[^devid] out of the factory, or may not.
- Users may deploy Keylime on a private network, expose it to the internet, or make it available on a semi-public network (e.g., that of a university campus).

A comprehensive solution should provide the user with options for deploying Keylime in all these situations.

### Non-Goals

These are not considered goals for this proposal:

- Requiring a single instance of the verifier to accept attestations from both push and pull agents side by side.
- Provide identity provisioning (payloads feature) in the new operational model (this is explained later in [Proposed Changes to Enrolment Procedure](#proposed-changes-to-enrolment-procedure)).
- Premature optimisation of the protocols.

## Proposal

Enabling agent-driven attestation in Keylime is not a simple matter of reversing the directionality of the protocols. There are a number of adjacent concerns which need to be considered. Given our security and operational requirements, which exceed those of the current pull model, it is necessary to revisit our trust and authentication assumptions.

While we ultimately need to make changes to the registration, enrolment and attestation protocols, these depend on authentication and trust mechanisms as shown in the below figure:

<p align="center">
    <img src="https://raw.githubusercontent.com/stringlytyped/keylime-push-proposal/main/diagrams/auth-and-trust-mechanisms.png" alt="Diagram showing that the attestation and registration protocols depend on an agent authentication mechanism and that the agent authentication mechanism and the enrolment protocol depend on a trust mechanism" width="612">
</p>

As different authentication and trust mechanisms are more or less appropriate depending on the deployment context, we do not fix these and instead provide the user with a couple options for each. This makes it possible to support many different environments without overwhelming the user with choices.

Our proposal therefore consists of the following:

- two new/improved agent authentication mechanisms;
- two new/improved trust mechanisms;
- a new agent-driven attestation protocol;
- changes to the registration protocol; and
- changes to policy enrolment.

### Proposed Agent Authentication Mechanisms

Attestations are of course authenticated by virtue of being signed by a TPM. But agents do not simply report attestations: they also need to access certain data stored by the verifier. An additional authentication mechanism is therefore needed to secure these API endpoints.

We propose that the user may choose between two agent authentication mechanisms:

- mutually authenticated TLS (mTLS) with pre-provisioned client certificates for the agents; or
- a challenge–response protocol secured by the TPM and the unilateral authentication provided by regular TLS.

These are discussed in the Design Details section, specifically in [Mutually Authenticated TLS (mTLS) for Agent Authentication](#mutually-authenticated-tls-mtls-for-agent-authentication) and [Challenge–Response Protocol for Agent Authentication](#challengeresponse-protocol-for-agent-authentication) respectively.

### Proposed Trust Mechanisms

Entity authentication is only useful when the entities are authenticated against a trusted identity. Here, we must be careful not to conflate the identity of the TPM with the identity of the agent or the identity of the attested node itself. This is because the Keylime protocols are effectively three way protocols between the TPM, agent and Keylime server (whether the registrar or the verifier). Per the stated [adversarial model](https://github.com/stringlytyped/keylime-push-proposal/blob/main/supporting-materials/adversarial-model.md), communications between the TPM and agent can be tampered with.

Reliably identifying the agent is likely impossible in our adversarial model. Any private key used by an agent to sign messages is also obtainable by an attacker resident on the attested node.

But even if we treat the agent as a simple intermediary between a TPM and the server without an identity of its own (like a network switch or router), this is insufficient. The root identity of the TPM, the EK, does not contain any information about which device the TPM is installed in (it does not contain a device-specific identifier). So, you cannot use the EK to identify the TPM as belonging to a particular device. From [TPM 2.0 Part 1, §9.4.3](https://trustedcomputinggroup.org/wp-content/uploads/TPM-Rev-2.0-Part-1-Architecture-01.07-2014-03-13.pdf#%5B%7B%22num%22%3A457%2C%22gen%22%3A0%7D%2C%7B%22name%22%3A%22XYZ%22%7D%2C70%2C453%2C0%5D):

> The TPM reports on the state of the platform by quoting the PCR values. For assurance that these PCR values accurately reflect that state, it is necessary to establish the binding between the RTR and the platform.

As such, we have two different trust concerns:

- **The need to identify a TPM as having been produced by a trusted manufacturer.** In order to have confidence in measurements of system state, it is necessary that the TPM behaves according to spec, e.g., that it correctly performs PCR extend operations and protects the contents of its registers (including keys) from access by outside entities except via the well-defined mechanisms.

- **The need to have a trusted binding between the TPM's (RTR) identity and the node's (platform) identity.** Otherwise, there is no assurance that a verification outcome is the result of applying the right verification policy to measurements of the right node.

In summary, when an agent sends an attestation or otherwise accesses a Keylime API, the trust that is placed in the agent's request needs to derive from some _root identity_ (e.g., EK), bound to a particular _logical node identifier_, which the user has specified as trusted (the user must trust the identity itself as well as the binding). The node may have certain _subordinate identities_ (e.g., AK) which are transitively trusted by their binding to one of the node's root identities. This is illustrated by the below figure:

<p align="center">
    <img src="https://raw.githubusercontent.com/stringlytyped/keylime-push-proposal/main/diagrams/trust-waterfall.png" alt="Diagram showing how trust in a verification result is derived from trust in a subordinate identity which in turn is derived from trust in a root identity and its binding to a logical identifier" width="800">
</p>

If the outcome of all the trust decisions shown on the diagram is favourable, we can trust that the right policy (associated with a node's logical identifier) is applied to the right attestation (produced by a subordinate cryptographic identity) by following both chains all the way to a trusted root identity.

Once again, we propose two different mechanism for making these trust decisions, which users can use independently or together at their discretion:

- a standard trust store mechanism for use when certificates are available to establish trust; and/or
- a webhook mechanism for when certificates are not an option or when custom trust logic is desirable.

The first already exists in Keylime in a form, but we suggest generalising it, so that it can be applied to a wider variety of use cases. This is discussed later in [Design Details: Certificate Trust Store Mechanism](#certificate-trust-store-mechanism).

The second is a new mechanism which is described in [Design Details: Webhook Mechanism for Custom Trust Decisions](#webhook-mechanism-for-custom-trust-decisions).

### Proposed Attestation Protocol

To obtain an integrity quote in the current pull architecture, the verifier issues a request to the agent, supplying the following details: 

- A nonce for the TPM to include in the quote
- A mask indicating which PCRs should be included in the quote
- An offset value indicating which IMA log entries should be sent by the agent

The agent then replies with:

- The UEFI measured boot log (kept in `/sys/kernel/security/tpm0/binary_bios_measurements`)
- A list of IMA entries from the given offset
- A quote of the relevant PCRs generated and signed by the TPM using the nonce

In a push version of the protocol where the UEFI logs, IMA entries and quote are delivered to the verifier as an HTTP request issued by the agent, the agent needs a mechanism to first obtain the nonce, PCR mask and IMA offset from the verifier. We suggest simply adding a new HTTP endpoint to the verifier to make this information available to an agent correctly authenticated with the expected certificate via mTLS.

As such, the push attestation protocol would operate in this manner:

1. When it is time to report the next scheduled attestation, the agent will request the attestation details from the verifier.

2. If the request is well formed, the verifier will reply with a new randomly-generated nonce and the PCR mask and IMA offset obtained from its database. Additionally, the verifier will persist the nonce to the database with the current timestamp. The nonce will expire after a configured period.

    Error conditions:

    - If the request is received before the verifier expects it (i.e., the agent is requesting the attestation details shortly after the last attestation without waiting the expected number of seconds), the verifier will reply with a 429 status code and the `Retry-After` HTTP header. In this case, the agent should retry after waiting the number of seconds specified by the verifier.

    - If the last attestation processed by the verifier for the given node failed verification, and the configured verification policy for the node has not since changed, the verifier will reply with a 503 status code. In this case, the agent should retry according to an exponential backoff.

3. The agent will gather the information required by the verifier (UEFI log, IMA entries and quote) and report these in a new HTTP request along with other information relevant to the quote (such as algorithms used).

4. The verifier will reply with the number of seconds the agent should wait before performing the next attestation and an indication of whether the request from agent appeared well formed according to basic validation checks. Actual processing and verification of the measurements against policy can happen asynchronously after the response is returned.

    Error conditions:

    - If the attestation received by the verifier was produced using an invalid or expired nonce, the verifier will reject the attestation with a 400 status code.

    - Once again, if the last attestation processed by the verifier for the given node failed verification, and the configured verification policy for the node has not since changed, the verifier will reply with a 503 status code.
    
    In either case, the agent should retry according to an exponential backoff.

This protocol is contrasted against the current pull protocol in the sequence diagrams which follow:
| Pull attestation protocol | Push attestation protocol |
| --- | --- |
| ![Current pull attestation protocol](https://www.plantuml.com/plantuml/proxy?cache=no&src=https://raw.githubusercontent.com/stringlytyped/keylime-push-proposal/main/diagrams/attestation_pull.iuml) | ![Proposed push attestation protocol](https://www.plantuml.com/plantuml/proxy?cache=no&src=https://raw.githubusercontent.com/stringlytyped/keylime-push-proposal/main/diagrams/attestation_push.iuml) |

_(The current protocol messages are in black while green indicates the new items proposed by this document.)_

One drawback of this approach is that the number of messages a verifier needs to process is doubled. However, this is unlikely to significantly impact performance as the most intensive operations performed by the verifier remain those related to verification of the received quotes. Any such impact should be offset by the increased opportunity for horizontal scaling presented by the push model (as it makes it easy to load balance multiple verifiers). Further optimisations of the protocol can be explored once work on the simple version presented above has been completed.

### Proposed Changes to Registration Protocol

Currently, registration serves mostly as a way for agents to report their root and subordinate identities to the registrar when they are run for the first time. Only limited checking of these identities is performed: the registrar will check that the reported AK and EK are linked by way of TPM2_ActivateCredential. All other identity verification is performed by the tenant at enrolment.

We propose that, when operating in push mode, the registrar perform these checks at registration time instead. As discussed in previous sections, this would be done using the new trust mechanisms: either the certificate trust store, the webhook mechanism, or both together. The registrar can make this determination on the fly depending on what identity information is received from the agent (e.g., when an EKcert is presented, the registrar will check it against the trust store and fall back on the webhook, if configured).

Additionally, the registration protocol should always be secured with TLS whenever Keylime is configured to operate in push mode (registration currently happens over simple HTTP).

### Proposed Changes to Enrolment Procedure

Enrolment currently is a fairly complicated protocol involving all parties (the agent, registrar, verifier and tenant) with most of the processing being performed by the tenant. We propose instead that enrolment is simplified significantly for the push model.

#### Do Not Verify the Root Identities During Enrolment

Verification of the agent's identities will be performed during registration, as stated in the last section, and the registrar will make the outcome of these decisions available through its REST API. As such, we propose that this function is not performed by the tenant for agents running in push mode.

#### Do Not Support the Payload Feature in the Push Model

As part of the current enrolment process, the user specifies a payload which is delivered to the agent and placed in a directory to be consumed by other software. The reason for this is to support the provisioning of identities to workloads running on the node (e.g., TLS certificates or long-lived shared secrets). The payload may optionally contain a script file, which is executed by the agent.

Considering the current landscape of the identity and access management space, a more modern approach to solving this problem would likely be to have Keylime report verification results to a [SPIRE](https://spiffe.io/) attestor plugin which could then handle provisioning of workload identities. This offloads issues related to revocation and suitability for cloud-native workloads.

The arbitrary nature of the payloads mechanism also raises concerns as to the attack surface of the agent and the whole Keylime system. Not only can Keylime server components query a node to report on its state but they also have the power to modify a node's state and execute arbitrary code. Enterprise users would consider this unacceptable.

As a result, we recommend that the payload feature is not implemented in the push model. This gives users the choice to opt into a more secure design which considers a stronger threat model without taking features away from existing users. And for users which do require identity provisioning alongside push support, they have the option of using SPIFFE/SPIRE.

#### The New Enrolment Procedure for Agent-Driven Attestation

With identity checks moved into the registration protocol and the payload feature removed entirely from the push model, enrolment now becomes, simply, a way for users to specify a verification policy for a node. The process is as follows:

1. The user identifies the relevant node by whatever logical identifier has been configured (e.g., UUID, hostname, EK hash, etc.) and provides a verification policy to the tenant.

2. The tenant contacts the registrar and retrieves information about the node, including the outcomes of the registrar's trust decisions.

    The tenant exits and displays an error to the user whenever:

    - the registrar replies with a 404 status indicating the node has not yet been registered; or
    - the registrar indicates that the AK could not be bound to a root identity which has been deemed trusted.

    In all other cases, the tenant proceeds with enrolment.

3. The tenant creates a new record for the node at the verifier, providing the node identifier and verification policy plus select information obtained from the registrar: the AK for the node and the agent's client certificate (if the agent is configured to use mTLS). If the verifier replies with a 200 status, the tenant exits and displays a success message to the user. Otherwise, the tenant exits with an error message.

Once the enrolment process has been completed successfully, the verifier will accept the next attestation received from the agent and verify it against the supplied policy.

As the role of the tenant in enrolment has effectively been reduced to contacting a couple simple REST endpoints, it should be easy to re-implement this if the tenant is replaced with something else (either by the Keylime project or by the end user).

### User Stories

The benefits of the proposed changes are illustrated by these two user stories:

#### Story 1

A SaaS provider wishes to monitor the integrity of virtual machines (VMs) running on the Google Cloud Platform (GCP) but firewall rules are configured to only allow specific inbound connections. The company aims to use Keylime, configured for agent-driven attestation, but when they install it, the registrar does not trust the registrations sent by the agents because of missing EKcerts (the vTPM exposed by the GCP hypervisor does not make these available through the standard interface).

To overcome this, the company writes a simple GCP Cloud Function which responds to HTTP requests and accepts a VM name as input. It then queries the GCP REST API to retrieve the EKcert for that VM, which it then sends in the response to the caller.

The company makes the following configuration changes to Keylime:

- They configure the Cloud Function's URI as a webhook to be called by the registrar.
- They add GCP's TPM CA certificates to the registrar's trust store.
- They configure the agents to use the VM name as their identifier.

After making these changes, agents are able to successfully register themselves with the registrar. Further, the company has assurance that any configured policies will be applied by the verifier to the correct VMs.

#### Story 2

A IoT company deploys equipment at customer locations which may not be well protected from physical access. Each device communicates with the company's servers over mutually authenticated TLS (mTLS), terminated at an Nginx load balancer. The devices reach the internet from LANs which the company does not control.

The company wants to detect unauthorised modification of their hardware. To achieve this, they use Keylime configured for agent-driven attestation.

Before deploying a device in the field, the company uses their internal PKI to issue each device with a set of certificates: one for the Keylime agent and one for the hardware TPM installed in the device. Both contain a unique identifier assigned to the device by the company. The certificates of the CAs which issue the agent and TPM certificates are appropriately configured as trusted by the Keylime registrar and verifier. 

When a device is installed on site, it can start sending attestations without any setup by the customer or changes to their network. The Keylime server is able to authenticate the agent and the attestations received as originating from a specific device.

### Risks and Mitigations

As this proposal introduces new protocols, including ones for authentication, there is a risk that there exists a flaw in the design of these protocols, or that the implementation deviates from the design in a way that introduces such a flaw.

To mitigate the first risk, the protocols have been designed in the context of a defined [adversarial model](https://github.com/stringlytyped/keylime-push-proposal/blob/main/supporting-materials/adversarial-model.md). Because of this effort, the push protocols include security improvements that are not present in the existing pull protocols.

The second risk will be mitigated by the test plan (below) and standard code review by the community.

## Design Details

### Mutually Authenticated TLS (mTLS) for Agent Authentication

In the current pull model, mTLS is used to authenticate agents using a certificate and private key which each agent generates on first start and is trusted automatically by the registrar. In the past, concerns have been expressed that this is not compliant with the certificate management policies in many organisations as they often require a central certificate authority&nbsp;(CA).

However, we do not wish to remove mTLS as an option for users who manage their own PKI. In such cases, the user would provision agents each with a certificate which is pre-trusted with the registrar and the verifier.

When this authentication option is chosen, the agent should bind the corresponding private key to the TPM's identity at registration time by way of TPM2_Certify,[^mtlsbind] as suggested by the earlier [draft proposal](https://github.com/keylime/enhancements/issues/60). Additionally, all endpoints which accept authentication via mTLS should check that the expected node identifier is contained within either `Subject` or `Subject Alternative Name` fields of the presented certificate to ensure that there is a binding to the identifier. The topic of identity binding is discussed in greater detail later in the earlier [Proposed Trust Mechanisms](#proposed-trust-mechanisms) section.

### Challenge–Response Protocol for Agent Authentication

When mTLS is not enabled, the verifier can use an alternate mechanism to authenticate agents (also shown in the sequence diagram to the right):

<img src="https://www.plantuml.com/plantuml/proxy?cache=no&src=https://raw.githubusercontent.com/stringlytyped/keylime-push-proposal/main/diagrams/ak-pop-authentication.iuml" alt="AK-Based Challenge–Response Authentication Protocol" align=right>

1. The agent will request a new nonce from a verifier endpoint which does not require authentication. The verifier will randomly generate a new nonce and store it alongside the node identifier provided by the agent (multiple nonces can be associated with a single node).
2. The agent will use TPM2_Certify ([TPM 2.0 Part 3, §18.2](https://www.trustedcomputinggroup.org/wp-content/uploads/TPM-Rev-2.0-Part-3-Commands-01.38.pdf#%5B%7B%22num%22%3A276%2C%22gen%22%3A0%7D%2C%7B%22name%22%3A%22XYZ%22%7D%2C69%2C734%2C0%5D)) to produce a proof of possession of the AK, based on the nonce. The AK will be the object certified and also the key used to produce the signature.
3. The agent will send the result to the verifier which will verify the signature using its record of the AKpub for the node and check that the signed data contains a valid nonce.
4. If verification of the AK possession proof is successful, the verifier will respond with a session token, which it will persist and deem valid for a configured time period.
5. The agent will use this token to authenticate subsequent requests to the verifier. If the token is still valid, the action will proceed. Otherwise, it will reply with a 401 status and the agent will repeat the challenge–response protocol to obtain a new session token.

Whenever the agent submits a valid attestation, the quote itself acts as a proof of possession of the AK. Therefore, on successful verification of a quote, the verifier may extend the validity of the session token, minimising the need to repeat the challenge–response protocol.

This mechanism is very similar to [DPoP](https://datatracker.ietf.org/doc/html/draft-ietf-oauth-dpop) which allows authentication of OAuth 2.0 clients by way of cryptographic proof. As such, it should be possible to evolve it into a standard OAuth-based solution for API authentication down the line, if desirable.

### Certificate Trust Store Mechanism

Keylime already uses certificate checking in a number of places, e.g., to verify an EKcert against trusted TPM manufacturer certificates and to authenticate TLS connections. However, the implementations are specific to each individual circumstance.

Instead, we propose that a single genericised mechanism is used across all situations in which certification verification is required. This should operate in similar fashion to the trust stores implemented by browsers and operating systems. Given (1) a certificate to check, (2) a list of intermediate certificates with indeterminate trust status and (3) a list of trusted certificates, the mechanism should deem (1) as trusted whenever a binding exists from (1) to a certificate in (3), possibly via one or more certificates in (2).

The generic nature of this mechanism allows it to be used to trust EKcerts, pre-provisioned agent TLS certificates (when using mTLS for agent authentication), IDevIDs/IAKs[^devid], or any future X.509 certificate-based identity. The user has the option of providing CA certificates, in which case leaf certificates will be trusted transitively, or individual leaf certificates, effectively whitelisting them.

However, it is important to note that this mechanism, as described above, does not necessarily provide a binding to the node's logical identifier. In the following cases, the binding can be established automatically:

- when the root identity being checked is an EKcert and the node identifier is a hash of the EK (this is already a configuration option in Keylime);
- when the root identity being checked is a pre-provisioned certificate issued to the agent and either its `Subject` or `Subject Alternative Name` field matches the node identifier; or
- when the root identity being checked is an IDevID/IAK containing a device serial number and the node identifier is set to the same serial number.

Outside of these situations, the binding of the node's root identity to its logical identifier will need be established via the webhook mechanism.

### Webhook Mechanism for Custom Trust Decisions

Currently, if no EKcert is available (for example, if the agent is running on a VM deployed in the cloud), then the tenant allows the user to specify a script to use to verify the EK using custom logic.

We propose adopting a new mechanism which serves as an evolution of, and replacement for, this script-based approach. Instead, the user will have the option of configuring a webhook which the registrar can query for a trust decision.

The webhook will not only be called in situations where an EKcert is unavailable, but rather whenever a favourable trust decision cannot be reached by the registrar. For example:

- when an EK has been provided by the agent but this is not accompanied by an EKcert;
- when an EKcert has been provided but verification against the trust store fails; or
- when an EKcert has been provided and verifies against the trust store, but the node identifier does not match the hash of the EKpub.

When the registrar issues its request to the webhook URI, it would provide all information it has about the agent: its keys and certificates and the outcomes of the checks which the registrar has already performed. For example, it may supply a JSON object similar to the following in its request:

```json
{
    "node_id": "...",
    "root_identities": ["ek"],
    "subordinate_identities": ["ak"],
    "ek": {
        "trust_status": "NOT_TRUSTED",
        "trust_details": ["EK_CERT_RECEIVED", "EK_CERT_NOT_TRUSTED", "EK_NOT_BOUND_TO_ID"],
        "ekcert": "..."
    },
    "ak": {
        "trust_status": "BOUND_TO_UNTRUSTED_ROOT",
        "trust_details": ["AK_BOUND_TO_EK"],
        "bound_root_identities": ["ek"]
    }
}
```

The webhook endpoint may reply with a list of decisions it wishes to override based on its own custom logic, e.g.:

```json
{
    "node_id": "...",
    "decisions": ["EK_CERT_TRUSTED", "EK_BOUND_TO_ID"]
}
```

It may also elect not to override any decisions, or override only a subset of decisions, and instead provide additional information to enable the registrar to reach its own new decisions. For example, consider the case in which an agent is deployed in a VM on a cloud provider and, as a result, the EKcert is unavailable and thus the EK is not trusted nor bound to the node's logical identifier, set to the hostname of the node. The web service serving the webhook endpoint could use the node identifier to retrieve the EKcert from an API provided by the cloud provider and respond to the registrar with the following:

```json
{
    "node_id": "<hostname>",
    "decisions": ["EK_BOUND_TO_ID"],
    "ek": {
        "ek_cert": "<ek_cert_from_cloud_provider>",
        "intermediate_certs": ["<cloud_provider_intermediate_cert>"]
    }
}
```

Given that the cloud provider's root CA certificate is present in the registrar's trust store, the registrar will re-evaluate its EK decision, and mark the EK as trusted.

This proposed webhook functionality can be added to the existing registration protocol while remaining backwards compatible with previous versions and is illustrated by the sequence diagram:

<p align="center">
    <img src="https://www.plantuml.com/plantuml/proxy?cache=no&src=https://raw.githubusercontent.com/stringlytyped/keylime-push-proposal/main/diagrams/webhook.iuml" alt="Future registration protocol">
</p>

_(The current protocol messages are in black while green indicates the new items proposed by this document. Items not relevant to the push model are struck out in red.)_

Note that the outgoing request to the webhook URI is performed in a non-blocking way, so the registrar can reply to an agent's registration request without waiting for a response from the outside web service. If no well-formed response is received from the web service, it should reattempt the request using an exponential backoff, similar to what the verifier currently does when a request for attestation data from an agent fails.

### Agent Lifecycle

Putting all the above recommendations together, the lifecycle of an agent operating in push mode can be described according to the following steps (also shown in the flowchart to the right):

<img src="https://www.plantuml.com/plantuml/proxy?cache=no&src=https://raw.githubusercontent.com/stringlytyped/keylime-push-proposal/main/diagrams/agent-lifecycle.iuml" alt="Flowchart of agent lifecycle" align=right>

1. The agent starts for the first time. If the agent has not been pre-configured with a specific identifier, the agent will use the configured mechanism to obtain the identifier for the node (i.e., DMI UUID, hostname, EK hash, or IDevID serial number).

2. The agent will, as is currently the case, register itself at the registrar, providing its identifier and all available root and subordinate identities of the node (e.g., an AK and an EK, with an EKcert if available).

    _**Not shown:** the registrar will process the provided keys and certificates and reach a trust decision asynchronously, either on its own, or by invoking the configured webhook._

    If the agent is configured to use mTLS for authentication, it will proceed directly to step 5. Otherwise, it continues to step 3.

3. The agent will attempt to authenticate itself to the verifier, providing its node identifier and obtaining, in response, a nonce. Then, it will use the TPM to construct a proof of possession of its AK, based on the nonce, and try to exchange it for a session token.

    If the verifier replies with a session token, the agent proceeds to step 5. Otherwise, it continues to step 4.

4. At this point, it is likely the user has not yet enrolled the agent with the verifier and provided a verification policy for the node. As such, the verifier will reply will an error and the agent will attempt authentication again from step 3 repeatedly, employing an exponential backoff.

    _**Not shown:** the user may subsequently use the tenant to enrol the node for verification. The user supplies the identifier of the node and desired verification policy and the tenant will obtain the keys for the node from the registrar. If the registrar indicates that all identity checks for the node have passed (the AK is associated with a trusted root identity that is bound to the node identifier), then the tenant sends the node identifier, AK and verification policy to the verifier. The next authentication attempt performed by the agent should succeed such that the agent obtains a valid session token._

5. The agent retrieves, from the verifier, the information needed to produce an attestation for the node (authenticating via mTLS, if configured, or using a token obtained in the previous steps). It prepares the quote, gathers logs and sends it to the verifier. The verifier will process the quote received and arrive at a decision asynchronously.

    The verifier may reject a request sent by an agent for a number of reasons:

    - (a) The authentication token is no longer valid. The agent will reattempt authentication from step 3.

    - (b) The request is invalid or the verifier is not accepting attestations. This can happen if:

        - The request is received before it is expected.
        - The attestation has been produced using an invalid nonce or a nonce that has expired.
        - The previous attestation failed verification (and the verification policy has not since changed).

        In these cases, the agent will attempt the request again, according to some backoff.

6. The agent will continue to send periodic attestations to the verifier from step 5.

Note that full separation between the registrar and verifier is maintained and no communication takes place between them.

### HTTP Proxy Support

Enterprise users often route HTTP traffic through proxies. In addition to all the other improvements in the proposal, this enhancement would also add the ability to specify an HTTP proxy URI through which the Keylime agent will proxy its requests.

### Test Plan

Unit tests will be required for the new trust mechanisms, at minimum. Additionally, new end-to-end (E2E) tests will be needed to test the integration between agents, verifiers and registrars operating in push mode.

### Upgrade / Downgrade Strategy

Agent-driven attestation will be added as an optional alternative to server-driven attestation. Each agent and verifier will be configurable to operate in either push or pull mode. When upgrading an existing installation of any of these component, the mode will be assumed to be pull and the component will continue operating in the same manner as before the upgrade.

To downgrade a Keylime component to a version prior to the introduction of agent-driven attestation, the component must be configured to operate in pull mode. In that case, the downgrade should be seamless.

Switching between push and pull mode is a separate matter. First, all components which are being switched from one mode to the other must be running a version which supports the new push mode. Then, the user will need to perform the steps below, depending on the component in question:

#### Registrar

The registrar will simultaneously support agents operating in push and pull mode, so it is not necessary to change the mode of the registrar. However, the user may need to review the settings which affect the trust decisions made by the registrar (e.g., by configuring a webhook URI, if that feature is desired).

#### Verifier

To switch a verifier's operating mode, change the appropriate configuration option and restart the verifier. After restarting, all agents which are enrolled with the verifier will be deemed to have failed verification. To restart attestation of the agents, the user will need to change the operating mode of each agent (according the directions below) and then issue a command via the tenant. This is so that the tenant can re-check the trust status of the node's identities before restarting attestation.

#### Agent

To switch an agent's operating mode, again, simply change the agent's configuration and restart the agent. After restarting, the agent will attempt to update its registration at the registrar. If this succeeds and the agent's new operating mode is push, the agent will start sending attestations to the verifier specified in the agent's configuration file. Attestations will be rejected by the verifier until attestations are restarted by issuing the tenant command described above.

### Dependency Requirements

No additional dependencies are foreseen at this time.

## Drawbacks

This proposal for agent-driven attestation is not simply the old server-driven approach with the directionality of the protocols reversed. As such, existing users who wish to switch from server-driven attestation to agent-driven attestation will need to take into account the following:

- If they use Keylime to deliver payloads to attested nodes, they will need an alternate way of achieving this (e.g., [SPIFFE/SPIRE](https://spiffe.io/)).
- They will need a way of binding EKs to logical node identifiers (likely via the webhook mechanism), use a hash of the EK as the node identifier, or switch to using an alternate root identity which includes the binding.
- Any scripts for making trust decisions about EKs will need to be replaced with an equivalent webhook implementation.
- All agents and verifiers will need to be updated to a version with agent-driven attestation support at the same time, or alternatively, the user will need to incrementally migrate each agent from a verifier operating in pull mode to a verifier operating in push mode.

In return, the user gains a robustly secure design which has been considered against a defined [adversarial model](https://github.com/stringlytyped/keylime-push-proposal/blob/main/supporting-materials/adversarial-model.md).

## Alternatives

Many different approaches to agent-driven attestation are possible. Some of the variants previously considered included aspects such as:

- retaining the payloads feature
- requiring mTLS for all users
- performing part of the enrolment of an agent with the verifier automatically
- using a shared secret to derive the nonce used in TPM quotes
- retaining the old trust model
- no support for IDevIDs/IAKs or other identity types beyond EKs/AKs

We have arrived at the current proposal which addresses each of these points after extensive community consultation.

## Acknowledgements

Many thanks to Thore Sommer (@THS-on) for sharing his ideas, both face to face and in his previous [draft proposal](https://github.com/keylime/enhancements/issues/60), to Marcus Heese (@mheese) for many helpful discussions around threat model and operational concerns and to everyone else who has commented on the various iterations of this proposal. We also greatly appreciate the feedback and guidance received from the maintainers and community members in the [June](https://github.com/keylime/meetings/issues/66), [July](https://github.com/keylime/meetings/issues/67) and [August](https://github.com/keylime/meetings/issues/68) community meetings.

<!-- No infrastructure needed, so section dropped -->

<br><br>

**Footnotes**

[^devid]: IDevIDs/IAKs are a type of cryptographic identity issued by a device manufacturer and tied to the TPM of the device. For further discussion of device identity as it relates to Keylime agent-driven attestation, see the following supplement to this proposal: [_802.1AR Secure Device Identity and Agent-Driven Attestation_](https://github.com/stringlytyped/keylime-push-proposal/blob/main/supporting-materials/devids-and-push-model.md).

[^mtlsbind]: TPM2_Certify, described in [TPM 2.0 Part 3, §18.2](https://www.trustedcomputinggroup.org/wp-content/uploads/TPM-Rev-2.0-Part-3-Commands-01.38.pdf#%5B%7B%22num%22%3A276%2C%22gen%22%3A0%7D%2C%7B%22name%22%3A%22XYZ%22%7D%2C69%2C734%2C0%5D), does not, on its own, prevent all attacks on the binding between the EK and the agent's client certificate used for mTLS. An attacker who obtains the secret key associated with the certificate (possible in our [adversarial model](https://github.com/stringlytyped/keylime-push-proposal/blob/main/supporting-materials/adversarial-model.md)) could block the legitimate node's registration messages from reaching the registrar. Then, the attacker could register their own node instead, causing the certificate to be associated with a TPM under the attacker's control. Because of this, the CA should perform its own binding during certificate issuance and include the EK hash in the certificate.