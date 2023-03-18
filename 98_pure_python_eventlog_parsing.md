# Python Eventlog Parsing for Measured Boot Attestation

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

## Proposal summary

The proposal is to replace the dependency on `tpm2-tools` in the
Keylime measured boot attestation code with native python
code.

Most of the change in Keylime itself to the nature of the parser's
invocation -- from `os.system` to native Python. Additional complexity
reduction can be achieved by removing the Python based post-processing
of the event log and merging that code into the parser. Current
features of post-processing include correct handling of UUIDs (badly
parsed in earlier versions of `tpm2-tools`) and `libefivar`
enhancements to parse device path entries.

We propose to keep the event log parser itself as a separate project
within the Keylime code base, and to package it independently. This
was proposed by the project maintainers as a way to keep disruptions
in Keylime to a minimum; once the python based event log parser is
packaged, it can be listed as a dependency on Keylime itself. To this
end, the python event log parser will have its own CI and unit tests,
code checks, and its own packaging mechanisms.

## Motivation

The chief motivation for this proposal is to keep measured boot
attestation safe from bit rot. We have, in the last two years of
operating Keylime (2021 to 2023), experienced at least two incidents
in which upgrading `tpm2-tools` to a new version caused measured boot
attestation to malfunction, because the output format (nominally YAML)
of the tool was changed.

Generally speaking the quality of `tpm2-tools` has been improving.
Some examples of disruptive changes we have encountered:
* Trailing zeroes used to be excluded, now are excluded;
* multiline output handled incorrectly in `tpm2-tools` v. 5.5
* certain input parsed to strings of incorrect length, resulting in utf-16 garbage
* UUIDs being interpreted incorrectly as in network order in v. 5.11

These are generally easy to correct for, but require us to actively
follow changes in the tooling, and each new version of `tpm2-tools`
requires us to adjust.

Therefore we propose to replace the approx. 1000 lines of C code in
the `tpm2-tools` package with a mostly equivalent length (700 lines at
present) of native Python code, maintained by the Keylime project. The
code savings come from the fact that OO features of Python allow for
simpler, more natural code organization; Python `struct` is very
effective at parsing binary structures into Python native objects; and
lastly, Python has JSON support.

### Goals

* Improve the stability of measured boot attestation over time.
* Improve the quality of the measured boot event log parser.
* Reduce Keylime's dependency on foreign tooling.

### Non-Goals

An explicit non-goal of _this_ proposal is to also enhance the
`tpm2-tools` package by enumerating the issues found during the
rewrite. That (entirely reasonable) work item should be handled
somewhere else.

We do not (in this proposal) aim to change anything else about
measured boot attestation in Keylime. The python event
log parser is designed to produce output comparible with
`tpm2-tools`.

### Notes/Constraints/Caveats (optional)

The most obvious caveat to address is that this project is basically
duplicating code that already exists in `tpm2-tools`, and therefore
could be seen as a duplication of effort, unwise both because of the
wasted labor and the implied long tail of required maintenance.

The developers' argument is that (a) the duplication of effort is not
that great, since it was done in 2 weeks of part-time work, minus the
organizational aspects such as writing this document (b) the long tail
of management is not really that long, because the specifications we
are following are changing very slowly.

In any case we think that the implicit advantages of python-based
development (e.g. native JSON support) and the much shorter
development loop (i.e. the event log parser's developers and the end
users of Keylime overlap) mitigate the disadvantage of code
duplication.

### Risks and Mitigations

The largest risk is that the Python based event loop stops being
maintained, develops bit rot and contributes to a decline of the
quality of the Keylime code base. Mitigation lays in the decision to
continue to use JSON as the exchange format between the parser and the
Keylime measured boot policy agent, which would make a move *back* to
`tpm2-tools` possible at least in principle.

There are no serious security aspects regarding the event log
parser. To the extent that the measured boot event log is
self-checking (i.e. event digests can be recalculated for certain
types of events), the event log parser is already doing so. NB we are
not aware of the `tpm2-tools` based event log parser doing any
self-checking. And of course the python based event log parser is
capable of generating PCR reference values in the same way
as `tpm2-tools`.

Corrupt or maliciously crafted binary event logs have the capability
to disrupt the parser (insufficient self checks have been implemented
to date). To the best of our knowledge such events merely result in
the parsing being aborted with an python Exception.

## Design Details

The parser is implemented as a single Python file, `eventlog.py`. It
includes an `EventLog` class, which is `list` of `Event` objects.

An `EventLog` object can be instantiated with a binary buffer
containing the binary event log, and becomes a list of the individual
events in the log.

`Event` is actually a class hierarchy organized by the TCG list of
event types. Every Event has an attached list of digests, can be
natively parsed into JSON, and has a self-verification method (which
is a noop for certain event types).

Currently only three operations are implemented on the EventLog class
itself: (a) parse into JSON, (b) self-verification and (c) generate a
list of PCRs for authentication against a TPM quote.

Typical use case for the EventLog class:

```
    args = parser.parse_args()
    assert args.file, "file argument is required"

    with open (args.file, 'rb') as fp:
        buffer = fp.read()
        evlog = eventlog.EventLog(buffer, len(buffer))
        print(json.dumps(evlog, default=lambda o: o.toJson(), indent=4))
```

The current implementation is somewhat lacking in buffer overflow
checks (these would only happen if the binary event log is corrupted
or crafted maliciously). Exception handling is also somewhat lacking.


### Test Plan

The event log parser has its own test suite (10 different event logs
collected from real machines). The comparison is performed against
`tpm2_eventlog` from `tpm2-tools` version 5.5.

Attached is example output from the CI system:

```
+------------------------------+-------+-------+-------+----
|    log file                  | #Evts | #Fail |  Pct. | msg.
+------------------------------+-------+-------+-------+----
|bootlog-5.0.0-rhel-20210423T13|     34|      0|  0.00%|
|thyme-eventlog.bin            |     39|      0|  0.00%|
|p511.bin                      |     58|      0|  0.00%|
|swtpm_measurements.bin        |    104|      0|  0.00%|
|puiterwijk.bin                |    101|      0|  0.00%|
|201123-eventlog.bin           |     60|      0|  0.00%|
|ideapad1.bin                  |     39|      0|  0.00%|
|inspur.bin                    |    101|      0|  0.00%|
|rk049-s26.bin                 |     47|      0|  0.00%|
|intel_svr.bin                 |    119|      0|  0.00%|
|css-flex14vm4-bootlog.bin     |    106|      1|  0.94%|
+------------------------------+-------+--------+------+
|     Totals:                  |    808|      1|  0.12%|
+------------------------------+-------+--------+------+
```

TODO: integration testing
TODO: antagonistic testing (broken event logs)
TODO: testing of PCR generation
TODO: self-check testing
TODO: coverage testing

### Upgrade / Downgrade Strategy

TODO

### Dependency  requirements

No major dependencies we are aware of, beyond Python itself. There is
an optional dependency on `libefivar`, which would be used by means of
a `CDLL` call in Python.

Of course, for Keylime itself to use this code, it would have to be
packaged first.

## Drawbacks

"Why should this enhancement _not_ be implemented?" -- the most
serious argument against this code is the duplication of effort
(discussed in detail earlier).

## Unresolved problems at the time of writing this proposal

* TRANSFER: transferring the current Python-native event log parser into the
Keylime project. The current implementation
(https://github.com/galmasi/python3-uefi-eventlog) has its own CI
tooling based on Github Actions, which includes pylint, python type
checking and extensive functional testing against `tpm2_eventlog` by
comparing JSON outputs for a collection of 10 different event logs.

* TESTING: The developer[s] would absolutely welcome contributions of
other binary event logs for even more extensive functional
testing. Including maliciously crafted binary event logs designed to
trip up the parser.

* PACKAGING: There is no packaging code implemented in the project at
this point. The developer[s] would welcome both help with the `pypy`
packaging code and with Red Hat/Canonical packages.

* PEER REVIEW: The code was written mostly by a single person. A peer
review of the code would be welcome.

* DOCUMENTATION: A lot of the event log parser is based on TCG
documentation. Better documentation (attributing parts of the code to
specific TCG and EFI documentation, by chapter and verse) would be
welcome.

* INTEGRATION: a PR in keylime to replace the `tpm2-tools` call with
the native call. Goal 1 would be a pre-requisitve, since the
implication would be that the python event log parser is a package
dependency of keylime.

