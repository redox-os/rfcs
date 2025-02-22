# Redox RFCs
[Redox RFCs]: #redox-rfcs

Many changes, including bug fixes and documentation improvements can be implemented and reviewed via the normal GitLab merge request workflow.

Some changes though are "substantial", and we ask that these be put through a bit of a design process and produce a consensus among the Redox community.

The "RFC" (request for comments) process is intended to provide a consistent and controlled path for new features to enter, so that all stakeholders can be confident about the direction the OS is evolving in.

## When you need to follow this process
[When you need to follow this process]: #when-you-need-to-follow-this-process

You need to follow this process if you intend to make "substantial" changes to
Redox, Cargo, Crates.io, or the RFC process itself. What constitutes a
"substantial" change is evolving based on community norms and varies depending
on what part of the ecosystem you are proposing to change.

Some changes do not require an RFC:

   - Rephrasing, reorganizing, refactoring, or otherwise "changing shape
does not change meaning".
   - Additions that strictly improve objective, numerical quality
criteria (warning removal, speedup, better software compatibility, more
parallelism, trap more errors, etc.)
invisible to users-of-redox.

If you submit a merge request to implement a new feature without going
through the RFC process, it may be closed with a polite request to
submit an RFC first.

## Before creating an RFC
[Before creating an RFC]: #before-creating-an-rfc

A hastily-proposed RFC can hurt its chances of acceptance. Low quality
proposals, proposals for previously-rejected features, or those that
don't fit into the near-term roadmap, may be quickly rejected, which
can be demotivating for the unprepared contributor. Laying some
groundwork ahead of the RFC can make the process smoother.

Although there is no single way to prepare for submitting an RFC, it
is generally a good idea to pursue feedback from other project
developers beforehand, to ascertain that the RFC may be desirable:
having a consistent impact on the project requires concerted effort
toward consensus-building.

As a rule of thumb, receiving encouraging feedback from long-standing project developers, is a good indication that the RFC is worth pursuing.

## What the process is
[What the process is]: #what-the-process-is

In short, to get a major feature added to Redox, one must first get the
RFC merged into the RFC repo as a markdown file. At that point the RFC
is 'active' and may be implemented with the goal of eventual inclusion
into Redox.

* Fork the RFC repo http://gitlab.redox-os.org/redox-os/rfcs
* Copy `0000-template.md` to `text/0000-my-feature.md` (where 'my-feature' is
descriptive. don't assign an RFC number yet).
* Fill in the RFC. Put care into the details: RFCs that do not present
convincing motivation, demonstrate understanding of the impact of the design, or
are disingenuous about the drawbacks or alternatives tend to be poorly-received.
* Submit a merge request. As a merge request the RFC will receive design feedback
from the larger community, and the author should be prepared to revise it in
response.
* RFCs rarely go through this process unchanged, especially as alternatives and
drawbacks are shown. You can make edits, big and small, to the RFC to
clarify or change the design, but make changes as new commits to the PR, and
leave a comment on the PR explaining your changes. Specifically, do not squash
or rebase commits after they are visible on the PR.
<!--
* Once both proponents and opponents have clarified and defended positions and
the conversation has settled, the RFC will enter its *final comment period*
(FCP). This is a final opportunity for the community to comment on the PR and is
a reminder for all members of the sub-team to be aware of the RFC.
* The FCP lasts one week. It may be extended if consensus between sub-team
members cannot be reached. At the end of the FCP,  the [sub-team] will either
accept the RFC by merging the merge request, assigning the RFC a number
(corresponding to the merge request number), at which point the RFC is 'active',
or reject it by closing the merge request. How exactly the sub-team decide on an
RFC is up to the sub-team.
-->

## The RFC life-cycle
[The RFC life-cycle]: #the-rfc-life-cycle

Once an RFC becomes active then authors may implement it and submit
the feature as a merge request to the Redox repo. Being 'active' is not
a rubber stamp, and in particular still does not mean the feature will
ultimately be merged; it does mean that in principle all the major
stakeholders have agreed to the feature and are amenable to merging
it.

Furthermore, the fact that a given RFC has been accepted and is
'active' implies nothing about what priority is assigned to its
implementation, nor does it imply anything about whether a Redox
developer has been assigned the task of implementing the feature.
While it is not *necessary* that the author of the RFC also write the
implementation, it is by far the most effective way to see an RFC
through to completion: authors should not expect that other project
developers will take on responsibility for implementing their accepted
feature.

Modifications to active RFC's can be done in follow-up PR's. We strive
to write each RFC in a manner that it will reflect the final design of
the feature; but the nature of the process means that we cannot expect
every merged RFC to actually reflect what the end result will be at
the time of the next major release.

In general, once accepted, RFCs should not be substantially changed. Only very
minor changes should be submitted as amendments. More substantial changes should
be new RFCs, with a note added to the original RFC.

## Implementing an RFC
[Implementing an RFC]: #implementing-an-rfc

Some accepted RFC's represent vital features that need to be implemented right away.
Other accepted RFC's can represent features that can wait until some arbitrary developer feels like doing the
work.
Every accepted RFC has an associated issue tracking its implementation in the Redox repository; thus that associated issue can be assigned a priority that the team uses for all issues in the Redox repository.

The author of an RFC is not obligated to implement it.
Of course, the RFC author (like any other developer) is welcome to post an implementation for review after the RFC has been accepted.

If you are interested in working on the implementation for an 'active' RFC, but cannot determine if someone else is already working on it, feel free to ask (e.g. by leaving a comment on the associated issue).


## RFC Postponement
[RFC Postponement]: #rfc-postponement

Some RFC merge requests are tagged with the 'postponed' label when they are closed (as part of the rejection process).
An RFC closed with “postponed” is marked as such because we want neither to think about evaluating the proposal nor about implementing the described feature until some time in the future, and we believe that we can afford to wait until then to do so.
Postponed PRs may be re-opened when the time is right.

Usually an RFC merge request marked as “postponed” has already passed
an informal first round of evaluation, namely the round of “do we
think we would ever possibly consider making this change, as outlined
in the RFC merge request, or some semi-obvious variation of it.”  (When
the answer to the latter question is “no”, then the appropriate
response is to close the RFC, not postpone it.)


### Help this is all too informal!
[Help this is all too informal!]: #help-this-is-all-too-informal

The process is intended to be as lightweight as reasonable for the
present circumstances. As usual, we are trying to let the process be
driven by consensus and community norms, not impose more structure than
necessary.

#### This text

This text is originally based on an older version of the README from https://github.com/rust-lang/rfcs .
