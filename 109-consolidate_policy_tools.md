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
# enhancement-109: Consolidate policy generation tools

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

Keylime provides multiple policy generation and editting tools with separate
functionalities located in different parts of the repository:

* `scripts/create_runtime_policy.sh`
  * A shell script that measures all the files in the filesystem to generate a
    runtime policy as an allowlist
* `keylime/cmd/convert_runtime_policy.py`
  * Converts a runtime policy from the legacy format (i.e. allowlist and excludelists)
    to the unified JSON format
* `keylime/cmd/create_policy.py`
  * The main tool to create an manipulate policies in the unified JSON format.
* `keylime/cmd/sign_runtime_policy.py`
  * Signs a runtime policy, generating a signed policy in the Dead Simple
    Signing Envelope (DSSE) format

These scripts are not fit for usage in production environment:

* There is no consistency in their user interface
* There is not enough documentation on their usage
* Each script provides a different functionality, although all scripts are used
  for policy creation and editing
* They are not located in a consistent location in the repository

The goal of this enhancement is to provide a single tool consolidating all the
existing functionality from the scripts above adding consistent interface and
documentation

## Motivation

<!--
This section is for explicitly listing the motivation, goals and non-goals of
this enhancement.  Describe why the change is important and the benefits to users.
-->

Creating the attestation policies is a requirement for the deployment of
keylime, but the existing scripts are difficult to use. The main goal is to
lower the difficulty of creating and editing policies.

### Goals

<!--
List the specific goals of the enhancement.  What is it trying to achieve?  How will we
know that this has succeeded?
-->

* Consolidate all the policy generation and editing tools provided in the
  repository in a single tool
* Provide a consistent user interface for all the functionalities
* Provide documentation for the policy generation tool

### Non-Goals

<!--
What is out of scope for this enhancement?  Listing non-goals helps to focus discussion
and make progress.
-->

* Adding new functionalities to the consolidated policy generation tool is out
  of scope. The goal of this enhancement is only to consolidate the existing
  tools.

## Proposal

<!--
This is where we get down to the specifics of what the proposal actually is.
This should have enough detail that reviewers can understand exactly what
you're proposing, but should not include things like API designs or
implementation.  The "Design Details" section below is for the real
nitty-gritty.
-->

The proposal is to create a single tool that will provide the functionality of
the existing scripts. The necessary steps would be:

* Design a consistent interface for the consolidated tool
* Move all the functionality from the existing scripts into the new created tool
  following the interface design

### User Stories (optional)

<!--
Detail the things that people will be able to do if this enhancement is implemented.
Include as much detail as possible so that people can understand the "how" of
the system.  The goal here is to make this feel real for users without getting
bogged down.
-->

#### Story 1

* The user consults the policy generation tool documentation
* The user executes the policy generation tool to create a new policy file
  based on the measurements of the files in the file system
* Using the same tool, the user can combine an allowlist and an excludelist to
  generate a new policy
* Using the same tool, the user can sign the generated policy using a private
  key

### Notes/Constraints/Caveats (optional)

<!--
What are the caveats to the proposal?
What are some important details that didn't come across above.
Go in to as much detail as necessary here.
This might be a good place to talk about core concepts and how they relate.
-->

If some functionality is considered insecure, useless, or the implementation is
not possible, it can be excluded from the consolidated tool after obtaining
approval from the community.

### Risks and Mitigations

<!--
What are the risks of this proposal and how do we mitigate.  Think broadly.
For example, consider both security and how this will impact the larger
enhancement ecosystem.

How will security be reviewed and by whom?
-->

The existing tools should be kept for a grace period alongside the new tool.

## Design Details

<!--
This section should contain enough information that the specifics of your
change are understandable.  This may include API specs (though not always
required) or even code snippets.  If there's any ambiguity about HOW your
proposal will be implemented, this is the place to discuss them.
-->

The interface of the new consolidated policy generation tool should be
consistent, intuitive, and organized, meaning:

* Use subcommands to separate functionalities
* Use commonly used options for their usual purpose:
  * For example, `--help` or `-h` for help, `-o` for output, etc.
* Similar parameters of different subcommands should be passed using similar options
  * For example, 2 subcommands that receive a key as input could use the same
    `--key` option or `-k` as short
* Provide sane default values for optional parameters

The documentation should cover all the subcommands and options:

* With helpful error messages, which may indicate next steps or clarify missing
  information
* Ideally, with both concise and long versions
  * For example, a concise usage explanation with `--help` and a long usage
    explanation in manpages

A suggestion for the consolidation of the functionalities:

* Rewrite the functionality provided by `scripts/create_runtime_policy.sh` in python
* Move all the functionalities from the various scripts to the consolidated
  tool.

The output of the consolidated tool must be compatible with the current
components code and should not require any change to the components.

Additional usability improvements can be added to the tool, such as:

* Autocompleting of commands and/or options
* Possibility to provide machine-friendly input
  * For example, accepting the options setting via a JSON input
* Multiple levels of verbosity for the output

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

The code should be covered by unit tests included as part of the repository code
and additional end-to-end tests added to the CI test suite (in
https://github.com/RedHat-SP-Security/keylime-tests)

### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this enhancement
-->

This enhancement should not affect components upgrading or downgrading as the
intended use case is orthogonal to the components functionality.

### Dependencie requirements

<!--
If your new change requires new dependencies, please outline and demonstrate that your selected dependency 
is well maintained and packaged in Keylime's supported Operating Systems (currently Debian Stable
and as of time writing Fedora 32/33). 

During code implementation you will also be expected to add the package to CI , the keylime ansible role and 
keylimes main installer (`keylime/installers.sh`).

If the package is not available in the supported Operated systems, the PR will not be merged into master. 

Adding the package in `requirements.txt` is not sufficent for master which is where we tag releases from. 

You may however be able to work within an experimental branch until a package is made available. If this is
the case, please outline it in this enhancement.

-->

The addition of new dependencies should be minimized, and when needed, the
addition should be properly justified and approved by the maintainers.

## Drawbacks

<!--
Why should this enhancement _not_ be implemented?
-->

No drawbacks were identified as during the grace period nothing would be removed
from the repository

## Alternatives

<!--
What other approaches did you consider and why did you rule them out?  These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->

The alternative is to continue use the existing tools, with the option to try to
improve their usability individually. The downside of this alternative is that
the problem of having multiple tools with similar purpose will continue to
affect the users.

## Infrastructure Needed (optional)

<!--
Use this section if you need things infrastructure related specific to your enhancement.  Examples include a
new subproject, repos requested, github webhook, changes to CI (travis).
-->

No additional infrastructure should be needed.
