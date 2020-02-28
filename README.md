# Linkerd GSoC

Welcome to the Linkerd GSoC program!

This repository contains everything you need to know about the program.

## Resources

* Official website: https://linkerd.io/gsoc
* Linkerd slack (`#gsoc` channel): https://linkerd.slack.com

## Request For Comments (RFC)

All GSoC students are required to submit a RFC for the GSoC project that they
are interested in.

The RFC process is intended to provide a consistent and controlled path for
contributions to enter the Linkerd project, so that all stakeholders
can be confident about the project direction and maintainability of the codebase.

### Before creating an RFC

[before creating an rfc]: #before-creating-an-rfc

A hastily-proposed RFC can hurt its chances of acceptance. Low quality
proposals, proposals for previously-rejected features, or those that don't fit
into the near-term roadmap, may be quickly rejected, which can be demotivating
for the unprepared contributor. Laying some groundwork ahead of the RFC can
make the process smoother.

Although there is no single way to prepare for submitting an RFC, it is
generally a good idea to pursue feedback from the GSoC mentors beforehand, to
ascertain that the RFC may be desirable; having a consistent impact on the
project requires concerted effort toward consensus-building.

The most common preparations for writing and submitting an RFC include talking
the idea over on our
[Linkerd Slack, #gsoc channel](https://slack.linkerd.io).

As a rule of thumb, receiving encouraging feedback from long-standing Linkerd
developers is a good indication that the RFC is worth pursuing.

### What the process is

[what the process is]: #what-the-process-is

In short, to get a major feature added to Linkerd, one must first get the RFC
merged into this repository as a markdown file. At that point the RFC is
"active" and may be implemented with the goal of eventual inclusion into
Linkerd.

1. Fork the [GSoC repository](https://github.com/linkerd/gsoc)
2. Copy `templates/rfc.md` to
   `rfc/year/project-name/your-github-handle.md`.
3. Fill in the RFC, starting with `Step 1` - Problem Statement. This is where
   you demonstrate your understanding of the problem to be solved.
4. Submit a pull request. As a pull request the RFC will receive feedback from
   the larger community, and the author should be prepared to revise it in
   response.
5. Each pull request will be labeled with the most relevant reviewer, who will
   lead its triage.
6. Build consensus and integrate feedback. RFCs that have broad support are much
   more likely to make progress than those that don't receive any comments. Feel
   free to reach out to the RFC assignee in particular to get help identifying
   stakeholders and obstacles.
7. The team will discuss the RFC pull request, as much as possible in the
   comment thread of the pull request itself. Offline discussion will be
   summarized on the pull request comment thread.
8. When all the feedback are integrated, update your pull request to complete
   `Step 2` - Design Proposal. Put care into the details: RFCs that do not
   present convincing motivation, demonstrate lack of understanding of the
   design's impact, or are disingenuous about the drawbacks or alternatives tend
   to be poorly-received.

RFCs rarely go through this process unchanged, especially as alternatives and
drawbacks are shown. You can make edits, big and small, to the RFC to clarify or
change the design, but make changes as new commits to the pull request, and
leave a comment on the pull request explaining your changes. Specifically, do
not squash or rebase commits after they are visible on the pull request.

### The RFC life-cycle

[the rfc life-cycle]: #the-rfc-life-cycle

Once an RFC is tagged as "active" then author may implement it and submit the
feature as a series of pull requests to the
[Linkerd2](https://github.com/linkerd/linkerd2) repo. Being "active" is not
a rubber stamp, and in particular still does not mean the feature will
ultimately be merged; it does mean that in principle all the major stakeholders
have agreed to the feature and are amenable to merging it.

Modifications to "active" RFCs can be done in follow-up pull requests. We strive
to write each RFC in a manner that it will reflect the final design of the
feature; but the nature of the process means that we cannot expect every merged
RFC to actually reflect what the end result will be at the time of the next
major release.

In general, once accepted, RFCs should not be substantially changed. Only very
minor changes should be submitted as amendments. More substantial changes should
be new RFCs, with a note added to the original RFC. Exactly what counts as a
"very minor change" is up to the team to decide.

### Implementing an RFC

[implementing an rfc]: #implementing-an-rfc

Every accepted RFC has an associated issue tracking its implementation in the
[Linkerd2](https://github.com/linkerd/linkerd2) repository. The issue will be
assigned to the RFC author.

### Help this is all too informal!

[help this is all too informal!]: #help-this-is-all-too-informal

The process is intended to be as lightweight as reasonable for the present
circumstances. As usual, we are trying to let the process be driven by
consensus and community norms, not impose more structure than necessary.

## License

[license]: #license

This repository is currently licensed under:

- Apache License, Version 2.0, ([LICENSE-APACHE](LICENSE-APACHE) or http://www.apache.org/licenses/LICENSE-2.0)
- https://github.com/linkerd/linkerd2/blob/master/LICENSE

### Contributions

Unless you explicitly state otherwise, any contribution intentionally submitted
for inclusion in the work by you, as defined in the Apache-2.0 license, shall be
licensed as above, without any additional terms or conditions.
