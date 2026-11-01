# Contributing

This repository is the specification side of a clean room (see the README). Its content
changes only through pull requests that the owner merges, after `sacd-spec-pr-check` passes.
Every change to a specification records what was consulted in [PROVENANCE.md](PROVENANCE.md).

## Issues

Most issues here are questions raised while implementing the specification: a point that
is ambiguous, missing, or contradicts another. Each is answered by a change to the text, or by
an explanation of why the text already answers it. Implementers raise them as issues here,
never by pointing to another program's code.

### Milestones

The specification has no numbered releases; implementations cite the commit they follow.
Milestones therefore name what an answer is needed for:

| Milestone | Meaning |
|---|---|
| `1.0` | needed before scarletbook-rs 1.0 freezes its interface |
| `Later` | not planned for a particular release |

Every open issue has one of them, except umbrella and design issues, which have none and carry
the `tracking` label.

### Labels

Every open issue has one priority, the status that applies, and one model label. The type
labels (`bug`, `enhancement`, `documentation`, `question`, ...) stay as they are.

| Label | Meaning |
|---|---|
| `P1` | must be answered in its milestone |
| `P2` | planned for its milestone; can slip one |
| `P3` | nice to have |
| `status: in progress` | claimed and being worked on |
| `status: in review` | a pull request is open |
| `status: fixed on main` | the answer is merged but the issue is still open, for example while a follow-up question is settled; most issues close when their pull request merges instead |
| `status: needs decision` | waiting on the owner |
| `tracking` | an umbrella or design issue that tracks other work |
| `model: opus` | hard work: design, cross-chapter changes, problems nobody can yet explain |
| `model: sonnet` | routine work: well-specified edits, documentation, process |

### Claiming

Before starting on an issue, an agent checks that nobody has claimed it. Then:

1. It comments `Claimed by <agent> (<model>)` and adds `status: in progress`.
2. If it stops without a pull request, it comments that it is releasing the issue and removes
   that label.
3. When its pull request opens, it changes the label to `status: in review`. The pull request
   says `Closes #N` when it fully answers the issue, else `Part of #N`.
4. Nobody starts an issue that someone else has claimed; ask on the issue instead.

The specification has no releases: implementations read `main`, so a merged answer has
shipped. An issue therefore closes when the pull request that fully answers it merges. If
an implementation later finds a problem with that answer, reopen the issue, or open a new
one that links to it. An issue that is only partly answered gets `status: fixed on main`
until the rest is merged.
