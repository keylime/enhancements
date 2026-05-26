# enhancement-129: Dual Certification and Crypto Agility

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
    - [Story 3](#story-3)
  - [Notes/Constraints/Caveats (optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Phase 1: Algorithm-Agnostic CA](#phase-1-algorithm-agnostic-ca)
  - [Phase 2a: Dual Certification in Verifier](#phase-2a-dual-certification-in-verifier)
  - [Phase 2b: Dual Certification in Registrar](#phase-2b-dual-certification-in-registrar)
  - [Phase 3a: Agent Dual Certificate Support](#phase-3a-agent-dual-certificate-support)
  - [Phase 3b: Registrar Support for Agent Secondary Certificate](#phase-3b-registrar-support-for-agent-secondary-certificate)
  - [Phase 3c: Verifier Trust for Dual-Cert Agents](#phase-3c-verifier-trust-for-dual-cert-agents)
  - [Phase 3d: Tenant Support for Secondary Agent Certificate](#phase-3d-tenant-support-for-secondary-agent-certificate)
  - [Future Work: PQC Validation and Testing](#future-work-pqc-validation-and-testing)
  - [Future Work: TLS Key Exchange Groups](#future-work-tls-key-exchange-groups)
  - [Test Plan](#test-plan)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Dependency requirements](#dependency-requirements)
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

This enhancement adds crypto agility and dual certification support to all
Keylime servers (verifier, registrar, and agent). The design removes RSA
hardcoding from the CA infrastructure, enables Keylime TLS servers to serve
two independent certificate chains of any algorithm type simultaneously, and
extends existing configuration options to accept list values for cert/key
paths. The approach is algorithm-agnostic: Keylime does not need to know
which algorithm a certificate uses. Users provide certificate and key file
paths; OpenSSL handles algorithm detection and negotiation automatically.
This provides immediate value (e.g., serving RSA + ECDSA) and future-proofs
Keylime for algorithm transitions (including future post-quantum algorithms)
without requiring any algorithm-specific code.

## Motivation

Keylime's TLS infrastructure currently has two significant limitations:

1. **RSA-only internal CA.** The built-in CA implementation
   (`ca_impl_openssl.py`) generates only RSA keys, uses RSA-specific type
   annotations, and hardcodes SHA-256 as the signing hash algorithm. The
   verification code in `ca_util.py` asserts
   `isinstance(pubkey, rsa.RSAPublicKey)` and uses `PKCS1v15()` padding.
   While the TLS layer itself already supports non-RSA certificates (e.g.,
   ECC) when provided by an external CA, the built-in CA cannot generate
   them, forcing operators who want non-RSA algorithms to use external CA
   infrastructure.

2. **Single certificate per server.** Each Keylime component loads exactly one
   certificate/key pair into its TLS context. This means there is no way to
   serve clients that require different algorithm types, and no way to run a
   transitional deployment where both old and new algorithm types coexist.

These limitations affect Keylime regardless of PQC considerations:

- **Algorithm transitions.** The industry has repeatedly transitioned between
  algorithm families (DSA to RSA, RSA to ECDSA). Each transition requires a
  period where servers present both certificate types. Without dual
  certification, Keylime cannot support such transitions gracefully — even
  though the TLS layer supports non-RSA certificates, only one certificate
  type can be served at a time, forcing a hard cutover.

- **Compliance requirements.** Different security standards may mandate
  different algorithm families. An operator subject to multiple compliance
  regimes may need to serve both RSA and ECDSA certificates simultaneously.

- **Crypto agility as a security property.** The ability to rapidly replace a
  broken algorithm is itself a security property. If a vulnerability is
  discovered in one algorithm family, a crypto-agile system can transition to
  another without a hard cutover.

- **PQC readiness.** Post-quantum signature algorithms (ML-DSA, SLH-DSA) are
  now NIST-standardized and arriving in TLS libraries. OpenSSL 3.5 (released
  April 2025) includes built-in support for ML-DSA and SLH-DSA (as well as
  ML-KEM for key exchange). When operators are ready to deploy PQC
  certificates, a crypto-agile Keylime supports them without any
  PQC-specific code. Note: ML-KEM is a Key Encapsulation Mechanism for TLS
  key exchange, not authentication. This proposal covers only the
  authentication side (certificates and signatures); TLS key exchange group
  configuration is deferred to future work (see [Future Work: TLS Key Exchange
  Groups](#future-work-tls-key-exchange-groups)).

The key insight is that **PQC support is not a special feature**. It is a
natural consequence of proper crypto agility. If Keylime can serve RSA + ECDSA
certificates simultaneously, it can serve RSA + ML-DSA certificates the same
way.

### Goals

* Remove RSA hardcoding from the Keylime CA infrastructure
  (`ca_impl_openssl.py`, `ca_util.py`, `cert_utils.py`). Support generating
  certificates with any algorithm the `cryptography` library supports,
  controlled by a configuration option.
* Enable all Keylime TLS servers (verifier, registrar, agent) to serve two
  independent certificate chains simultaneously, using standard OpenSSL
  multi-certificate support.
* Extend existing TLS configuration options (`server_cert`, `server_key`,
  `client_cert`, `client_key`) to accept list values for configuring
  multiple certificate/key pairs. No new option names for TLS paths.
  `tls_dir` remains a single directory where all certificates and keys
  are placed.
* Maintain full backward compatibility: existing single-value configurations
  continue to work without modification.
* Prove the design works with RSA + ECDSA (available today, testable
  everywhere) before considering PQC algorithms.
* Provide clear configuration validation with informative error messages for
  invalid list configurations (length mismatches, special keywords in list
  context, cert/key pair mismatches).

### Non-Goals

* Add PQC-specific configuration options or code paths. PQC is just another
  algorithm family, like ECDSA or Ed25519.
* Implement algorithm detection or classification logic. Keylime does not
  need to know whether a certificate is RSA, ECDSA, or ML-DSA.
* Maintain an allow-list of supported algorithms. If the underlying
  OpenSSL/cryptography library supports an algorithm, Keylime supports it.
* Change authentication mechanisms (e.g., replace mTLS with something else).
* Implement the IETF `draft-yusef-tls-pqt-dual-certs` TLS extension for
  explicit dual-chain presentation.
* Support more than two concurrent certificate chains (the list-based approach
  technically allows it, but the primary use case is two).
* Change the Keylime REST APIs beyond adding an optional field for the
  secondary agent certificate.

## Proposal

### Overview

Serving multiple certificates of different key types from the same TLS server
is standard TLS behavior. OpenSSL supports loading multiple certificate/key
pairs into the same `SSL_CTX` by calling `SSL_CTX_use_certificate()` and
`SSL_CTX_use_PrivateKey()` for each pair. During the TLS handshake, OpenSSL
automatically selects the certificate whose key type matches the client's
advertised `signature_algorithms` extension. This is the same mechanism that
has enabled web servers to serve both RSA and ECDSA certificates for over a
decade.

The proposal has three parts:

1. **Algorithm-Agnostic CA:** Remove RSA hardcoding from `ca_impl_openssl.py`
   and `ca_util.py`. Add an `algorithm` configuration option to the `[ca]`
   section. The CA reads this option to determine which key type to generate
   (e.g., `rsa`, `ecdsa`, `ed25519`).

2. **Dual Certification in TLS (Verifier and Registrar):** Extend existing
   configuration options (`server_cert`, `server_key`, `client_cert`,
   `client_key`) to accept Python list values alongside single strings.
   `tls_dir` remains a single directory. Extend `generate_tls_context()` in
   `web_util.py` to call `load_cert_chain()` for each cert/key pair.

3. **Agent Dual Certificate Support:** Enable the Rust agent to be configured
   with two certificate/key pairs for mTLS. Extend the registrar to store
   the agent's secondary certificate. Extend the verifier to trust both
   agent certificates.

The algorithm-agnostic design ensures that future algorithm families (including
post-quantum algorithms) will work through configuration alone, without
requiring code changes.

### Configuration Model

No new TLS option names are introduced. Instead, the existing options are
extended to accept Python list syntax:

**Single value (backward compatible, no changes needed):**
```ini
server_cert = default
server_key = default
```

**List value (dual certification):**
```ini
tls_dir = /var/lib/keylime/cv_ca
server_cert = ['server-cert-rsa.crt', 'server-cert-ecdsa.crt']
server_key = ['server-private-rsa.pem', 'server-private-ecdsa.pem']
trusted_client_ca = ['cacert-rsa.crt', 'cacert-ecdsa.crt']
```

`tls_dir` remains a single directory. All certificate and key files are
placed in (or referenced relative to) this directory. Relative paths in
cert/key lists are resolved against the single `tls_dir` value. Absolute
paths are used as-is.

**Validation rules:**

1. All related lists (`server_cert`, `server_key`) must have the same
   length.
2. Each certificate/key pair is validated at load time using
   `SSL_CTX_check_private_key()`.
3. The special keywords `default`, `generate`, and `all` are only valid as
   single values, not inside lists. Using them in a list is a fatal startup
   error with a clear message.
4. No empty entries are allowed in lists.
5. When more than one certificate is provided, each must have a distinct
   public key type (e.g., RSA, EC). OpenSSL's `SSL_CTX_use_certificate()`
   maintains one certificate slot per key type; loading two certificates of
   the same key type silently overwrites the first with no error or warning.
   Keylime detects this at startup by inspecting the public key type of each
   certificate and rejects duplicate key types with a clear error message.

**List detection logic:**

- Python components: Use `ast.literal_eval()`. If it returns a list, treat as
  list. Otherwise, treat as a single path. This is consistent with the
  existing `config.getlist()` approach.
- Rust agent: Use the existing Pest-based list parser in `list_parser.rs`.

### User Stories (optional)

#### Story 1

As a Keylime operator in an environment with mixed RSA and ECDSA clients, I
want the verifier and registrar to present both RSA and ECDSA certificates so
that all clients can connect without requiring algorithm uniformity across my
fleet.

#### Story 2

As a security engineer planning a future algorithm transition (e.g., from RSA
to ECDSA, or from classical to post-quantum algorithms), I want to deploy
certificates of the new algorithm type alongside my existing certificates so
that clients supporting the new algorithm can use it while legacy clients
continue with the existing algorithm. I want this to be a configuration change,
not a code change.

#### Story 3

As a Keylime operator, I want to generate ECDSA (or Ed25519) certificates
using Keylime's built-in CA, instead of relying on an external CA
infrastructure, so that I can use more modern and efficient algorithms
without added operational complexity.

### Notes/Constraints/Caveats (optional)

* The `trusted_client_ca` and `trusted_server_ca` options already accept
  lists of CA files. Their semantics are different from `server_cert`/
  `server_key` lists: trusted CA lists load **all** entries into the trust
  store, while cert/key lists correspond to each other **by index**
  (first cert with first key). No semantic change is needed for trusted CA
  options.

* The `default` keyword (which resolves to standard file names like
  `server-cert.crt`) only works with single-value settings. When using dual
  certification, explicit file names must be provided because `default`
  resolves to a single file name.

* `tls_dir` remains a single-value option (not extended to lists). All
  certificate and key files for dual certification must be placed in the
  same `tls_dir` directory, with distinct file names (e.g.,
  `server-cert-rsa.crt` and `server-cert-ecdsa.crt`). This keeps the key
  material in one location and avoids confusion from having certificates
  scattered across multiple directories.

* The verifier does **not** connect to the registrar over HTTPS. Agent data
  (including the agent's mTLS certificate) is provided by the tenant via
  `POST /agents` and stored in the verifier's own database. The verifier's
  `client_key`/`client_cert` are used only for outbound mTLS connections to
  agents.

* The push model agent is a TLS client only (no TLS server). It does not need
  dual server certificates but benefits from dual certification on the trust
  side (loading multiple CA certificates via `trusted_server_ca`).

* The Rust agent currently generates a self-signed certificate only when no
  certificate file is found. The same fallback behavior applies to the
  secondary certificate: if configured but the file does not exist, the
  agent generates one using the configured algorithm.

### Risks and Mitigations

**Algorithm downgrade attacks.** When a server presents two certificates, an
attacker could attempt to force the use of a weaker algorithm by modifying
the client's `signature_algorithms` extension. Mitigation: TLS 1.3 encrypts
most of the handshake, and mTLS provides mutual authentication. An attacker
who modifies the ClientHello causes a handshake failure due to transcript hash
mismatch. Additionally, if both configured algorithms are secure, there is no
meaningful downgrade.

**Dual CA trust model.** When using two independent CAs, both must be secured.
Compromising one CA allows issuing certificates for that algorithm type.
Mitigation: independent CA key storage, same operational practices for both
CAs.

**Secondary agent certificate trust gap.** The primary mTLS certificate is
bound to the TPM through the EK/AK chain and `TPM2_Certify` on the NK. The
secondary certificate, as submitted during registration, has no TPM binding.
Because registration can use plain HTTP, a network attacker could substitute
the secondary certificate in the registration payload while leaving the
primary certificate, AK, and EK intact — the TPM challenge-response would
still pass, granting the attacker a verifier-trusted identity through the
injected secondary certificate. Mitigation: the agent proves possession of
the secondary private key during registration by signing a registrar-issued
nonce with that key; the registrar verifies the signature before storing
the certificate. Operators are also recommended to enable TLS for
registration when using dual agent certificates, which prevents substitution
attacks on the registration payload regardless.

**Python `ssl.SSLContext` multiple `load_cert_chain()` behavior.** The Python
documentation does not explicitly state that multiple `load_cert_chain()` calls
work for different key types. The underlying OpenSSL behavior is well-defined,
but validation is needed. Mitigation: early prototyping and unit testing
against the specific Python versions Keylime supports.

**Rust `openssl` crate dual certificate API.** The crate may not expose
`add_certificate()` on `SslAcceptorBuilder` directly. Mitigation: use
lower-level `SslContextBuilder` methods, which map directly to OpenSSL's C API.

**Configuration complexity.** List-based configuration adds complexity for
operators. Mitigation: single-value configs (the default) are unchanged;
dual cert is entirely opt-in; comprehensive validation with clear error
messages guides correct configuration.

## Design Details

The implementation is structured in six tasks across three phases, each
independently deliverable. A future PQC validation exercise is planned
separately.

### Phase 1: Algorithm-Agnostic CA

**Files:** `ca_impl_openssl.py`, `ca_util.py`, `cert_utils.py`, `ca.conf`

1. Add `algorithm` and `cert_key_params` options to `[ca]` config section.
   The `algorithm` option accepts values like `rsa`, `ecdsa`, `ed25519`,
   `ed448`, or any algorithm name the `cryptography` library recognizes.
   Default: `rsa` (preserves current behavior).

2. Generalize `mk_request()` in `ca_impl_openssl.py`: replace RSA-only key
   generation with algorithm-dispatched generation based on the `algorithm`
   parameter.

3. Generalize `mk_cacert()` and `mk_signed_cert()`: use
   algorithm-appropriate signing (including `algorithm=None` for algorithms
   with built-in hashing like Ed25519). Update return types from
   RSA-specific (`RSAPrivateKey`) to generic
   (`CERTIFICATE_PRIVATE_KEY_TYPES`).

4. Remove RSA-specific assertions in `ca_util.py`. Replace
   `isinstance(pubkey, rsa.RSAPublicKey)` and `PKCS1v15()` verification with
   algorithm-agnostic verification (e.g.,
   `Certificate.verify_directly_issued_by()` from `cryptography >= 40.0`).

5. Update `cert_utils.py` `verify_cert()` with a generic fallback for
   key types beyond RSA and ECDSA.

6. Add `--algorithm` and `--key-params` flags to the `keylime_ca` CLI.

**Deliverables:**
- `keylime_ca -c init --algorithm ecdsa` generates an ECDSA CA.
- Default behavior remains RSA.
- All existing tests continue to pass.

### Phase 2a: Dual Certification in Verifier

**Files:** `web_util.py`, `config.py`, `verifier.conf`

1. Implement list detection logic in `config.py` using `ast.literal_eval()`.
   Single values are normalized to single-element lists at the parsing layer.

2. Implement configuration validation: list length consistency, no empty
   entries, no special keywords (`default`, `generate`, `all`) in list
   context, cert/key pair validity, and duplicate public key type detection
   (validation rule 5).

3. Extend `generate_tls_context()` in `web_util.py` to accept lists of
   cert/key paths and call `load_cert_chain()` for each pair:
   ```python
   for i, (cert, key) in enumerate(zip(certificates, private_keys)):
       password = private_key_passwords[i] if private_key_passwords else None
       context.load_cert_chain(certfile=cert, keyfile=key, password=password)
   ```

4. Extend `get_tls_options()` and `init_mtls()` to handle list-valued
   cert/key options. `tls_dir` remains a single value; relative paths in
   cert/key lists are resolved against it.

**Deliverables:**
- Verifier serves dual certificates (e.g., RSA + ECDSA).
- Clients receive the certificate matching their `signature_algorithms`.
- Existing single-certificate verifier configurations work unchanged.

### Phase 2b: Dual Certification in Registrar

**Files:** `web_util.py`, `config.py`, `registrar.conf`

The registrar uses the same TLS infrastructure as the verifier
(`web_util.py`). Phase 2a implements the shared infrastructure; Phase 2b
ensures it works correctly for the registrar's specific configuration and
ports.

1. Verify that the list-valued configuration parsing and TLS context
   generation from Phase 2a work for registrar configuration options.

2. Verify that the registrar's TLS port serves dual certificates correctly
   and that the HTTP port is unaffected.

**Deliverables:**
- Registrar serves dual certificates on its TLS port.
- HTTP port is unaffected.
- Existing single-certificate registrar configurations work unchanged.

### Phase 3a: Agent Dual Certificate Support

**Files:** `crypto.rs`, `crypto/x509.rs`, `main.rs` (Rust agent),
agent config

1. Extend the agent's TLS context to accept multiple certificate/key pairs.
   The Rust `openssl` crate does not expose `add_certificate()` or
   `add_private_key()`. Instead, use `set_certificate()` and
   `set_private_key()` in a loop. Despite the `set_` naming, OpenSSL
   internally indexes by key type, so each call with a different key type
   populates a new slot rather than overwriting:
   ```rust
   for (cert, key) in tls_certs.iter().zip(keys.iter()) {
       ssl_context_builder.set_certificate(cert)?;
       ssl_context_builder.set_private_key(key)?;
   }
   ```

2. Add `secondary_cert_algorithm` and `secondary_cert_key_params` config
   options. When `secondary_cert_algorithm` is set and no secondary
   certificate file exists, the agent generates a self-signed certificate
   using the configured algorithm (same fallback as the primary certificate).
   When the file exists, it is loaded directly.

3. Extend the Rust list parser to handle list-valued `server_cert` and
   `server_key` for operator-provided certificates (using the existing
   Pest-based parser in `list_parser.rs`).

4. When the agent has a secondary certificate, include it in the
   registration payload sent to the registrar as an optional field.
   When no secondary certificate is configured, the payload is unchanged.

5. To prove possession of the secondary private key, the registrar issues
   a short-lived nonce during the registration handshake. The agent signs
   the nonce with the secondary private key and includes the signature in
   the registration payload alongside the secondary certificate. The
   registrar verifies the signature using the public key from the secondary
   certificate before storing it. If verification fails, registration is
   rejected with a clear error.

**Deliverables:**
- Agent serves dual certificates when configured.
- Agent generates a secondary self-signed certificate as fallback when the
  file is not found.
- Agent registration includes the secondary certificate when present.

### Phase 3b: Registrar Support for Agent Secondary Certificate

**Files:** `registrar_db.py`, `agents_controller.py`, new Alembic migration

1. Add an optional column for the secondary certificate to the registrar's
   agent table via Alembic migration. The column is nullable; existing
   agents have `NULL`. The migration is non-destructive and reversible.

2. Update the registrar agent controller to handle the optional secondary
   certificate during agent registration:
   a. Issue a short-lived nonce when a secondary certificate is present in
      the registration payload.
   b. Verify the proof-of-possession signature (from Phase 3a step 5) over
      the nonce using the public key in the secondary certificate.
   c. Store the secondary certificate only after successful verification.
      Reject registration with a clear error if verification fails.

3. Update agent retrieval endpoints to include the secondary certificate
   in the response when present.

**Deliverables:**
- Registrar verifies proof-of-possession and stores the agent's secondary
  certificate.
- Registrar API returns the secondary certificate when present.
- Registration from agents without a secondary certificate continues to
  work unchanged.

### Phase 3c: Verifier Trust for Dual-Cert Agents

**Files:** `web_util.py` (agent TLS context), verifier quote-polling code

1. Extend `generate_agent_tls_context()` in `web_util.py` to accept
   multiple cert blobs:
   ```python
   def generate_agent_tls_context(
       component: str,
       cert_blobs: Union[str, List[str]],
       logger: Optional[Logger] = None,
   ) -> ssl.SSLContext:
   ```
   Each non-empty cert blob is loaded into the trust store.

2. Update the verifier code that creates per-agent TLS contexts to pass
   both the primary and secondary agent certificates (when present) from
   the agent data. The secondary certificate reaches the verifier via the
   tenant (see Phase 3d).

**Deliverables:**
- Verifier trusts both agent certificates in per-agent TLS contexts.
- Verifier connects to dual-cert agents using whichever certificate TLS
  negotiation selects.
- Verifier connects to single-cert agents identically to today.

### Phase 3d: Tenant Support for Secondary Agent Certificate

**Files:** `tenant.py`, `tenant_cmd_actions.py`, verifier agent API schema

The tenant is the component that delivers agent data to the verifier via
`POST /agents`. Without tenant changes, the secondary certificate stored in
the registrar never reaches the verifier. Phase 3d completes the
end-to-end flow.

1. Update the verifier's `POST /agents` API schema to accept an optional
   `secondary_mtls_cert` field alongside the existing `mtls_cert` field.
   When present, the verifier stores it in its agent database for use in
   per-agent TLS contexts (Phase 3c).

2. Update the tenant to include the secondary certificate in the agent
   data it sends to the verifier:
   a. Fetch the secondary certificate from the registrar's agent response
      (already returned by the registrar after Phase 3b).
   b. Include it as the optional `secondary_mtls_cert` field in the
      `POST /agents` payload to the verifier.

**Deliverables:**
- Tenant forwards the agent's secondary certificate from the registrar to
  the verifier.
- Verifier `POST /agents` API accepts and stores the optional secondary
  certificate.
- End-to-end dual-cert attestation flow works: agent registers with
  registrar → tenant provisions verifier with both certs → verifier
  connects to agent using TLS-negotiated certificate.
- Single-cert configurations are unchanged.

### Future Work: PQC Validation and Testing

**Dependencies:** OpenSSL 3.5+, `cryptography` library with ML-DSA support.

This is a future validation exercise, not part of the current scope. When PQC
libraries and operating system support mature (OpenSSL 3.5+, Fedora 43+/RHEL
10+), the algorithm-agnostic infrastructure from Phases 1-3 should support PQC
certificates (e.g., ML-DSA) without any code changes — only configuration.

### Future Work: TLS Key Exchange Groups

**Dependencies:** No new system dependencies; `SSL_CTX_set1_groups_list()` is
available in all currently supported OpenSSL versions.

This proposal covers authentication (certificates and signatures) but not TLS
key exchange. Keylime currently has no configuration surface for TLS key
exchange groups, so an operator who wants PQC key exchange (e.g.,
`X25519MLKEM768`) would need code changes beyond this proposal.

A future enhancement could add a `tls_groups` configuration option (in the
`[verifier]`, `[registrar]`, and agent config sections) that passes the
configured group list through to `SSL_CTX_set1_groups_list()`. This would
complete the PQC key exchange story with relatively little implementation
effort and the same algorithm-agnostic approach as this proposal.

### Test Plan

**Unit tests:**
- Algorithm-agnostic CA generation (RSA, ECDSA P-256/P-384, Ed25519).
- Config value list parsing: single string, Python list syntax,
  single-element list, edge cases with brackets in paths.
- Config validation: length mismatch, cert/key mismatch, `default`/
  `generate`/`all` in list context, empty entries, duplicate key types
  (validation rule 5).
- Dual `load_cert_chain()` with different key types.
- CRL generation with ECDSA and Ed25519 CA keys.
- Relative paths in cert/key lists resolved against single `tls_dir`.
- Proof-of-possession: agent signs registrar nonce; valid and invalid
  signatures; missing signature rejected.

**Integration tests (using `swtpm`):**
- Dual-cert verifier startup with RSA + ECDSA; RSA-only and ECDSA-only
  clients connect and receive the correct certificate.
- Same for registrar (TLS port only; HTTP port unaffected).
- Agent configured with dual certs serves both via TLS.
- Agent registration with dual certs stored in registrar DB (including
  proof-of-possession verification).
- Tenant forwards secondary cert to verifier; verifier `POST /agents`
  schema accepts and stores it.
- Full end-to-end attestation flow with dual-cert agent: agent →
  registrar → tenant → verifier → mTLS connection.
- Verifier-agent mTLS with dual-cert agent.
- Single-cert backward compatibility (existing configs work identically).
- Full attestation flow with ECDSA CA instead of RSA.
- Invalid config rejection with clear error messages (including duplicate
  key type rejection).

### Upgrade / Downgrade Strategy

**Upgrade path:**

1. **Stage 1 (no config changes):** Upgrade Keylime to the version that
   includes Phases 1-2. No configuration changes needed. System continues
   with the existing single RSA certificate. All behavior is identical to
   pre-upgrade.

2. **Stage 2 (enable dual cert on verifier):** Generate secondary
   certificates using the algorithm-agnostic CA or an external tool. Place
   all certificate and key files in `tls_dir`. Update verifier cert/key
   configuration to provide list values. Verifier now presents both
   certificate types.

3. **Stage 3 (enable dual cert on registrar):** Same as Stage 2 for the
   registrar.

4. **Stage 4 (enable agent dual certs):** Upgrade the tenant (Phase 3d)
   before or simultaneously with the agent (Phase 3a). Configure the agent
   with a secondary certificate algorithm. If no certificate file exists,
   the agent generates one on next startup. The agent registers both
   certificates with the registrar (including proof-of-possession). The
   tenant forwards the secondary certificate from the registrar to the
   verifier via `POST /agents`.

5. **Stage 5 (enable tenant dual certs):** Update tenant config to provide
   list values for client cert/key.

Each stage is independent and can be adopted at the operator's pace.

**Downgrade path:**

At any stage, reverting list-valued options back to single values restores
single-certificate operation. Secondary certificate files remain on disk but
are not loaded. The secondary certificate database column is additive and
does not affect downgraded operation (the column is simply ignored).

**Database migration:**

A new column for the secondary certificate is added to the registrar database
via Alembic migration. This is non-destructive. Downgrading the database
schema requires a reverse Alembic migration that drops the column.

### Dependency requirements

**Phase 1 (Algorithm-Agnostic CA):**

| Dependency       | Required Version | Purpose                           | Status   |
|------------------|------------------|-----------------------------------|----------|
| Python           | >= 3.9           | Type hints                        | Existing |
| `cryptography`   | >= 40.0.0        | Generic key types, `verify_directly_issued_by()` | Existing |

No new system dependencies. ECDSA and Ed25519 are supported by all currently
deployed OpenSSL versions.

**Phase 2a/2b (Dual Certification in Verifier and Registrar):**

| Dependency       | Required Version | Purpose                           | Status   |
|------------------|------------------|-----------------------------------|----------|
| OpenSSL          | >= 1.0.2         | Multiple cert types per `SSL_CTX` | Existing |
| Python `ssl`     | (stdlib)         | Multiple `load_cert_chain()` calls | Existing |

No new system dependencies.

**Phase 3a (Agent Dual Certs):**

| Dependency       | Required Version | Purpose                           | Status   |
|------------------|------------------|-----------------------------------|----------|
| `openssl` crate  | >= 0.10.x        | `add_certificate()` / low-level builder | Verify |

**Phase 3b (Registrar Secondary Cert):**

No new dependencies beyond existing Python keylime requirements.

**Phase 3c (Verifier Dual Agent Trust):**

No new dependencies beyond existing Python keylime requirements.

All dependencies are already present in Keylime's supported
operating systems (Fedora, Debian Stable, RHEL). No new packages need to be
added to CI or the installer.

## Drawbacks

* **Configuration complexity.** List-based TLS options add a new syntax that
  operators must learn for dual certification. However, this is entirely
  opt-in: single-value configs (the default) work exactly as today.

* **Positional coupling in lists.** The cert/key list approach relies on
  positional correspondence (first cert with first key, second cert with
  second key). An ordering mistake produces a cert/key mismatch. This is
  mitigated by the `SSL_CTX_check_private_key()` validation at startup.

* **Testing surface.** Dual certification multiplies the TLS testing matrix
  (single cert, dual cert, different algorithm combinations). However, the
  algorithm-agnostic design means individual algorithm combinations do not
  require separate code paths.

* **Environment variable encoding.** Encoding lists in environment variables
  is less natural than in config files. The canonical syntax
  (`KEYLIME_VERIFIER_SERVER_CERT="['a.crt', 'b.crt']"`) works with the
  `ast.literal_eval()` parser but is verbose.

## Alternatives

**Alternative 1: Dedicated `_2` suffix options**
(`server_cert_2`, `server_key_2`, etc.)

Instead of extending existing options to accept lists, add new options for
the secondary certificate. This avoids list parsing complexity and makes each
cert/key pair independently configurable.

Rejected because:
- Pollutes the configuration namespace with many new options.
- Does not generalize to more than two certificates.
- Inconsistent with how `trusted_client_ca` already accepts lists.
- Every new option requires its own documentation, validation, and tests.

**Alternative 2: PQC-specific options** (`pqc_enabled`, `pqc_server_cert`,
`pqc_sign_algorithm`, etc.)

Add PQC-specific configuration options and code paths.

Rejected because:
- PQC is just another algorithm family. Treating it specially creates
  unnecessary complexity and maintenance burden.
- Algorithm-specific code paths break the crypto agility design principle.
- Does not help with non-PQC dual certification use cases (e.g., RSA + ECDSA).

**Alternative 3: Directory-based convention** (separate directories for
primary/secondary certs with fixed naming).

Rejected because:
- Imposes file system layout constraints on operators.
- Less flexible than explicit list configuration.
- Harder to integrate with external certificate management tools.

## Infrastructure Needed (optional)

* CI testing with dual-certificate configurations (RSA + ECDSA) alongside
  existing single-cert tests.
