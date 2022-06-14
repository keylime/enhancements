<!--
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
# enhancement-72: Move configuration to TOML and Simplify TLS Setup

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

- [ ] Enhancement issue in release milestone, which links to pull request in
  [keylime/enhancements]
- [ ] Core members have approved the issue with the label `implementable`
- [ ] Design details are appropriately documented
- [ ] Test plan is in place
- [ ] User-facing documentation has been created in [keylime/keylime-docs]

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

## Summary

The `keylime.conf` is cleaned up and split into different sections.
The new configuration format is versioned to allow for simpler upgrades in the
future.

The current TLS setup uses one CA for the verifier, one for the registrar and
one for the agent mTLS authentication. This will be replaced by a client and
server (key and certificate) option in each component (agent, verifier,
registrar, tenant). The client pair is used to authenticate the component using
mTLS and the server pair is used for TLS server (like the registrar, agent and
verifier have). Also two new options are introduced, one for trustworthy
incoming CAs for server side mTLS authentication and another one for trustworthy
CAs from the client side.

## Motivation

Keylime uses the Python specific INI inspired configuration format and
mixes different data types (comma separated list vs. Python syntax).
Furthermore the configuration for different components are not separated
and there is no nice way to upgrade users to newly introduced configuration
options.

The current way TLS is setup in Keylime is confusing to newcomers. Also the
different configuration options are not consistent in their naming scheme.


### Goals
* Split the config into five new configuration files: global, agent, tenant,
  registrar, verifier
* Add versioning to the config
* Provide a migration tool to port common Keylime configurations to the new
  format
* Simplify the TLS setup of Keylime
* Cleanup keylime_ca to make it work with the new setup
* Formally allow the Rust agent configuration to diverge from the Python agent

### Non-Goals

* Implement other mechanisms of authentication other than mTLS
* Move registration to HTTPS (but can be easily done in a followup change)
* Add a migration path for all possible old Keylime configurations

## Proposal - Split the global configuration into the different components and Cleanup

Distributions should ship the default configuration in `/usr/etc/keylime/`
or if not supported in `/etc/keylime/`. If `/etc/keylime/` is populated the
options in `/usr/etc/keylime` will be ignored. Manual overrides can be
provided in `/etc/keylime.d/`. The Rust agent will adopt another
configuration format and can diverge from the Python agent options (not all
old options are useful for the rust agent).

For the new configuration is split into the following files with the following
options:

* `global.conf`: Contains global configuration.
  * `enable_tls`: Enable or disable TLS for the servers (registrar, verifier) and
    for the tenant if it tries to use TLS. (This setting is currently broken, so
    we either fix it or fully remove it)
  * Logging configuration
  * `version`: Version number of the entire configuration.
* `agent.conf`: Contains the agent specific configuration.
  * `ip`:  IP where the agent listens on. (Should this be a list?)
  * `port`: Port where the agent listens on
  * `contact_ip` (optional): On which IP can the verifier/tenant contact the agent 
  * `contact_port` (optional): On which port can the verifier/tenant contact the agent 
  * `registrar_ip`: IP of the registrar
  * `registrar_port`: Port of the registrar
  * `enable_agent_mtls`:  See TLS simplification
  * `tls_dir`: See TLS simplification
  * `server_key`: See TLS simplification
  * `server_cert`: See TLS simplification
  * `trusted_client_ca`: See TLS simplification
  * `trusted_server_ca`: See TLS simplification
  * `enc_keyname`: Derived key K from U/V split for payload
  * `dec_payload_file`: Name of the decrypted payload
  * `secure_size`: Size of the tmpfs. Note hat in most cases this is provided by
    the distribution already.
  * `tpm_ownerpassword`: (Only Python agent)  TPM owner password (We do not
    really need taking ownership for remote attestation). Will be probably
    removed in the future.
  * `extract_payload_zip`: Try to extract the payload
  * `uuid`: UUID of the agent or one of the following options
    * `generate`: creates random UUID
    * `hash_ek`: Hash of the EK_pub in PEM format
    * `hostname`: The hostname of the system
    * `environment`: Uses the environment variable "KEYLIME_AGENT_UUID" as UUID.
    * `dmidecode`: system UUID (Can we deprecate this option?)
  * `enable_revocation_notifications`: Option to enable or disable listening for
    revocation notifications from the verifier.
  * `revocation_notifier_ip`: IP of the revocation notifier service
  * `revocation_notifier_port`: Port of the revocation notifier service
  * `revocation_cert`: Place of the certificate to check revocation messages against
  * `revocation_actions`: List of revocations to run (Finally deprecate the Python only actions)
  * `payload_script`: Name of the script to run
  * `enable_insecure_payload`: Enable payloads even if mTLS is disabled
  * `measure_payload_pcr`: Measure the payload into a PCR (With IMA is there any need for this feature anymore?)
  * `exponential_backoff`: Use exponential backoff for retries
  * `retry_interval`: Time to retry again or base for exponential retry if enabled
  * `max_retries`: Maximum number of retries
  * `tpm_hash_alg`: Hash algorithm used (e.g. for PCRs): sha1, sha256, sha384, sha512
  * `tpm_encryption_alg`:  Used encryption algorithm: ecc, rsa (Has anyone tested actually ecc?)
  * `tpm_signing_alg`: Signing algorithms
  * `ek_handle`: Use persisted EK handle (Do we need this option?)
  * `run_as`: Drop privileges to a different user after start
* `verifier.conf`: Configuration of the verifier
  * `ip`:  IP where the verifier listens on. (Should this be a list?)
  * `port`: Port where the verifier listens on
  * `enable_agent_mtls`:  See TLS simplification
  * `tls_dir`: See TLS simplification
  * `server_key`: See TLS simplification
  * `server_cert`: See TLS simplification
  * `trusted_client_ca`: See TLS simplification
  * `trusted_server_ca`: See TLS simplification
  * `database_url`: Database configuration
  * `database_pool_sz_ovfl`: SQAlchemy setting
  * `auto_migrate_db`: Automatically update DB schema using alembic
  * `num_workers`: Number of processes (workers) to spawn
  * `exponential_backoff`: Use exponential backoff for retries
  * `retry_interval`: Time to retry again or base for exponential retry if enabled
  * `max_retries`: Maximum number of retries
  * `quote_interval`: polling interval for getting quotes from the agent
  * `revocations`: Section for revocation related configuration
    * `enabled_revocation_notifications`: List of enabled methods for sending revocation notifications
    * `zmq_ip`: Listing IP of the revocation notifier
    * `zmq_port`: Listing port of the revocation notifier
    * `webhook_url`: URL to send notifications to
  * `max_upload_size`: Upload size for policies to the verifier
  * `measured_boot_policy_name`: Name of the policy that is used for measured
    boot
  * `severity_labels`: List of used severity levels
  * `severity_policy`: List of regexes that match a event ID to a severity label
  * `ignore_tomtou_errors`: If "time of measure, time of use" (ToMToU) errors
    should be treated as a failure. (Should this part of the IMA policy instead
    the Keylime configuration?)
* `tenant.conf`: C
  * `verifier_ip`: IP of the verifier
  * `verifier_port`: Port of the verifier
  * `registrar_ip`: IP of the registrar
  * `registrar_port`: Port of the registrar
  * `tls_dir`: See TLS simplification
  * `enable_agent_mtls`:  See TLS simplification
  * `client_key`: See TLS simplification
  * `client_cert`: See TLS simplification
  * `trusted_server_ca`: See TLS simplification
  * `tpm_cert_store`: Location of the directory with the trusted EK CA certificates
  * `max_payload_size`: Maximal size of the payload to send. Should match tmpfs size for the agent secure mount.
  * `accept_tpm_hash_algs`: List of allowed hash algorithms
    * Default: sha512,sha384,sha256,sha1
  * `accept_tpm_encryption_algs`: List of accepted encryption algorithms. (Note: some parts of Keylime only support RSA)
    * Default: ecc, rsa
  * `accept_tpm_signing_algs`: List of supported signing algorithms
    * Default: ecschnorr,rsassa
  * `exponential_backoff`: Use exponential backoff for retries
  * `retry_interval`: Time to retry again or base for exponential retry if enabled
  * `max_retries`: Maximum number of retries
  * `require_ek_cert`: Require EK certificate validation
  * `ek_check_script`: Path to a script for validating the EK certificate. The following environment variables are set:
    * EK: PEM encoded version of the EK
    * EK_CERT: DER encoded EK certificate if available
* `registrar.conf`: Registrar specific configurations
  * `registrar_ip`: Listing IP of the registrar
  * `registrar_port`: Listing port of the registrar (agents)
  * `registrar_tls_port` Listing port of the registrar (server side)
  * `tls_dir`: See TLS simplification
  * `server_key`: See TLS simplification
  * `server_cert`: See TLS simplification
  * `trusted_client_ca`: See TLS simplification
  * `database_url`: Database configuration
  * `database_pool_sz_ovfl`: SQAlchemy setting
  * `auto_migrate_db`: Automatically update DB schema using alembic
* `ca.conf`: Configuration for the CA
  * `cert_country`
  * `cert_ca_name`
  * `cert_state`
  * `cert_locality`
  * `cert_organization`
  * `cert_org_unit`
  * `cert_ca_lifetime`
  * `cert_lifetime`
  * `cert_bits`
  * `cert_crl_dist`
* `webapp.conf`: WebApp specific configuration. TODO: do we want to deprecate the webapp?
  * `webapp_ip`
  * `webapp_port`
  * `populate_agents_interval`
  * `update_agents_interval`
  * `update_terminal_interval`
* `logging.conf`: Global logging configuration. Same as in the currently 


### Unification of data types
Because we the configuration is for Python we use the default Python types.

* Booleans: `True` and `False`
* Strings: Not escaped or  escaped with `"` e.g. `test = test.txt` or `test = "test with space.txt"`
* Lists: Defined as in Python e.g. `[1, 2, "string"]`
* Dicts: Defined as in Python e.g. `{"key": "value", "key2": 1}`

We will provide a `getlist` and `getdict` that are converting those entries to
the Python structures using `ast.literal_eval`.

### Removal of the following options
* `tls_check_hostnames`: Only in webapp code, but unused. 
* Some database options, because URLs are the most flexible format for SQLAlchemy:
  * `database_drivername`
  * `database_username`
  * `database_password`
  * `database_host`
  * `database_name`
* Keeping default fallbacks for the tenant in the config is mostly unused.
  Therefore we remove the options.
  * `tpm_policy`
  * `ima_allowlist`
  * `ima_excludelist`


## Proposal - TLS setup simplification
To the TLS related options in Keylime we will make the following changes.

### New options
The old options are replaced with the following new ones:

* `tls_dir`: Path to where the files are located
* `client_key`: Key for mTLS client connections
* `client_cert`: Certificate for mTLS client connections
* `server_key`: Key for a TLS server
* `server_cert`: Certificate for a TLS server
* `trusted_client_ca`: List of trusted CAs for client certificates
* `trusted_server_ca`: List of trusted CAs for server certificates

### Renamed options
* `agent_mtls_cert_enabled`: Is renamed to `enable_agent_mtls`.
* `mtls_cert_enabled`: Is renamed to `enable_agent_mtls`.

### Old and removed options
* `tls_dir`: Is still the same.
* `ca_cert`: Removed. Add CA to `trusted_client_ca` instead
* `my_cert`: Removed. Use `server_cert` instead in the verifier and `client_cert`
  instead in the tenant.
* `private_key`: Removed. Use `server_key` instead in the verifier and
  `client_key` instead the in tenant.
* `private_key_pw`: Removed. We will no longer support keys with passwords.
* `check_client_cert`: Removed. Use the `all` keyword in `trusted_client_ca`
  instead.
* `agent_mtls_cert_enabled`: Is renamed to `enable_agent_mtls`.
* `agent_mtls_cert`: Removed. Add the CA of `client_cert` to `trusted_client_ca` of
  the agent.
* `agent_mtls_private_key`: Removed. The `client_key` is used instead.
* `agent_mtls_private_key_pw`: Removed.
* `registrar_tls_dir`: Removed. There is only `tls_dir`.
* `registrar_ca_cert`: Removed. Add the CA used by the registrar to
  `trusted_ca_out` instead.
* `registrar_my_cert`: Removed. Use `client_cert` and add the CA of the
  tenant/verifier to `trusted_client_ca` in the registrar.
* `registrar_private_key` Removed. Use `client_key` instead.
* `registrar_private_key_pw`: Removed.
* `rsa_keyname`: (Agent only) Removed. Use `server_key` instead.
* `mtls_cert`: (Agent only) Removed. Use `server_cert` instead.
* `keylime_ca`: (Agent only) Removed. Use `trusted_client_ca` instead. 
* `mtls_cert_enabled`: Renamed to `enable_agent_mtls`.

### Special keywords
* `default`: If an option is set to default it will use the default of the
  verifier as before
  * `tls_dir`: `/var/lib/keylime/cv_ca` for server components. `/var/lib/keylime/secure` for the agent.
  * `client_key`: `private.key` (Use generic name instead of FQDN which is more
    portable)
  * `client_cert`: `cert.pem` (PEM encoded)
  * `server_key`: `private.key`
  * `server_cert`: `cert.pem`
  * `trusted_client_ca`: `[cacert.pem]`
  * `trusted_server_ca`: `[cacert.pem]`
* `all` (only in `trusted_client_ca` and `trusted_server_ca`): Disables validation of
  the certificates.
* `generate` (only in `tls_dir`): Generates a CA with the correct certificates.
  Agent will generate the key and certificate if not present regardless of this option.


### Component specific details
* Agent: Will not have the `client_key` or `client_cert` option, because it does not connect anywhere with mTLS.
* Tenant: The `server_key` and `server_cert` option will be only used by the tenant webapp.

### User Stories (optional)

<!--
Detail the things that people will be able to do if this enhancement is implemented.
Include as much detail as possible so that people can understand the "how" of
the system.  The goal here is to make this feel real for users without getting
bogged down.
-->

#### Story 1

#### Story 2

### Notes/Constraints/Caveats (optional)

The TLS setup is moved from a connection basis to a component basis.

### Risks and Mitigations

Setting up the client TLS verification is important for the overall security of
Keylime. Warning for obvious insecure settings should be added where not already
in place. The documentation will describe the default configuration and one more
complex setup as a reference.


## Design Details

TBD

### Test Plan

The current e2e tests will be changed to use the new TLS setup.

### Upgrade / Downgrade Strategy

For the most common setups we provide a migration script to the new config
format. If only an old `keylime.conf` is found Keylime will error with a message
that migration to the new format is required.

Automatic downgrading will not be supported. The user can just use their old `keylime.conf`.


### Dependency requirements



## Drawbacks

* Configuration must be migrated. We will not support the old and the new format
  simultaneously.
* The CAs are now component wise, not based on who the connection goes to. So
  instead of having one CA for verifier, one for registrar on the server side
  and for the agent on the client side, there is a (potential) CA for the
  verifier, one for the registrar, one for the tenant used for there client and
  server certificates. For trusting client certificates a list of trusted CAs
  can be specified.

## Alternatives

* Keep the current TLS setup