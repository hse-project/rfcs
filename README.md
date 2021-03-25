# HSE RFCs

This repo tracks RFCs (request for comment) for all project under the
[`hse-project`](https://github.com/hse-project).

## Purpose

Although HSE uses issues and pull requests to implement and coordinate
development, some changes require more thorough consideration with an
opportunity for various stake holders to provide input on how the proposed
changes affect them. The RFC (request for comments) process intends to provide
a consistent and controlled way of developing HSE for the benefit of all.

## When to use an RFC

When determining whether you should submit an RFC, use the following criteria:

- Any backwards-incompatible change including breaks of either the API or the
  ABI
- Additions to the API
- Major or significant internal change
- Changes to the RFC process

The following cases most likely do not require an RFC:

- Changes that are likely to only be visible to HSE developers themselves,
  not user-facing
- Minor reorganization or refactoring of code
- Additions that would reduce and strictly improve numerical criteria like
  warnings, speed optimization, binary sizes, etc.

## Before Submitting an RFC

We value the opinion of all those willing to contribute to HSE and our major
stakeholders. In order to make sure your RFC has the largest chance of being
accepted, it is important to take your time. Evaluate whether the topic of your
RFC has been proposed before and been previously rejected. It is also important
to solicit maintainer feedback early on in your ideation process. Open a
[discussion](https://github.com/hse-project/hse/discussions) on the main
[HSE repository](https://github.com/hse-project/hse) about your idea, and
maintainers can give their opinion on whether or not your RFC is worth pursuing.

## RFC Process

In order to submit an RFC, please follow the following guidelines:

1. Gather opinions of HSE maintainers in order to make sure the RFC is not dead
  on arrival.
1. Fork this repo.
1. Copy the template located at the root of this repository to `rfcs/$project`
  where `$project` is what part of the ecosystem your RFC pertains to.
1. Rename the file...TBD on template name format.
1. Begin filling out the template content.
1. Once completed, open a PR from your fork to this repository.

### Lifecycle

Now that the RFC has been submitted, stakeholders, maintainers, and developers
will have the opportunity to comment and suggest changes to make the RFC the
best it can be. Please iterate on any comments that may be given, and feel free
to discuss why certain comments were made.

Once all comments have been resolved and discussion has reached a point of
consensus, the RFC will enter a final comment period. This final comment period
signifies to others that a decision will be made about the RFC soon (TODO:
should this be a hard length of time?).

After any potential discussion has been resolved in the final comment period,
the maintainers of the project the RFC pertains to will vote on whether your RFC
has been accepted. In this case, a successful vote means a quorum of maintainers
voted yes on the RFC (TODO: should this be all maintainers? should major
stakeholders get a vote like red hat if HSE+Ceph works out?).

In the case the RFC has been accepted, the PR will be merged. The RFC being
accepted means that the implementation will be prioritized around existing work
and work that is coming up on the road-map (TODO: link to a public road-map?).
External contributors or stakeholders are welcome to implement the RFC
themselves if the work is not prioritized to their liking. The accepted RFC will
then get a tracking issue on the repository of the project with a link to the
RFC (TODO: create a github issue template for this).

In the case the RFC has been denied, the PR will be declined. It is important to
understand that not all RFCs can be accepted. Reasons for declining an RFC could
be feature bloat, a disagreement with the final approach, or scope.
