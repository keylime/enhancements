<!-- **Note:** When your enhancement is complete, all of these comment blocks
should be removed.

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
# enhancement-#73: Durable (Offline) Attestation support in Keylime

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

The goal of this enhancement is to provide Keylime with the ability to store 
all the required information to perform a full attestation, in a persistent 
external time-series like datastore.
This should also include some proof that a given AK (still referred as "AIK"
inside Keylime) created on a TPM by an `agent` was indeed tied to a given EK
 (from the TPM located at the node where it was running). This AK/EK 
association process is done by the `registrar`, and it will be its 
responsibility to store a record of such association on a tamper-resistant 
metadatastore (e.g. transparency log).

The main reason for two different kinds of stores is the unequal nature of 
its contents. While the time-series like datastore will hold actual objects
(in our case, JSON contents), the tamper-resistant metadastore will hold 
records of signatures of such objects.

## Motivation

<!--
This section is for explicitly listing the motivation, goals and non-goals of
this enhancement.  Describe why the change is important and the benefits to users.
-->

The main motivation for adding this functionality is to give auditors and 
other compliance officers the ability to answer, with a proper degree of 
certainty and trust, the following question: did node N have its software 
stack fully attested at date D? Being date "D" a point in time that could 
be well in the past, we cannot rely on the accessibility to (or even the 
existence of) the given node. Furthermore we cannot even rely on the 
accessibility to (or even the existence of) the server-side components 
of the Keylime cluster (i.e., `registrar` and `verifier`) and thus need 
to design with these boundary conditions in mind.

### Goals

<!--
List the specific goals of the enhancement.  What is it trying to achieve?  How will we
know that this has succeeded?
-->

- Add functionality on the `registrar` to record (in a tamper-resistant
  transparency log) the association between the EK and AK (i.e.
`tpm2_makecredential`).
- Add functionality on the `registrar` to record (in a time-series like 
persistent datastore) all the information required to check the association
 of EK and AK.
- Add functionality on the `verifier` to record (in a time-series like 
persistent datastore) all the information needed to perform attestation
 standlone and offline (i.e., quotes and MB/IMA logs).
- Add functionality on the `verifier` to **optionally** record (in a 
tamper-resistant transparency log) a hash/signature of the contents 
of the information needed to perform attestation standalone and offline.
- Add functionality, on both `verifer` and `registrar` to **optionally**
request a timestamp from an external Time Stamp Authority whenever storing
data on the tamper-resistant transparency log.
- Add a new CLI which will interface with the aforementioned persistent 
stores, and will call the main, unmodified `verifier` code in order to do
 post-facto attestation.  

### Non-Goals

<!--
What is out of scope for this enhancement?  Listing non-goals helps to focus discussion
and make progress.
-->

- The interaction between the time-series like persistent datastore and
  tamper-resistant transparency log will be done by keylime user/operator.
Inside the core Keylime, a "plugin" architecture will be adopted (similar
to the "policies" for `Measured Boot`) and the implementation details of the
code which will interact with such stores are outside of the scope of this
 enhancement. Nevertheless, some relevant examples of interaction with some
backends (e.g., Redis, Rekor, "Flat file") will be provided "as is".


## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation.  The "Design Details" section below is for the real
nitty-gritty.
-->

- The `registrar` will be modified to, upon initial `agent` registration -
  which includes the execution of `tpm2_makecredential` - record the EK, AK
into a JSON file, sign it (using the private key generated as part of the
certificates for mTLS interaction with both `tenant` and `registrar`) and then
make a record of it on a tamper-resistant transparency log (e.g., Rekor). In
addition to that, it will store the JSON file, the signature, and the public
key on the time-series like persistent datastore. This should allow an external
component/user to check, provided that there is trust on the `registrar`, that
a particular AK is indeed tied to a particular EK. The reason for having this
data stored into a time-series is due to the fact that AKs are regenereated
every time an `agent` is restarted on Keylime.
- The `verifier` will be modified to take the `json_response` (python
  dictionary) from the `agent` - which will include both TPM quotes and logs (MB
and IMA) - plus `agent` data (python dictionary which includes all columns stored)
from the SQL database, internal to Keylime), combine it into a single (python 
dictionary serialized into JSON contents) record and store it on a 
time-series like datastore.
- The `verifier` will also be modified to extract the "TPM clock information"
 (i.e, "clock", "resetCount", "restartCount", "safe") from the quote 
(part of `json_response`) and make it available on this same python
dictionary. The `verifier` will then use this information to compare this
 timestamp with the "previous" one (in the case of "online" Keylime 
attestation, the one stored on the SQL database, as part of the `agent` data
 python dictionary) in order to decide if a quote is being replayed. This 
mechanism is not strictly necessary for the online attestation (given trust
in the `verifier`), but will become crucial for offline attestation, where 
entries on the time-series like persistent datastore could be modified 
by an attacker. 
- Three new parameters: the name of a python module to be dynamically
  imported (which will contain code used to interact with these new proposed
stores) and the URLs for these these new proposed stores will be
supplied by the user as parameters under `[cloud_verifier]` and 
`[registrar]` sections : `durable_attestation_import`, 
`persistent_store_url` and `transparency_log_url`. The URL format is
 similar to the one already used to establish connectivity to SQL 
databases within Keylime.
- First additional tunable: an user can specify, via an attribute on 
`keylime.conf` (`[cloud_verifier]` section), additional attributes of the
 `agent` python dictionary to be signed by the `verifier`, being this 
signing also stored on tamper-resistant transparency log. The value of 
this attribute - `signed_attributes` - could be set to `all`, which will 
result in the `verifier` signing **everything**, including the 
`json_response` data package from the `agent`, and storing a record on the 
transparency log. As it is the case with `registrar`, in this context
"signing" means "dump the whole dictionary into a JSON file, sign it
(using `openssl`) and record an entry on the tamper-resistant transparency
log".
- Second additional tunable: an user can specifiy, via an attribute on
 `keylime.conf` (under `[cloud_verifier]` and `[registrar]` sections), 
a Time Stamp Authority URL (`time_stamp_authority_url`), and any 
signature done by either the `registrar` or the `verfier` will be 
timestamped (being the timestamp request also stored on the 
time-series like persistent store).
- A new API is proposed with the following operations: *bulk_record_retrieval*,
*record_read*, *record_create*. The *bulk_record_retrieval* and *record_create* 
calls will be the only ones which will interact with different datastores, in a
"plugin" architecture. Three additional "internal" (implicit) calls are also
defined: *record_sign*, *record_timestamp* and *record_check*. The first
two are called within *record_create* and the latter is called from
within *record_read*. 
- A new CLI interface - `keylime_attest` - will contact both the transparency
  log and the time-series like datastore, get a list of AKs proven to be associated
with an EK, and then call the same code used by the `verifier` (i.e.,
`cloud_verifier_common.process_quote_response`) to perform a series of point in
time attestation on all records retrieved from the persistent datastore.

### Notes/Constraints/Caveats (optional)

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above.
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->

- The `keylime_attest` CLI will call the attestation code used by the
  `verifier` without any modification, and should be up to the user to write a
more complex policy, or even an independent replacement code, if he choses to do so.

### Risks and Mitigations

<!--
What are the risks of this proposal and how do we mitigate.  Think broadly.
For example, consider both security and how this will impact the larger
enhancement ecosystem.

How will security be reviewed and by whom?
-->

- The first key security aspect here is to convince ourselves (and others) that the
  record generated by the `registrar` to indicate the association between EK
and AK is enough. Once this is done, offline attestation has basically the
same level of security of the online attestation (which was already evaluated)
as it uses the very same code base.
- The second key security aspect is the level of trust one needs to have on the
time-series like persistent datastore. Provided that the offline attestation can
have access to TPM timing information from within a quote, it is possible
to provide garantees that an attacker cannot alter the information on the
datastore in manner that causes a false positive for attestation. This can be
established without any need for trust on the `verifier` itself.
- If an user decides that trust on a `verifier` can be established post-facto for
all involved parties (e.g, Keylime administrators and third-party compliance 
officers) then there is the **optional** possibility having every attestation
data sample (which will include TPM quotes and MB/IMA logs, among other info) signed
and recorded on the tamper-resistant transparency log.
- Furthermore, it should be possible to store **all** the data (`agent` and 
`json_response`) directly on the transparency log (e.g., Rekor allows the creation
of new "rich types" that can store complex data). The current enhancement is 
agnostic about this possibility and it should be simple to create an "adapter"
which does not require a time-series like datastore.
- Finally, while we do expect very little impact on KeyLime's scalability by adding this
  capability, it is important to remember that we are constantly testing KL in
a configuration with 5K nodes (with both MB and IMA simultaneously activated),
and can provide experimental evidence to back this hypothesis.

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable.  This may include API specs (though not always
required) or even code snippets.  If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->

- The first PR will provide the "persistent datastore" plugin capability, to be
  called from with both `registrar` and `verifier` code. It will include a
default, "null operation" and all the required changes into `config.py` and
`keylime.conf`. The plan is to use the `[general]` section add a four new
entries `durable_attestation_import`, `persistent_store_url`, 
`transparency_log_url` and `time_stamp_authority_url`, where the URLs will
assume a format similar to the already used for database access.
- A second PR will give the `verifier` the ability to extract and store "TPM
  clock information". This might include changes on the database schema.
- A third PR will provide a CLI utility to perform offline attestation
- A fourth PR will provide some example for interaction with a few select 
backends (e.g., Redis + Rekor, Flat files).

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

- The default "null operation" plugin for the persistent datastore will allow
  the base keylime code to be continuosly tested as it is today.
- Given that we are not mandating any kind of specific persistent store,
  neither for the time-series like datastore nor for the tamper-resistant
transparency log, there are no plans to perform any continous testing on it.

### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this enhancement
-->

- This is an optional feature, and thoroughly backward compatible with current
  Keylime deployments.

## Drawbacks

<!--
Why should this enhancement _not_ be implemented?
-->

- No known drawbacks.

## Alternatives

<!--
What other approaches did you consider and why did you rule them out?  These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->

- No known alternatives.

## Infrastructure Needed (optional)

<!--
Use this section if you need things infrastructure related specific to your enhancement.  Examples include a
new subproject, repos requested, github webhook, changes to CI (travis).
-->

- Some sort of external time-series like datastore and tamper-resistant transparency
  log will be needed in order to enable this feature. 
