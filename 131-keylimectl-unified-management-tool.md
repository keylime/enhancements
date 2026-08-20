# enhancement-131: keylimectl — Unified Keylime Management Tool

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Story 1: Enrolling an agent with push model](#story-1-enrolling-an-agent-with-push-model)
    - [Story 2: Managing policies end-to-end](#story-2-managing-policies-end-to-end)
    - [Story 3: Scripting and automation](#story-3-scripting-and-automation)
  - [Notes/Constraints/Caveats](#notesconstraintscaveats)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Command Structure](#command-structure)
  - [API Version Support](#api-version-support)
  - [Configuration](#configuration)
  - [Output Formats](#output-formats)
  - [Test Plan](#test-plan)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Dependency Requirements](#dependency-requirements)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
- [Infrastructure Needed](#infrastructure-needed)
<!-- /toc -->

## Release Signoff Checklist

- [ ] Enhancement issue in release milestone, which links to pull request in [keylime/enhancements]
- [ ] Core members have approved the issue with the label `implementable`
- [ ] Design details are appropriately documented
- [ ] Test plan is in place
- [ ] User-facing documentation has been created in [keylime/keylime-docs]

## Summary

This enhancement introduces `keylimectl`, a Rust-implemented unified command-line
management tool that replaces both `keylime_tenant` and the Python policy tools
(`keylime_create_policy`, `keylime_sign_runtime_policy`, `keylime_convert_runtime_policy`,
etc.) with a single, cohesive interface.

`keylimectl` is not a drop-in replacement: its interface was designed from scratch
with usability as a primary concern. While it covers the full functional scope of
the tools it replaces, command names, option names, and invocation patterns differ
deliberately to provide a consistent and intuitive experience.

By being implemented in Rust and distributed as a standalone binary, `keylimectl`
removes the Python runtime dependency from the Keylime management plane, which is
especially relevant for environments where the only Rust agent is deployed and
Python is not required for any other component.

`keylimectl` also implements full API v3 support, including the push model for
agent enrollment, which the current `keylime_tenant` does not support.

## Motivation

Managing Keylime agents today requires using multiple tools spread across the
Python codebase:

- **`keylime_tenant`**: the primary management tool, handles agent enrollment,
  removal, and status queries. It is implemented in Python, requires the full
  Keylime Python package to be installed, and only supports the pull model (API
  v2). Its interface mixes flags for unrelated operations and is difficult to
  script reliably.

- **`keylime_create_policy`**, **`keylime_sign_runtime_policy`**,
  **`keylime_convert_runtime_policy`**: a set of Python scripts for creating,
  signing, and converting runtime policies. Each has its own interface
  conventions; they are not easily composable and lack consistent documentation
  (see enhancement-109 for a prior consolidation attempt that proposed keeping
  these in Python).

This situation creates several pain points for operators:

1. **Python dependency in Rust-agent deployments**: Systems running only the Rust
   agent still need the Python Keylime package installed solely to run
   `keylime_tenant` and the policy tools. This increases the attack surface,
   complicates packaging, and is inconsistent with the direction of moving to
   a Python-free agent.

2. **No push model support in `keylime_tenant`**: API v3 introduces a push model
   where the agent connects to the verifier, removing the need for the tenant to
   know the agent's IP and port. Implementing API v3 support in the Python tenant
   would require significant effort to maintain going forward, for a tool that is
   already targeted for replacement.

3. **Fragmented policy tooling**: Generating, signing, validating, converting, and
   uploading policies requires chaining multiple tools with inconsistent interfaces.
   There is no single command to go from an IMA measurement list to a signed policy
   on the verifier.

4. **Poor scripting experience**: The tenant exits with non-zero codes in ways
   that are hard to distinguish, and its output is human-readable text rather
   than machine-parseable JSON, making automation brittle.

### Goals

- Provide a single Rust binary (`keylimectl`) that covers all agent lifecycle
  management and policy operations currently split across `keylime_tenant` and
  the Python policy tools.
- Eliminate the Python runtime dependency for Keylime management plane operations.
- Support API v3 natively, including the push model for agent enrollment.
- Deliver a consistent, intuitive command-line interface using a subcommand
  hierarchy (`keylimectl agent add`, `keylimectl policy generate runtime`, etc.).
- Support machine-readable output (JSON, YAML, table) for scripting and
  automation.
- Provide a `keylimectl configure` wizard to help users create a valid
  configuration file interactively.

### Non-Goals

- **Drop-in compatibility with `keylime_tenant`**: Command names, option names,
  and positional arguments intentionally differ. A compatibility shim or alias
  layer is out of scope.
- **Replacing the Keylime server components**: `keylimectl` is a client-only
  tool; verifier, registrar, and agent are not in scope.
- **Removing `keylime_tenant` immediately**: `keylime_tenant` and the Python
  policy tools will remain available for a transition period. Their deprecation
  timeline is a separate decision.
- **Supporting API v1**: Only API v2 (pull model, for backwards compatibility
  with existing deployments) and API v3 (push model, for new deployments) are
  in scope.

## Proposal

`keylimectl` is organized into top-level subcommands corresponding to the
resource or concern being managed:

| Subcommand       | Replaces                                        |
|------------------|-------------------------------------------------|
| `agent`          | `keylime_tenant` (add/remove/update/status/list/reactivate) |
| `policy`         | `keylime_create_policy`, `keylime_sign_runtime_policy`, `keylime_convert_runtime_policy` + verifier policy upload |
| `measured-boot`  | Measured boot policy management on the verifier |
| `verify`         | Offline TPM/TEE evidence verification           |
| `info`           | Connectivity diagnostics and version queries    |
| `configure`      | Interactive configuration file creation         |

Each subcommand has its own help text (`keylimectl <subcommand> --help`), and
the tool provides a summary of the active configuration when invoked without
arguments.

### User Stories

#### Story 1: Enrolling an agent with push model

An operator is setting up a new Fedora host running only the Rust keylime agent.
No Python is installed on the management workstation. The operator:

1. Installs the `keylimectl` RPM (no Python dependency).
2. Runs `keylimectl configure` and follows the interactive prompts to set
   verifier and registrar addresses and TLS certificate paths.
3. Creates a runtime policy:
   ```
   keylimectl policy generate runtime --ima-measurement-list -o my-policy.json
   ```
4. Signs the policy:
   ```
   keylimectl policy sign my-policy.json --output my-policy.dsse.json
   ```
5. Pushes the signed policy to the verifier:
   ```
   keylimectl policy push my-signed-policy --file my-policy.dsse.json
   ```
6. Enrolls the agent using the push model (agent connects to verifier):
   ```
   keylimectl agent add <agent-id> --push-model --runtime-policy my-signed-policy
   ```
7. Verifies the agent is being attested:
   ```
   keylimectl agent status <agent-id>
   ```

Throughout, the operator uses `--format table` for human-readable output and
`--format json` in automation pipelines.

#### Story 2: Managing policies end-to-end

A security engineer needs to update the runtime policy for a fleet of agents.
They:

1. Generate a new policy from the current measurement list:
   ```
   keylimectl policy generate runtime --ima-measurement-list \
     --excludelist /etc/keylime/excludelist.txt --output new-policy.json
   ```
2. Validate the policy structure:
   ```
   keylimectl policy validate new-policy.json
   ```
3. Sign it with an existing ECDSA key:
   ```
   keylimectl policy sign new-policy.json --keyfile signing.key \
     --output new-policy.dsse.json
   ```
4. Update the policy on the verifier without re-enrolling agents:
   ```
   keylimectl policy update fleet-policy --file new-policy.dsse.json
   ```
5. Confirm the updated policy is live:
   ```
   keylimectl policy show fleet-policy
   ```

#### Story 3: Scripting and automation

An automation pipeline checks whether a specific agent has been successfully
attested within the last cycle:

```bash
status=$(keylimectl agent status <agent-id> --format json | jq -r '.operational_state')
if [[ "$status" != "Get Quote" ]]; then
  echo "Agent not attested: $status"
  exit 1
fi
```

The consistent JSON output and predictable exit codes make this straightforward
without fragile text parsing.

### Notes/Constraints/Caveats

**Not a drop-in replacement**: Users upgrading from `keylime_tenant` must adapt
their scripts and documentation. The interface differences are intentional:
the tenant's flat flag model (`-t add`, `-t delete`, `-u <uuid>`, etc.) is
replaced with a subcommand hierarchy that is more consistent and discoverable.

**API version auto-detection**: When no API version is configured, `keylimectl`
probes the verifier to determine the highest supported version. It uses the
absence of the `/version` endpoint (HTTP 410) as a signal that the verifier
supports API v3, then confirms by testing the v3 endpoint directly. Users can
override the negotiated version via configuration.

**Push model requires API v3**: The `--push-model` flag on `keylimectl agent add`
is only usable when the verifier reports API v3 support. Attempting to use push
model with a v2 verifier results in a clear error message.

**Policy tools are local-only by default**: Policy generation, signing, validation,
and conversion do not require network connectivity; they operate entirely on local
files. Only `policy push`, `policy update`, `policy show`, `policy list`, and
`policy delete` contact the verifier.

**TLS**: `keylimectl` uses mutual TLS by default, consistent with other Keylime
components. Certificate paths are configured via `keylimectl configure` or the
configuration file.

### Risks and Mitigations

**Risk**: Users accustomed to `keylime_tenant` may be confused by the interface change.

**Mitigation**: Both tools will coexist during a deprecation period. The
`keylime_tenant` tool will emit deprecation warnings pointing users to
`keylimectl`. Documentation and migration notes will be provided.

**Risk**: Incomplete coverage of `keylime_tenant` edge cases.

**Mitigation**: The implementation is developed alongside the existing functional
test suite in `keylime-tests`. Any gap found during testing is treated as a blocker
for the deprecation of `keylime_tenant`.

**Risk**: Rust binary packaging is not yet uniform across all supported distributions.

**Mitigation**: The Rust agent (`keylime-agent`) is already packaged for Fedora,
CentOS Stream, RHEL, and Debian. `keylimectl` follows the same packaging
approach and shares build infrastructure with the Rust agent.

## Design Details

### Command Structure

```
keylimectl [OPTIONS] <SUBCOMMAND>

Options:
  --config <FILE>          Configuration file path
  --verifier-ip <IP>       Verifier IP address
  --verifier-port <PORT>   Verifier port
  --registrar-ip <IP>      Registrar IP address
  --registrar-port <PORT>  Registrar port
  --timeout <SECONDS>      Request timeout
  -v, --verbose            Increase verbosity (repeatable)
  -q, --quiet              Suppress all output except JSON results
  --format <FORMAT>        Output format: json (default), table, yaml
  --color <MODE>           Color mode: auto (default), always, never

Subcommands:
  agent         Manage agents (add, remove, update, status, list, reactivate)
  policy        Manage runtime policies (push, show, update, delete, list,
                generate, sign, verify-signature, validate, convert)
  measured-boot Manage measured boot policies (list, push, show, update, delete)
  verify        Verify attestation evidence offline (evidence)
  info          Show diagnostic information (verifier, registrar, agent, tls)
  configure     Create or update a configuration file
```

Notable design choices:

- **Positional `AGENT_ID`** instead of `-u <uuid>`: agent identifiers are
  positional arguments, consistent with modern CLI conventions.
- **`--push-model` / `--pull-model` flags** instead of implicit behavior:
  the attestation model is explicit, with push model being the preferred
  default for API v3 deployments.
- **`policy generate runtime`** replaces the separate measurement and allowlist
  scripts, with a single command accepting IMA logs, allowlists, rootfs scans,
  and RPM repositories as input sources.
- **`--wait-for-attestation`** on `agent add`: optionally blocks until the
  first successful attestation round completes, useful in provisioning pipelines.

### API Version Support

`keylimectl` is built with two compile-time feature flags, both enabled by default:

- `api-v2`: enables support for API versions 2.x (pull model)
- `api-v3`: enables support for API versions 3.x (push model, JSON:API format)

API version negotiation at runtime:
1. `keylimectl` issues a `GET /version` request to the verifier.
2. If the response is HTTP 410 Gone (the v3 verifier removes this endpoint),
   it confirms v3 by probing `GET /v3.0/`.
3. If v3 is confirmed, all subsequent requests use the JSON:API content type
   (`application/vnd.api+json`) and v3 URL paths.
4. If v3 is not confirmed, the tool falls back to v2.1 (highest v2 version).

Users can pin the API version in the configuration file to skip negotiation.

### Configuration

Configuration is loaded from the following sources in priority order (highest first):

1. Command-line arguments (`--verifier-ip`, `--timeout`, etc.)
2. Environment variables (`KEYLIME_VERIFIER__IP`, `KEYLIME_CLIENT__TIMEOUT`, etc.)
3. Configuration files searched in order:
   - Path specified via `--config`
   - `./keylimectl.toml` (project-local)
   - `~/.config/keylimectl/config.toml` (user)
   - `/etc/keylime/keylimectl.conf` (system)
4. Built-in defaults

The `keylimectl configure` subcommand creates a configuration file
interactively (with `--non-interactive` for automation) at the user or system
scope.

### Output Formats

All commands support three output formats selectable with `--format`:

- **`json`** (default): machine-readable JSON, suitable for `jq` pipelines and
  automation.
- **`table`**: human-readable columnar output, suitable for interactive use.
- **`yaml`**: YAML representation of the same data as JSON.

Exit codes follow standard UNIX conventions: 0 for success, non-zero for errors,
with distinct exit codes for configuration errors, network errors, and
attestation failures.

### Test Plan

- **Unit tests** cover policy generation, signing, validation, conversion, and
  API client logic. These are part of the `keylimectl` crate and run as part of
  `cargo test`.
- **Integration tests** use Mockoon-based mock servers for verifier and registrar
  responses, covering both API v2 and API v3 code paths without requiring a live
  Keylime deployment.
- **End-to-end tests** are added to the `keylime-tests` repository
  (`https://github.com/RedHat-SP-Security/keylime-tests`), exercising the full
  workflow:
  - Agent enrollment via push model with API v3 verifier
  - Agent enrollment via pull model with API v2 verifier
  - Policy lifecycle: generate → sign → push → update → delete
  - Measured boot policy lifecycle
  - Evidence verification
  - Error paths: invalid policy, unreachable verifier, certificate errors
- **Regression tests** confirm that all scenarios currently covered by
  `keylime_tenant` functional tests also pass with `keylimectl`.

### Upgrade / Downgrade Strategy

**Upgrade**: `keylimectl` is an additive introduction. Existing deployments using
`keylime_tenant` are unaffected. Administrators may adopt `keylimectl` on their
own schedule.

`keylime_tenant` will be deprecated (emitting a deprecation warning) once
`keylimectl` reaches feature parity as confirmed by the functional test suite.
Its removal will be scheduled for a future major version.

**Downgrade**: Removing `keylimectl` has no impact on server components. Agents
enrolled via `keylimectl` are managed identically by the verifier; the enrollment
tool is not involved in subsequent attestation rounds.

### Dependency Requirements

`keylimectl` is part of the `rust-keylime` workspace and shares its build
infrastructure. It introduces no new system-level dependencies beyond those
already required by the Rust agent:

- `openssl` (TLS)
- `tpm2-tss` (optional, for local TPM access features)

Optional Cargo dependencies introduced specifically for `keylimectl`:

- `dialoguer`: interactive terminal prompts for the `configure` wizard (optional,
  controlled by the `wizard` feature flag)
- `rpm`, `quick-xml`, `sequoia-openpgp`: RPM repository scanning for policy
  generation (optional, controlled by the `rpm-repo` feature flag)
- `indicatif`, `console`: progress display for long-running operations (e.g.
  filesystem scans)

All optional features can be disabled at build time; distributors may choose
which to include in their packages. The default build enables `api-v2`, `api-v3`,
and `wizard`, but not `rpm-repo` or `tpm-local`.

`keylimectl` is already packaged with the `rust-keylime` RPM spec and Debian
packaging. No new repositories or external infrastructure are required.

## Drawbacks

- **Interface migration cost**: Operators with existing `keylime_tenant` scripts
  must update them. This is mitigated by a deprecation period and migration
  documentation, but the cost is real.
- **Parallel maintenance during transition**: Both `keylime_tenant` and
  `keylimectl` will need to be kept working until the deprecation period ends.
  This is bounded in time and in scope (no new features will be added to
  `keylime_tenant`).

## Alternatives

**Implement API v3 / push model support in `keylime_tenant`**

This was considered and rejected. It would require maintaining a hybrid
Python implementation that supports two significantly different API models
(v2 pull and v3 push, with its JSON:API wire format). The Python tenant is
already complex; adding API v3 support would increase that complexity for a
tool that is targeted for eventual removal. The Rust implementation provides
a cleaner foundation and eliminates the Python runtime dependency at the same
time.

**Extend enhancement-109 (Python policy tool consolidation) instead**

Enhancement-109 proposed consolidating the Python policy tools into a single
`keylime_policy` binary. That approach keeps the Python runtime dependency and
results in two separate tools (tenant + policy) rather than one. `keylimectl`
supersedes enhancement-109 by providing both policy tooling and agent management
in a single Rust binary, making the Python consolidation effort unnecessary.

**Keep `keylime_tenant` as-is and add a thin Python wrapper for push model**

A shim that translates API v3 calls through the existing tenant code was
prototyped but abandoned: the tenant's internal model is too tightly coupled
to the pull-model workflow to accommodate push-model enrollment cleanly. The
resulting code would be difficult to test and maintain.

## Infrastructure Needed

`keylimectl` is developed within the `rust-keylime` repository
(`https://github.com/keylime/rust-keylime`) under the existing workspace
structure. No new repositories, webhooks, or CI infrastructure are required
beyond what is already in place for the Rust agent.

End-to-end tests are added to the existing `keylime-tests` repository.
