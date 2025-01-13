# enhancement-1552: EK Certificate Chain support

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

- [ ] Enhancement issue in release milestone, which links to pull request in [keylime/enhancements]
- [ ] Core members have approved the issue with the label `implementable`
- [ ] Design details are appropriately documented
- [ ] Test plan is in place
- [ ] User-facing documentation has been created in [keylime/keylime-docs]

## Summary

**Issues:**
* rust-keylime, [#859](https://github.com/keylime/rust-keylime/issues/859)
* keylime, [#1552](https://github.com/keylime/keylime/issues/1552)

Add support for EK certificates that are signed by intermediate certificates.
The current setup can only verify EK certificates that are signed by a certificate that
was added to the `tpm_cert_store`. The current setup does not support the verification of
a chain.

New [intel chips](https://community.intel.com/t5/Processors/How-to-verify-an-Intel-PTT-endorsement-key-certificate/td-p/1603153/page/2) (starting 11th core gen) are certified using the intel ODCA (On Die Certificate Authority). The certificate chain is as follows:

| tpm_cert_store | TPM | TPM | TPM |
|:--------------:|:---:|:---:|:---:|
|Root CA | Intermediate | Intermediate | EK |
|On Die CSME P_MCC 00001881 Issuing CA| CSME MCC ROM CA | CSME MCC SVN01 Kernel CA | CSME MCC PTT  01SVN |

This is also defined by the [TCG EK specification](https://trustedcomputinggroup.org/wp-content/uploads/TCG-EK-Credential-Profile-V-2.5-R2_published.pdf) (Chapter 2.2.1.5.2)

The current setup only allows to verify the EK cert against the `tpm_cert_store`.
As the EK certificate is signed by intermediate certificates, which are unique to device,
it is not possible to verify the EK certificate against the `tpm_cert_store`. The EK
must be verified against the EK Certificate Chain, till the top level certificate
from the chain can be verified against the `tpm_cert_store`.

## Motivation

Add support for intel firmware TPMs and possibly other TPMs that provide an EK certificate chain,
or setups that also depend on a certificate chain that must work offline or without any other
access to the chain.

### Goals

* Verify EK certificates that are not signed directly by a root ca, but require the verification of a chain.

### Non-Goals

* Implementing a generic interface to provide the content of any NV-Index.

## Proposal

* keylime_agent, send `ek_ca_chain` to registrar
* keylime registrar, store `ek_ca_chain` in database
* keylime tenant, verify `ekcert` against `ek_ca_chain` and `ek_ca_chain` against `tpm_cert_store`


### User Stories

#### Story 1
Users will be able to use keylime with intel firmware TPMs (11th gen core and onwards),
or any other TPM that stores an EK certificate that is signed by a chain that is stored
in the TPM.


### Risks and Mitigations

#### Registrar/Tenant could be become incompatible with older database
* Update database to new scheme, only a single key is added to the registar db 'ek_ca_chain'

#### Registrar/Tenant could become incompatible with older Agent
* Make 'ek_ca_chain' optional

#### Additional memory will be required to store the chain in the database.
* If the feature can't be used, due to missing certificates in the TPM, the memory footprint will stay around the same.

#### Providing big chains as attack
* Limit the amount of allowed chain size
* Use mTLS to only allow verified clients access


## Design Details
First some words from TCG EK documentations:

```txt
TPM NV MAY contain all certificates of the EK Certificate Chain except the Root CA certificate. The
EK Certificate Chain MUST be stored as X.509 DER encoded certificates. If the chain consists of
more than one certificate, or if multiple chains exist, they MUST be stored in NV as a concatenated
sequence. The TPM manufacturer MAY provision certificate chains using the following list of indices:

0x01c00100 - 0x01c001ff

Revision 2 January 26, 2022 Published Page 15 of 60
The NV indices MUST be populated starting with index 0x01c00100 through index 0x01c001ff. There
is no requirement to store the certificates in any particular order whatsoever. If a concatenated
certificate chain does not fit in a single index, the chain MUST overflow to the next numerically larger
index in the list of NV Indices. If the storage space in a single index is insufficient to store the entire
certificate, the certificate MAY overflow into the next numerically larger index in the list of NV Indices.
```

The feature requires changes in the rust-keylime agent and in the keylime servers registrar and tenant. The certificate chain must and can be only provided by the keylime-agent.

The keylime-agent must do the following:
* get all available NV-Index-Handles in the defined range (`0X01C00100 - 0X01C001FF`)
* read the chain from the NV-Index-Handles
  * Theoretically the certificates can be in any order. This would require an option to make a sorting possible in the agent. As it does not make really sense to store the chain in an unsorted order i would postpone any implementation till this case really exists.
  * Only a subset of the NV-Index-Handles may be required. This would require an option to filter/set the required handles. I would also wait for implementation till a real case exists for that.
* as the certificate chain is stored in DER format, first the certificates must be separated
* transform der DER certificates into PEM format.
* send the chain to the keylime server

The certificate chain should be transformed and sorted by the keylime-agent to keep the work
away from the server and provide a generic server implementation that does not need to behave
a certain way (sort/filter the chain) depending on the client.

The keylime registrar only needs to store the certificate chain in the database.

The keylime tenant must detect the presence of an EK Certificate Chain and verify the
EK Cert against the chain, and finally the top level certificate against the `tpm_cert_store`.

The flow can be kept mostly as it is in `check_ek`. In case of a present ek_ca_chain, the ek must be verified against the provided chain and in case of a success the ek will be replaced by the top level certificate from the chain. Afterwards the flow can be kept as it was.

See provided implementation:
* rust-keylime: https://github.com/ematery/rust-keylime/commit/ff448ec9f68a50b89685f8f4f3e6777d8c80ef1b
* keylime: https://github.com/ematery/keylime/commit/772f6808510d797d7cecd3347215fa835a583868

### Test Plan

### Upgrade / Downgrade Strategy

### Dependencie requirements

## Drawbacks

## Alternatives
