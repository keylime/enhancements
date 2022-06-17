# enhancement-38: Allowlist management

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

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

## Summary

Allowlist management is, obviously, managing allowlists. Currently the allowlist
is bound with agent records, there are two issues in current design:

* Allowlist reusing: An allowlist can't be shared between agents, so we can't apply
  the same data to identical environments. By supporting reuse, we can also provide
  batch allowlist updates for matching agents.
* Performance: Since the ima measurements could be very large, it has impact on query
  efficiency when the allowlist does not participate in the agent processing, e.g., we
  don't need to load the allowlist for updating agent state.


## Motivation

### Goals

The goal is to provide an allowlist management API and CLI for allowlist management,
and establish a relationship between an allowlist and an agent. The allowlist used by
an agent can be changed when the host of the agent is upgraded and the old allowlist
doesn't apply.

### Non-Goals

Import allowlist from agent.

## Proposal

The proposal is to:

* Create a new table named ``allowlists`` to hold allowlist data that was originally hosted
  in the ``verifiermain`` table, namely: ``tpm_policy``, ``vtpm_policy``, and ``allowlist``.
* Add a foreign key named ``allowlist_id`` in the ``verifiermain`` table to reference the
  allowlist data that will be used for the agent.
* Implement ``/v2/allowlists/`` API for Verifier to support CRD operations. Updating an
  allowlist is not included in the initial proposal but can be implemented if necessary.
* Enhance ``tenant`` CLI to manage allowlists. When adding an agent to the Verifier, a user
  specifies an existing ``allowlist`` name instead of allowlist data to complete the
  registration.


### User Stories (optional)


#### Story 1

A user can reuse an allowlist by:

* Create an allowlist record by posting policies to the Verifier.
* Add an agent to the Verifier by providing Agent ID, IP Address and the allowlist name
  the agent will use.
* Add another agent running under the same environment to the Verifier with the same allowlist.

#### Story 2

A user can update an agent with a new allowlist by:

* Create an allowlist record by posting new policies to the Verifier.
* Update an agent to the Verifier by updating the allowlist name.
* Agent will use new allowlist to verify system integrety.

### Notes/Constraints/Caveats (optional)

None

### Risks and Mitigations

None

## Design Details

DB Changes:

The ``allowlists`` table will be created as:

```python
op.create_table('allowlists',
                sa.Column('id', sa.Integer(), nullable=False),
                sa.Column('name', sa.String(255), nullable=False),
                sa.Column('tpm_policy', sa.Text(), nullable=True),
                sa.Column('vtpm_policy', sa.Text(), nullable=True),
                sa.Column('ima_policy', sa.Text(length=429400000), nullable=True),
                sa.PrimaryKeyConstraint('id'),
                sa.UniqueConstraint('name', name='uniq_allowlists0name'),
                mysql_engine='InnoDB',
                mysql_charset='UTF8'
                )
```

The ``verifiermain.allowlist`` column will be renamed to ``ima_policy`` in the
``allowlists`` which provides a better presentation.

The ``verifiermain`` will have a new column as foreign key to the ``allowlists`` table:

```python
with op.batch_alter_table('verifiermain') as batch_op:
    batch_op.add_column(sa.Column('allowlist_id', sa.Integer(), nullable=True))
    batch_op.create_foreign_key('fk_verifiermain_allowlists', 'allowlists',
                                ['allowlist_id'], ['id'])
    batch_op.drop_column('tpm_policy')
    batch_op.drop_column('vtpm_policy')
    batch_op.drop_column('allowlist')
```

``tpm_policy``, ``vtpm_policy``, and ``allowlist`` will be removed from the
``verifiermain`` table.

### Test Plan

Downstream testing showed that this is working as expected, will add integration
test cases to cover this feature.


### Upgrade / Downgrade Strategy

The database migration will be handled by migration scripts.

## Drawbacks

Existing database records will not be migrated, after migration, agents will need
to be re-registred to the Verifier. The Registrar is not affected.

## Alternatives

None

## Infrastructure Needed (optional)

None
