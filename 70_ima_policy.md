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
# enhancement-70: Merge allow and exclude list into an IMA policy

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

This enhancement proposal changes and unifies the IMA configuration options by
formally specifying a JSON schema. The tenant and verifier are modified to use
the IMA policy instead of the old flat file format to simplify the process and
allow further development like server side signature validation.

For users of the old flat file format a upgrade path using a commandline tool
will be provided.

## Motivation

The configuration for IMA validation is currently split over multiple options
and is not well documented. This makes it confusing to use and hard to implement
features like signature validation or storing all the information in a separate
DB table to reuse.

### Goals

* Merge the current allow and exclude list into one single JSON object
* Move `ima_sign_verification_keys` also into the JSON object
* Deprecate flat file allow and exclude list
* Simplify IMA related options in the tenant
* Provide a tool for converting flat file allow and exclude list to JSON object
* Convert the `create_allowlist` script to create a `ima_policy`.

### Non-Goals

* Change the current behavior of the IMA validation
* Change how the measured boot reference state works
* Implement server side signature validation

## Proposal

The entries `ima_sign_verification_keys` and `allowlist` will be replaced with
`ima_policy` in the verifier API. `ima_policy` takes a JSON object that contains
all the required data for IMA validation. A schema for the JSON object will be
provided in the Keylime repository.

The `allowlist*` options in the tenant will be renamed to `ima_policy*`.

The flat file format will be deprecated in favor of the JSON format. This is
done to simplify future development like server side signature validation. For
users of the old format a conversion tool will be provided.


### User Stories

#### Story 1a - Using the legacy flat file format
* User generates a allowlist `ima_allowlist.txt` and a exclude list `exclude.txt`
* Conversion to new JSON format with `keylime_convert_allowlist --allowlist ima_allowlist.txt --exclude exclude.txt --out ima_policy.json`
* The conversion tool produces `ima_policy.json`
* Adds a new agent with `keylime_tenant -c add --ima_policy ima.json ...`
* Agent is added to attestation.

#### Story 1b - Using the legacy flat file format with signature
* User generates a allowlist `ima_allowlist.txt` and a exclude list
  `exclude.txt`, allowlist signature `ima_allowlist.txt.sig` and the public key `allowlist.key`
* Conversion to new JSON format with `keylime_convert_allowlist --allowlist ima_allowlist.txt --exclude exclude.txt --allowlist-sig ima_allowlist.txt.sig --allowlist-sig allowlist.key --out ima_policy.json`
* The conversion tool validates the signature and produces `ima_policy.json`
* Adds a new agent with `keylime_tenant -c add --ima_policy ima.json ...`
* Agent is added to attestation.


Note that in this case the signature is checked by the conversion tool and that
the exclude list does not have a signature that can be validated. This is fine
because the signature validation is currently only done locally.


#### Story 2a - Using a JSON policy
 * User generates a IMA JSON policy called `ima.json`
 * Adds a new agent with `keylime_tenant -c add --ima_policy ima.json ...`
 * Agent is added to attestation.

#### Story 2b - Using a JSON policy with signature
 * User generates a IMA JSON policy called `ima.json` and has a signature
   `ima.json.sig` and a public key `policy.key`.
 * Adds a new agent with `keylime_tenant -c add --ima_policy ima.json --ima-policy-sig ima.json.sig --ima-policy-sig-key policy.key ...`
 * Agent is added to attestation.


### Notes/Constraints/Caveats

* This only affects the server side API of the verifier. Therefore we will not
  implement backwards compatibility for older tenant versions.
* Signature validation in the tenant will be only possible with the JSON schema.

### Risks and Mitigations

There are no new risks because we are just unifying the existing configuration
options.

## Design Details

### New IMA policy JSON schema
The IMA policy is formally specified as:
```json
{
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "title": "Keylime IMA policy",
    "type": "object",
    "properties": {
        "meta": {
            "type": "object",
            "properties": {
                "version": {
                    "type": "integer",
                    "description": "Version number of the IMA policy schema"
                }
            },
            "required": ["version"],
            "additionalProperties": false
        },
        "release": {
            "type": "number",
            "title": "Release version",
            "description": "Version of the IMA policy (arbitrarily chosen by the user)"
        },
        "digests": {
            "type": "object",
            "title": "File paths and their digests",
            "patternProperties": {
                ".*": {
                    "type": "array",
                    "title": "Path of a valid file",
                    "items": {
                        "type": "string",
                        "title": "Hash of an valid file"
                    }
                }
            }
        },
        "excludes": {
            "type": "array",
            "title": "Excluded file paths",
            "items": {
                "type": "string",
                "format": "regex"
            }
        },
        "keyrings": {
            "type": "object",
            "patternProperties": {
                ".*": {
                    "type": "string",
                    "title": "Hash of the content in the keyring"
                }
            }
        },
        "ima-buf": {
            "type": "object",
            "title": "Validation of ima-buf entries",
            "patternProperties": {
                ".*": {
                    "type": "string",
                    "title": "Hash of the ima-buf entry"
                }
            }
        },
        "verification-keys": {
            "type": "array",
            "title": "Public keys to verify IMA attached signatures",
            "items": {
                "type": "string"
            }
        },
        "ima": {
            "type": "object",
            "title": "IMA validation configuration",
            "properties": {
                "ignored_keyrings": {
                    "type": "array",
                    "title": "Ignored keyrings for key learning",
                    "description": "The IMA validation can learn the used keyrings embedded in the kernel. Use '*' to never learn any key from the IMA keyring measurements",
                    "items": {
                        "type": "string",
                        "title": "Keyring name"
                    }
                },
                "log_hash_alg": {
                    "type": "string",
                    "title": "IMA entry running hash algorithm",
                    "description": "The hash algorithm used for the running hash in IMA entries (second value). The kernel currently hardcodes it to sha1.",
                    "const": "sha1"
                }
            },
            "required": ["ignored_keyrings", "log_hash_alg"],
            "additionalProperties": false
        }
    },
    "required": ["meta", "release", "digests", "excludes", "keyrings", "ima", "ima-buf", "verification-keys"],
    "additionalProperties": false
}
```
### Verifier API changes
The option `ima_sign_verification_keys` will be removed. `allowlist` will be
renamed to `ima_policy` and only accepts a policy formatted in the schema above.


### Database changes
In `verifiermain` the column `ima_sign_verification_keys` will be dropped and
the column `allowlist` will be renamed to `ima_policy`.


### Tenant changes
Following changes are made to the tenant.

#### New options
* `--ima-policy`: Path to IMA policy in JSON format.
* `--ima-policy-url`: URL to IMA policy in JSON format.
* `--ima-policy-checksum`: SHA256 of the IMA policy.
* `--ima-policy-sig`: Path to the signature of the IMA policy.
* `--ima-policy-sig-url`: URL to the signature of the IMA policy.
* `--ima-policy-sig-key`: Key to verify the signature against (in the tenant).


#### Removal of following options
* `--allowlist`: Renamed to `--ima-policy`. No longer accepts flat files.
* `--allowlist-url`: Renamed to `--ima-policy-url`.
* `--allowlist-checksum`: Renamed to `--ima-policy-checksum`.
* `--allowlist-sig`: Renamed to `--ima-policy-sig`.
* `--allowlist-sig-url`: Renamed to `--ima-policy-sig-url`.
* `--allowlist-sig-key`: Renamed to `--ima-policy-sig-key`.
* `--exclude`: Is now part of the IMA policy.
* `--signature-verification-key`: The signature verification key is now part of
  the IMA policy.
* `--signature-verification-key-sig`: A separate signature is no longer needed,
  because it is part of the IMA policy.
* `--signature-verification-key-sig-key`: Same as above.
* `--signature-verification-key-url`: Same as above.
* `--signature-verification-key-sig-url`: Same as above.

The flat file to JSON conversion code will be moved out of the tenant into the
conversion tool. If a no longer supported argument is specified the tenant
errors with a note to use the conversion tool.

### Conversion tool
The conversion tool takes the removed options, does the optional verification
locally and returns a JSON policy. This will provided as a separate commandline
tool called `keylime_convert_allowlist`.


### IMA policy creation tool
Keylime provides a `create_allowlist` script that tries to produce an allowlist
from a running system. A new `create_ima_policy` script will be provided to do
the same for the new format. Note that these scripts only create examples and
not something that should be used in production.

### Test Plan

Existing tests for the IMA allowlist are migrated.


### Upgrade / Downgrade Strategy

Old database entries are converted during the upgrade. For downgrading the
agents have to be manually re-added with the different options.

### Dependency requirements

No new mandatory dependencies will be added. We might add a optional dependency
for implementing JSON schema validation.

## Drawbacks

The JSON format is more complex that the old flat file format, but more flexible
and future proof.

## Alternatives

* Keep the current configuration.
* Use another policy format like TOML or YAML.

## Infrastructure Needed (optional)

<!--
Use this section if you need things infrastructure related specific to your enhancement.  Examples include a
new subproject, repos requested, github webhook, changes to CI (travis).
-->
