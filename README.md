# HSE RFCs

This repo tracks RFCs (request for comments) for all projects under
[`hse-project`](https://github.com/hse-project).

## Purpose

Although HSE uses issues and pull requests to implement and coordinate
development, some changes require more thorough consideration with an
opportunity for various stake holders to provide input on how the proposed
changes affect them. The RFC (request for comments) process intends to provide a
consistent and controlled way of developing HSE for the benefit of all.

## When to use an RFC

When determining whether you should submit an RFC, use the following criteria:

- Any backwards-incompatible change including breaks of either the API or the
  ABI
- Additions to the API
- Major or significant internal changes
- Changes which would significantly and negatively impact performance
  characteristics including throughput, latency, read/write-amplification, and
  memory (DRAM) consumption
- Changes to the RFC process

The following cases most likely do not require an RFC:

- Changes that are likely to only be visible to HSE developers themselves, not
  user-facing
- Minor reorganization or refactoring of code
- Bug fixes
- Additions that would reduce and strictly improve numerical criteria like
  warnings, speed optimization, binary sizes, etc.

## Before Submitting an RFC

We value the opinion of all those willing to contribute to HSE and our major
stakeholders. In order to make sure your RFC has the largest chance of being
accepted, it is important to take your time. Evaluate whether the topic of your
RFC has been proposed before and been previously rejected. It is also important
to solicit maintainer feedback early on in your ideation process. Open a
[discussion](https://github.com/hse-project/hse/discussions) on the main
[HSE repository](https://github.com/hse-project/hse) about your idea with the
"ideas" tag set, and maintainers can give their opinion on whether or not your
RFC is aligned to the goals of the project and likely to be accepted.

## RFC Process

In order to submit an RFC, please adhere to the following guidelines:

1. Gather opinions of HSE maintainers as described above.
1. Open a tracking issue in this repository regarding the RFC.
1. Fork this repo.
1. Copy the template located at the root of this repository to `rfcs/$project`
   where `$project` is what part of the HSE ecosystem your RFC pertains to.
1. Rename the file to `(issue-number)-(lowercase-title-of-rfc).md`. Leave the
   issue number with leading zeros if necessary.
1. Begin filling out the template content.
1. Once completed, open a PR from your fork to this repository. Put in the
   description of the PR

### Lifecycle

Now that the RFC has been submitted, stakeholders, maintainers, and developers
will have the opportunity to comment and suggest changes to make the RFC the
best it can be. Please iterate on any comments that may be given, and feel free
to discuss why certain comments were made.

Once all comments have been resolved and discussion has reached a point of
consensus, the author will mark the corresponding issue in this repository as
ready for a final comment period. This final comment period signifies to others
that a decision will be made about the RFC within 7 business days.

After any potential discussion has been resolved in the final comment period,
the maintainers of the project will decide on whether or not to accept the RFC.

In the case the RFC has been accepted, the PR will be merged. The RFC being
accepted means that the implementation will be prioritized by the maintainers.
External contributors or stakeholders are welcome to implement the RFC
themselves if the work is not prioritized to their liking. The accepted RFC will
then get a tracking issue on the repository of the project with a link to the
RFC (TODO: create a github issue template for this).

In the case the RFC has been denied, the PR will be declined. It is important to
understand that not all RFCs can be accepted. Reasons for declining an RFC could
be could be misalignment with project goals, negative impact to performance, or
API/ABI breakage, among others.
