# Security policy

This repository holds a specification and test data, not software. A security problem
here is a place where the specification lets a conforming reader be harmed by a hostile
disc image: a missing limit or check, an unbounded loop or allocation, or a corpus case
that is wrong about the outcome it expects.

## Reporting

Please don't open a public issue for a problem that affects readers already written from
this specification. Report it privately through GitHub's private vulnerability reporting:

**<https://github.com/aclater/sacd-spec/security/advisories/new>**

Include the section and item number, what a reader that follows it does with the input,
and, if you have one, a disc image that shows it (a synthetic one is best: please don't
send copies of commercial discs).

Gaps that don't put readers at risk (an ambiguous sentence, a missing example) are
welcome as ordinary issues.
