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
# enhancement-72: Split configuration file and simplify TLS Setup

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

The configuration options present in the old `keylime.conf` are cleaned up and
split into individual files corresponding to each of the sections.

The TLS configuration options are simplified to make the naming more consistent
and intuitive.

The new configuration format is versioned to allow for simpler upgrades in the
future.

## Motivation

The Keylime configuration for different components are all in one single file
and there is no versioning to control newly introduced configuration options and
possibly implementing upgrading mechanisms for the configuration files.

The current way TLS is setup in Keylime is confusing, with inconsistent naming
scheme, and complex CA certificates configuration options.

Keylime uses the Python specific INI inspired configuration format and
mixes different data types (comma separated list vs. Python syntax).

### Goals
* Split the `keylime.conf` configuration file into files: `agent.conf`,
  `tenant.conf`, `registrar.conf`, `verifier.conf`, `ca.conf`, and
  `logging.conf`
* Add versioning to the configuration files
* Provide a migration tool to port current single Keylime configurations to the
  new split format
* Simplify the TLS setup of Keylime
* Cleanup keylime_ca to make it work with the new setup
* Formally allow the Rust agent configuration to diverge from the Python agent
  configuration

### Non-Goals

* Implement other mechanisms of authentication other than mTLS
* Move registration to HTTPS
* Add a migration path for all possible old Keylime configurations

## Proposal - TLS setup simplification

The current Keylime TLS setup allows setting the CA certificate to trust, the
private key, and certificate to be used when accessing each different component
as a client.

In the new configuration, components that access multiple servers use a single
key pair and certificate to access all servers. This reduces the flexibility,
but also reduces the complexity of the TLS configuration.

Each component has options to set a private key and certificate to be used for
each performed role (client and server). Note that not all components perform
both roles (client and server).

The new configuration introduces new options to set the trusted CA
certificates. Each component that connects to servers gets the
`trusted_server_ca` option to set the list of trusted server CA certificates,
and each component that acts as a server gets the `trusted_client_ca` to set the
list of trusted client CA certificates.

The old TLS configuration options are replaced with the following:

* `tls_dir`: Path to the directory where key and certificate files are located
* `enable_agent_mtls`: Enable the agent TLS with mutual authentication
* `client_key`: Private key used for TLS client side authentication
* `client_cert`: Certificate used for TLS client side authentication
* `server_key`: Private key used for TLS server side authentication
* `server_cert`: Certificate used for TLS server side authentication
* `trusted_client_ca`: List of trusted CAs for client certificates
* `trusted_server_ca`: List of trusted CAs for server certificates

Other TLS related options are removed.

The new TLS configuration options accept special keywords listed below:

* `default`: When this keyword is provided, the default value for the option is
  used. The table below lists the options that accept the `default` keyword and
  the value used when the keyword is provided:

| Option | Default value |
| ------ | ------------- |
| `tls_dir` | `/var/lib/keylime/cv_ca` for server components. `/var/lib/keylime/secure` for the agent. |
| `client_key` | `client.key` |
| `client_cert` | `client_cert.pem` |
| `server_key` | `server.key` |
| `server_cert` | `server_cert.pem` |
| `trusted_client_ca` | `[cacert.pem]` |
| `trusted_server_ca` | `[cacert.pem]` |

* `all` (only valid for `trusted_client_ca` and `trusted_server_ca`): Disables validation of
  the certificates.
* `generate` (only valid for `tls_dir`): Generates a CA with the correct certificates.
  The agent generates the key and certificate when they are not present
  regardless of this option.

The following sub-sections describe the changes to TLS related options for each
component.

### Agent

For the agent the changes are only the options naming scheme changes to make it
consistent with the other components. Since the agent does not access other
components as a client using TLS, it does not have the `client_key`,
`client_cert`, and `trusted_server_ca` options.

The following table summarizes the changes from the old configuration file
format.

| Old option          | New Option          | Notes |
| ------------------- | ------------------- | ----- |
| `rsa_keyname`       | `server_key`        | |
| `mtls_cert_enabled` | `enable_agent_mtls` | |
| `mtls_cert`         | `server_cert`       | |
| `keylime_ca`        | `trusted_client_ca` | Add the verifier and tenant clients CA certificates to `trusted_client_ca`|

### Verifier

For the verifier, besides the options naming scheme changes for consistency,
some options are removed since only one client key pair and certificate can be
set to access the other components.

Also the option to use password protected private keys is no longer supported,
allowing the `private_key_pw` and `agent_mtls_private_key_pw` to be removed.

The following table summarizes the changes for the verifier TLS configuration.

| Old option                 | New Option          | Notes |
| -------------------------- | ------------------- | ----- |
| `tls_dir`                  | `tls_dir`           | |
| `ca_cert`                  | `trusted_client_ca` | Add the tenant client CA certificate to `trusted_client_ca`|
| `my_cert`                  | `server_cert`       | |
| `private_key`              | `server_key`        | |
| `private_key_pw`           | (None, removed)     | Removed support for password protected private keys |
| `check_client_cert`        | (None, removed)     | Use `all` keyword in `trusted_client_ca`|
| `agent_mtls_cert_enabled`  | `enable_agent_mtls` | |
| `agent_mtls_cert`          | (None, removed)     | Add the CA of the agent's `server_cert` to `trusted_server_ca`
| `agent_mtls_private_key`   | `client_key`        | |
| `agent_mtls_private_key_pw`| (None, removed)     | Removed support for password protected private keys |

### Tenant

The tenant access multiple components as a client. The old configuration format
allows to set different private keys and certificates to be used when accessing
each different component. In the new configuration format, all services are
accessed using the same private key (`client_key`) and certificate
(`client_cert`).

To allow the components to authenticate the tenant, it is necessary to add the
tenant CA certificate to the `trusted_client_ca` list in each component
configuration.

Also the option to use password protected private keys is no longer supported,
allowing the `private_key_pw`, `agent_mtls_private_key_pw`, and
`registrar_private_key_pw` to be removed.

| Old option                 | New Option          | Notes |
| -------------------------- | ------------------- | ----- |
| `tls_dir`                  | `tls_dir`           | |
| `ca_cert`                  | `trusted_server_ca` | Add the verifier CA certificate to `trusted_server_ca`|
| `my_cert`                  | `client_cert`       | |
| `private_key`              | `client_key`        | |
| `private_key_pw`           | (None, removed)     | Removed support for password protected private keys |
| `check_client_cert`        | (None, removed)     | Use `all` keyword in `trusted_client_ca`|
| `agent_mtls_cert_enabled`  | `enable_agent_mtls` | |
| `agent_mtls_cert`          | (None, removed)     | Add the CA of the agent's `server_cert` to `trusted_server_ca`
| `agent_mtls_private_key`   | `client_key`        | |
| `agent_mtls_private_key_pw`| (None, removed)     | Removed support for password protected private keys |
| `registrar_tls_dir`        | (None, removed)     | Use `tls_dir` |
| `registrar_ca_cert`        | (None, removed)     | Add the CA of the registrar's `server_cert` to `trusted_server_ca`|
| `registrar_my_cert`        | `client_cert`       | |
| `registrar_private_key`    | `client_key`        | |
| `registrar_private_key_pw` | (None, removed)     | Removed support for password protected private keys |
| `check_client_cert`        | (None, removed)     | Use `all` keyword in `trusted_client_ca`|

### Registrar

For the registrar, the options are renamed to follow the new naming scheme and
some options are removed since there is no need to set multiple server private
keys and certificates.

The registrar does not access other services as a client, thus it does not have
the `client_key`, `client_cert`, and `trusted_server_ca` options.

Also the option to use password protected private keys is no longer supported,
allowing the `private_key_pw` and `registrar_private_key_pw` to be removed.

The following table summarizes the changes for the registrar TLS configuration.

| Old option                 | New Option          | Notes |
| -------------------------- | ------------------- | ----- |
| `registrar_tls_port`       | `tls_port`          | |
| `tls_dir`                  | `tls_dir`           | |
| `ca_cert`                  | `trusted_client_ca` | Add the tenant CA certificates to `trusted_client_ca`|
| `my_cert`                  | `server_cert`       | |
| `private_key`              | `server_key`        | |
| `private_key_pw`           | (None, removed)     | Removed support for password protected private keys |
| `registrar_tls_dir`        | (None, removed)     | Use `tls_dir` |
| `registrar_private_key_pw` | (None, removed)     | Removed support for password protected private keys |
| `check_client_cert`        | (None, removed)     | Use `all` keyword in `trusted_client_ca`|

## Proposal - Split the configuration into individual files for each component and cleanup options

The proposal consists in moving the configuration options from the current
configuration file sections into individual configuration files corresponding to
each of the sections.

As part of the options cleanup, the TLS configuration is simplified with the
goal to make the options naming more consistent and intuitive between the
different components, as described in the sections above.

The global configuration section is removed as the options present in that
section are not necessary in the new configuration format.

The logging and CA related sections are moved to dedicated configuration files.

The following table shows the file which contains the options from each of
the old configuration sections:

| Old configuration section   | File in new configuration |
| --------------------------- | ------------------------- |
| `[general]`                 | (None, removed)           |
| `[cloud_agent]`             | `agent.conf`              |
| `[cloud_verifier]`          | `verifier.conf`           |
| `[registrar]`               | `registrar.conf`          |
| `[tenant]`                  | `tenant.conf`             |
| `[ca]`                      | `ca.conf`                 |
| `[loggers]`                 | `logging.conf`            |
| `[handlers]`                | `logging.conf`            |
| `[formatters]`              | `logging.conf`            |
| `[logger_root]`             | `logging.conf`            |
| `[handler_consoleHandler]`  | `logging.conf`            |
| `[logger_keylime]`          | `logging.conf`            |

When packaging Keylime, distributions should ship the default configuration
files in `/usr/etc/keylime/` or, if not supported, in `/etc/keylime/`. When
`/etc/keylime/` is populated, the options from files in  `/usr/etc/keylime`
should be ignored. Manual overrides can be provided in files in
`/etc/keylime.d/`.

The Rust agent implementation can diverge from the configuration file format
used by the Python agent implementation, including adopting different
configuration options.

### Unification of data types
Since the configuration is for software written in Python the default Python
types are used.

* Booleans: `True` and `False`
* String: Strings; can be double quoted or not (e.g. `test = test.txt` or `test = "test with space.txt"`)
* List: Defined as in Python (e.g. `[1, 2, "string"]`)
* Dictionary: Defined as in Python (e.g. `{"key": "value", "key2": 1}`)

Internal APIs to convert the entries in this format to Python structures should
be provided. A suggestion is to provide a `getlist()` and `getdict()` APIs that
use `ast.literal_eval` to convert values from the format defined above to Python
structures.

The following sub-sections describe the content of each of the new configuration
files.

### Simplification of database configuration

The following options are removed in favor of keeping only the `database_url`
option for database configuration:

* `database_drivername`
* `database_username`
* `database_password`
* `database_host`
* `database_name`

### Removal of unused default values from the configuration file

The following options define default values which are mostly unused and
therefore are removed:

* `tmp_policy`
* `ima_allowlist`
* `ima_excludelist`

### agent.conf

This file contains the configuration options used by the Keylime agent.  The
option are derived from the options present in the `[cloud_agent]` section from
the old configuration file.

The following table lists the old options and the corresponding new options.

| Changed? | Old option from `[cloud_agent]` | New option in `agent.conf` |
| -------- | -------- | ---------- |
| New | (None) | `version`  |
| New | (None) | `tls_dir`|
| Yes | `cloud_agent_ip`       | `ip`       |
| Yes | `cloud_agent_port`     | `port`           |
| Yes | `agent_contact_ip` (optional)|`contact_ip` |
| Yes | `agent_contact_port` (optional) | `contact_port`|
| No  | `registrar_ip` | `registrar_ip`|
| No  | `registrar_port` | `registrar_port`|
| Yes | `rsa_keyname` | `server_key`|
| Yes | `mtls_cert_enabled` | `enable_agent_mtls`|
| Yes | `mtls_cert` | `server_cert`|
| Yes | `keylime_ca` | `trusted_client_ca`|
| No  | `enc_keyname` | `enc_keyname` |
| No  | `dec_payload_file` | `dec_payload_file`|
| No  | `secure_size` | `secure_size` |
| No  | `tpm_ownerpassword` | `tpm_ownerpassword`|
| No  | `extract_payload_zip` | `extract_payload_zip`|
| Yes | `agent_uuid` | `uuid`|
| Yes | `listen_notifications` | `enable_revocation_notifications` |
| No  | `revocation_cert` | `revocation_cert`|
| No  | `revocation_actions` | `revocation_actions`|
| No  | `payload_script` | `payload_script`|
| No  | `enable_insecure_payload` | `enable_insecure_payload`|
| No  | `measure_payload_pcr` | `measure_payload_pcr`|
| No  | `exponential_backoff` | `exponential_backoff`|
| No  | `retry_interval` | `retry_interval`|
| No  | `max_retries` | `max_retries`|
| No  | `tpm_hash_alg` | `tpm_hash_alg`|
| No  | `tpm_encryption_alg` | `tpm_encryption_alg`|
| No  | `tpm_signing_alg` | `tpm_signing_alg`|
| No  | `ek_handle` | `ek_handle`|
| No  | `run_as` | `run_as`|

Additionally to the options from the `[cloud_agent]` section, the `agent.conf`
file also receive the following options from the `[general]` section.

| Changed? | Old option from `[general]` | New option in `agent.conf` |
| -------- | -------- | ---------- |
| Yes  | `receive_revocation_ip` | `revocation_notification_ip`|
| Yes  | `receive_revocation_port` | `revocation_notification_port`|

Follows below the description of the options in the new configuration file
format:

* `version`: [String] Version number of the configuration file in Semver
  version number format
* `tls_dir`: [String] The directory where the keys and certificate files are stored. If
  the value is provided as `default`, will use the default value `/var/lib/keylime/secure`
* `ip`: [String] The agent server IP
* `port`: [String] The agent server port
* `contact_ip`: [String, optional] On which IP can the verifier/tenant contact the agent 
* `contact_port`: [String, optional] On which port can the verifier/tenant contact the agent 
* `registrar_ip`: [String] The registrar server IP
* `registrar_port`: [String] The registrar server port
* `enable_agent_mtls`: [Boolean] Enable mutual authentication when establishing TLS
  connection
* `server_key`: [String] The agent server private key
* `server_cert`: [String] The agent server certificate
* `trusted_client_ca`: [List(String)] The list of trusted client CA certificates
* `enc_keyname`: [String] Derived key K from U/V split for payload
* `dec_payload_file`: [String] Name of the file to store the decrypted payload
* `secure_size`: [String] Size of the secure mount tmpfs. Note that in most cases this
 is provided by the system distribution already. Use format accepted by the
 `mount` command for size parameter
* `tpm_ownerpassword`: [String] TPM owner password
* `extract_payload_zip`: [Boolean] Whether to try to unzip the received payload
* `uuid`: [String] UUID of the agent or one of the following keywords
  * `generate`: Creates a random agent UUID
  * `hash_ek`: Uses the hash of the EK public key in PEM format as the agent UUID
  * `hostname`: Uses the hostname of the system as the agent UUID
  * `environment`: Uses the environment variable "`KEYLIME_AGENT_UUID`" as the
    agent UUID.
  * `dmidecode`: Uses the system UUID obtained form `"dmidecode -s systemd-uuid"` as the agent UUID
* `enable_revocation_notifications`: [Boolean] Option to enable or disable listening for
revocation notifications from the verifier via ZeroMQ.
* `revocation_notification_ip`: [String] IP to listen for revocation notifications via ZeroMQ
* `revocation_notification_port`: [String] Port to listen for revocation notifications via ZeroMQ
* `revocation_cert`: [String] Place of the certificate to check revocation messages against
* `revocation_actions`: [String] List of revocations to run
* `payload_script`: [String] Name of the script to run
* `enable_insecure_payload`: [Boolean] Enable payloads even if mTLS is disabled
* `measure_payload_pcr`: [Number] Measure the payload into a PCR
* `exponential_backoff`: [Boolean] Whether to use exponential backoff for retries
* `retry_interval`: [Number] Time interval to wait between request retries in seconds, or base for the exponential backoff algorithm if enabled through `exponential_backoff` option.
* `max_retries`: [Number] Maximum number of retries
* `tpm_hash_alg`: [List(String)] List of hash algorithms used for PCRs
  * Default: `"sha256"`
* `tpm_encryption_alg`: [List(String)] List of encryption algorithms
  * Default: `"rsa"`
* `tpm_signing_alg`: [List(String)] List of signing algorithms
  * Default: `"rsassa"`
* `ek_handle`: [String] If the EK is already present on the TPM and Keylime
  should use it, the handle of the EK should be provided (e.g. `"0x81000000"`).
  If the `generate` keyword is provided, a new EK is generated.
* `run_as`: [String] User unde which the process will run after dropping privileges. Use the format "`user:group`"

### verifier.conf

This file contains the configuration options used by the Keylime verifier.  The
options are derived from the options present in the `[cloud_verifier]` section
from the old configuration file.

The following table lists the old options and the corresponding new options.

| Changed? | Old option in `[cloud_verifier]` | New option in `verifier.conf` |
| -------- | -------- | ---------- |
| New | (None) | `version` |
| Yes | `cloud_verifier_id` | `uuid` |
| Yes | `cloud_verifier_ip` | `ip` |
| Yes | `cloud_verifier_port` | `port` |
| No  | `registrar_ip` | `registrar_ip`|
| No  | `registrar_port` | `registrar_port`|
| No  | `tls_dir` | `tls_dir` |
| Yes | `ca_cert` | `trusted_client_ca` |
| Yes | `my_cert` | `server_cert` |
| Yes | `private_key` | `server_key` |
| Yes | `private_key_pw` | (None, removed) |
| Yes | `check_client_cert` | (None, removed) |
| Yes | `agent_mtls_cert_enabled` | `enable_agent_mtls` |
| Yes | `agent_mtls_cert` | (None, removed) |
| Yes | `agent_mtls_private_key` | `client_key` |
| Yes | `agent_mtls_private_key_pw` | (None, removed) |
| No  | `database_url` | `database_url` |
| Yes | `database_drivername` | (None, removed) |
| Yes | `database_username` | (None, removed) |
| Yes | `database_password` | (None, removed) |
| Yes | `database_host` | (None, removed) |
| Yes | `database_name` | (None, removed) |
| No  | `database_pool_sz_ovfl` | `database_pool_sz_ovfl` |
| No  | `auto_migrate_db` | `auto_migrate_db` |
| Yes | `multiprocessing_pool_num_workers` | `num_workers` |
| No  | `exponential_backoff` | `exponential_backoff` |
| No  | `retry_interval` | `retry_interval` |
| No  | `max_retries` | `max_retries` |
| No  | `quote_interval` | `quote_interval` |
| Yes | `revocation_notifiers` | (None, see `[revocations]` section description below ) |
| Yes | `revocation_notifier_ip` | (None, see `[revocations]` section description below ) |
| Yes | `revocation_notifier_port` | (None, see `[revocations]` section description below ) |
| Yes | `webhook_url` | (None, see `[revocations]` section description below ) |
| No  | `max_upload_size` | `max_upload_size` |
| No  | `measured_boot_policy_name` | `measured_boot_policy_name` |
| No  | `severity_labels` | `severity_labels` |
| No  | `severity_policy` | `severity_policy` |
| Yes | `tomtou_errors` | `ignore_tomtou_errors` |
| No  | `require_allow_list_signatures` | `require_allow_list_signatures` |

The `verifier.conf` file will get a new section `[revocations]` which contains
the revocation related configuration options. The table below describes the
content of the `[revocations]` section in the new configuration file format.

| Changed? | Old option in `[cloud_verifier]` | New option in `[revocations]` of `verifier.conf` |
| -------- | -------- | ---------- |
| New | (None) | `enabled_revocation_notifications` |
| Yes | `revocation_notifier_ip` | `zmq_ip` |
| Yes | `revocation_notifier_port` | `zmq_port` |
| No  | `webhook_url` | `webhook_url` |

Follows below the description of the options in the new configuration file
format:

* `version`: [String] Version number of the configuration file in Semver
  version number format
* `uuid`: [String] The verifier unique identifier
* `ip`: [String] The verifier server IP
* `port`: [String] The verifier server port
* `registrar_ip`: [String] The registrar server IP
* `registrar_port`: [String] The registrar server port
* `tls_dir`: [String] The directory where the keys and certificate files are stored. If
  the value is provided as `default`, will use the default value `/var/lib/keylime/cv_ca`
* `trusted_client_ca`: [String] The list of trusted client CA certificates
* `server_key`: [String] The verifier server private key
* `server_cert`: [String] The verifier server certificate
* `enable_agent_mtls`: [Boolean] Enable mutual authentication when establishing TLS
* `client_key`: [String] The verifier client private key
* `client_cert`: [String] The verifier client cerficate
* `trusted_server_ca`: [List(String)] List of trusted server CA certificates
* `database_url`: [String] Database configuration URL
* `database_pool_sz_ovfl`: [String] Limits for database connection pool size in SQAlchemy
* `auto_migrate_db`: [Boolean] Whether to automatically update DB schema using alembic
* `num_workers`: [Number] Number of processes (workers) to spawn
* `exponential_backoff`: [Boolean] Whether to use exponential backoff for retries
* `retry_interval`: [Number] Time interval to wait between request retries in seconds, or base for the exponential backoff algorithm if enabled through `exponential_backoff` option.
* `max_retries`: [Number] Maximum number of retries
* `quote_interval`: [Number] Polling interval in seconds for getting quotes from the agent
* `max_upload_size`: [Number] Maximum payload size in bytes for policies (allowlists)
* `measured_boot_policy_name`: [String] Name of the policy that is used for measured
boot
* `severity_labels`: [List] List of used severity levels
* `severity_policy`: [List(Dictionary(String: String))] List of dictionaries that map regexes that match a event ID to a severity label
* `ignore_tomtou_errors`: [Boolean] Whether "time of measure, time of use" (ToMToU) errors
should be treated as a failure
* `require_allow_list_signatures`: [Boolean] Whether allowlist signatures should
    be required

Below follows the description of the options in the `[revocations]` section:
* `enabled_revocation_notifications`: [List(String)] List of enabled methods for sending revocation notifications
* `zmq_ip`: [String] IP to listen for revocation notifications via ZeroMQ
* `zmq_port`: [String] Port to listen for revocation notifications via ZeroMQ
* `webhook_url`: [String] URL to send notifications via webhook

### `tenant.conf`

This file contains the configuration options used by the Keylime tenant.  The
options are derived from the options present in the `[tenant]` section from the
old configuration file.

The following table lists the old options and the corresponding new options.

| Changed? | Old option in `[tenant]` | New option in `tenant.conf` |
| -------- | -------- | ---------- |
| New | (None) | `version` |
| Yes | `cloud_verifier_ip` | `verifier_ip` |
| Yes | `cloud_verifier_port` | `verifier_port` |
| No  | `registrar_ip` | `registrar_ip` |
| No  | `registrar_port` | `registrar_port` |
| No  | `tls_dir` | `tls_dir` |
| Yes | `ca_cert` | `trusted_server_ca` |
| Yes | `my_cert` | `client_cert` |
| Yes | `private_key` | `client_key` |
| Yes | `agent_mtls_cert_enabled` | `enable_agent_mtls` |
| Yes | `agent_mtls_cert` | (None, removed) |
| Yes | `agent_mtls_private_key` | `client_key` |
| Yes | `agent_mtls_private_key_pw` | (None, removed) |
| No  | `tpm_cert_store` | `tpm_cert_store` |
| Yes | `private_key_pw` | (None, removed) |
| Yes | `registrar_tls_dir` | (None, removed) |
| Yes | `registrar_ca_cert` | `trusted_server_ca` |
| Yes | `registrar_my_cert` | `client_cert` |
| Yes | `registrar_private_key` | `client_key` |
| No  | `max_payload_size` | `max_payload_size` |
| Yes | `tpm_policy` | (None, removed) |
| Yes | `ima_allowlist` | (None, removed) |
| Yes | `ima_excludelist` | (None, removed) |
| No  | `accept_tpm_hash_algs` | `accept_tpm_hash_algs` |
| No  | `accept_tpm_encryption_algs` | `accept_tpm_encryption_algs` |
| No  | `accept_tpm_signing_algs` | `accept_tpm_signing_algs` |
| No  | `exponential_backoff` | `exponential_backoff` |
| No  | `retry_interval` | `retry_interval` |
| No  | `max_retries` | `max_retries` |
| No  | `require_ek_cert` | `require_ek_cert` |
| No  | `ek_check_script` | `ek_check_script` |

Follows below the description of the options in the new configuration file
format:

* `version`: [String] Version number of the configuration file in Semver
  version number format
* `verifier_ip`: [String] IP of the verifier server
* `verifier_port`: [String] Port of the verifier server
* `registrar_ip`: [String] IP of the registrar server
* `registrar_port`: [String] Port of the registrar server
* `tls_dir`: [String] The directory where the keys and certificate files are stored. If
  the value is provided as `default`, will use the default value `/var/lib/keylime/cv_ca`
* `enable_agent_mtls`: [Boolean] Enable mutual authentication when establishing TLS
* `client_key`: [String] The tenant client private key
* `client_cert`: [String] The tenant client certificate
* `trusted_server_ca`: [List(String)] List of trusted server CA certificates
* `tpm_cert_store`: [String] Location of the directory with the trusted EK CA certificates
* `max_payload_size`: [Number] Maximum size of the payload to send in bytes. Should match tmpfs size for the agent secure mount set via `secure_size` option in the `agent.conf` file.
* `accept_tpm_hash_algs`: [List(String)] List of allowed hash algorithms
  * Default: `"sha512, sha384, sha256, sha1"`
* `accept_tpm_encryption_algs`: [List(String)] List of accepted encryption algorithms.
  * Default: `"ecc, rsa"`
* `accept_tpm_signing_algs`: [List(String)]List of supported signing algorithms
  * Default: `"ecschnorr, rsassa"`
* `exponential_backoff`: [Boolean] Whether to use exponential backoff for retries
* `retry_interval`: [Number] Time interval to wait between request retries in seconds, or base for the exponential backoff algorithm if enabled through `exponential_backoff` option.
* `max_retries`: [Number] Maximum number of retries
* `require_ek_cert`: [Boolean] Whether to require EK certificate validation
* `ek_check_script`: [String] Path to a script for validating the EK certificate. The following environment variables are set:
  * `EK`: PEM encoded version of the EK
  * `EK_CERT`: DER encoded EK certificate if available

### `registrar.conf`

This file contains the configuration options used by the Keylime registrar.  The
options are derived from the options present in the `[registrar]` section from
the old configuration file.

The following table lists the old options and the corresponding new options.

| Changed? | Old option in `[registrar]` | New option in `registrar.conf` |
| -------- | -------- | ---------- |
| Yes | (None, new) | `version` |
| Yes | `registrar_ip` | `ip` |
| Yes | `registrar_port` | `port` |
| Yes | `registrar_tls_port` | `tls_port` |
| No  | `tls_dir` | `tls_dir` |
| Yes | `ca_cert` | `trusted_client_ca` |
| Yes | `my_cert` | `server_cert` |
| Yes | `private_key` | `server_key` |
| Yes | `private_key_pw` | (None, removed) |
| Yes | `registrar_tls_dir` | `tls_dir` |
| Yes | `registrar_private_key_pw` | (None, removed) |
| Yes | `check_client_cert` | (None, removed) |
| No  | `database_url` | `database_url` |
| Yes | `database_drivername` | (None, removed) |
| Yes | `database_username` | (None, removed) |
| Yes | `database_password` | (None, removed) |
| Yes | `database_host` | (None, removed) |
| Yes | `database_name` | (None, removed) |
| No  | `database_pool_sz_ovfl` | `database_pool_sz_ovfl` |
| No  | `auto_migrate_db` | `auto_migrate_db` |
| No  | `prov_db_filename` | `prov_db_filename` |

Follows below the description of the options in the new configuration file
format:

* `version`: [String] Version number of the configuration file in Semver
  version number format
* `ip`: [String] The registrar server IP
* `port`: [String] The registrar server port for plain HTTP connection.  Used for agent registration
* `tls_port` [String] The registrar server port for TLS connection.
* `tls_dir`: [String] The directory where the keys and certificate files are stored. If
  the value is provided as `default`, will use the default value `/var/lib/keylime/cv_ca`
* `server_key`: [String] The registrar server private key
* `server_cert`: [String] The reidtrar server certificate
* `trusted_client_ca`: [List(String)] The list of trusted client CA certificates
* `database_url`: [String] Database configuration URL
* `database_pool_sz_ovfl`: [String] Limits for database connection pool size in SQAlchemy
* `auto_migrate_db`: [Boolean] Whether to automatically update DB schema using alembic
* `prov_db_filename`: [String] File to persist provider hypervisor data on SQLite.

### `ca.conf`

For the CA configuration, no changes are made to the options. All options that
are in the `[ca]` section of the old configuration file format are moved to the
dedicated `ca.conf` file as they are.

| Changed? | Old option in `[ca]` | New option in `ca.conf` |
| -------- | -------- | ---------- |
| No | `cert_country` | `cert_country` |
| No | `cert_ca_name` | `cert_ca_name` |
| No | `cert_state` | `cert_state` |
| No | `cert_locality` | `cert_locality` |
| No | `cert_organization` | `cert_organization` |
| No | `cert_org_unit` | `cert_org_unit` |
| No | `cert_ca_lifetime` | `cert_ca_lifetime` |
| No | `cert_lifetime` | `cert_lifetime` |
| No | `cert_bits` | `cert_bits` |
| No | `cert_crl_dist` | `cert_crl_dist` |

Follows below the description of the options in the new configuration file
format:

* `cert_country`: [String] Used as the `CountryName` argument (`C`) of the `Issuer` when generating certificates
* `cert_ca_name`: [String] Used as the `CommonName` argument (`CN`) of the `Issuer` when generating certificates
* `cert_state`: [String] Used as the `StateOrProvinceName` argument (`S`) of the `Issuer` when generating certificates
* `cert_locality`: [String] Used as the `Locality` argument (`L`) of the `Issuer` when generating certificates
* `cert_organization`: [String] Used as the `Organization` argument (`O`) of the `Issuer` when generating certificates
* `cert_org_unit`: [String] Used as the `OrganizationalUnit` argument (`OU`) of the `Issuer` when generating certificates
* `cert_ca_lifetime`: [Number] CA certificate validity time in days
* `cert_lifetime`: [Number] Default generated certificate validity time in days
* `cert_bits`: [Number] Key length in bits
* `cert_crl_dist`: [String] Certification Revocation List (CRL) distribution
  address (URL)

### `logging.conf`

The logging configuration uses multiple sections in the old configuration
format.  In the new configuration format such sections are moved to a dedicated
`logging.conf` file, but no changes are made to the options and section names.

The sections moved to the `logging.conf` file are:

* `[loggers]`
* `[handlers]`
* `[formatters]`
* `[logger_root]`
* `[handler_consoleHandler]`
* `[logger_keylime]`

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

The new TLS configuration removes the possibility to set separate private key
and certificate per remote server, limiting to use a single private key and
certificate to access all servers.

### Risks and Mitigations

Setting up the client TLS verification is important for the overall security of
Keylime. Warning for obvious insecure settings should be added where not already
in place. The documentation will describe the default configuration and one more
complex setup as a reference.

## Design Details

The tool to convert the old configuration file to the new configuration format,
split in multiple files, should be implemented as a python script.

### Test Plan

The current tests will be modified to use the new configuration file format and
options names.

### Upgrade / Downgrade Strategy

For the most common setups we provide a migration script to the new config
format. If only an old `keylime.conf` is found Keylime will error with a message
that migration to the new format is required.

Automatic downgrading will not be supported. The user can just use their old `keylime.conf`.


### Dependency requirements


## Drawbacks

* Configuration must be migrated. We will not support the old and the new format
  simultaneously.
* The support to set components to use different client private key and
  certificate per accessed server is removed. Each component that access servers
  as clients will use a single private key and certificate to access all servers.

## Alternatives

* Keep the current TLS setup and configuration file format
