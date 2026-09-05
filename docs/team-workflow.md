# BookHaven — Team Workflow

This document explains how the team coordinates work while members are working
remotely and often at different times. It covers branch ownership, pull
requests, reviews, issues and progress updates.

The aim is that any team member can find out what is happening, and what has
already been decided, without waiting for someone else to be online.

---

## 1. Principles

**Work is visible in the repository.** Decisions taken in a call or a chat are
written back into an issue or a pull request. A conversation only the people
present can see is not a record, and cannot be assessed.

**One person owns each piece of work.** Every task has a single named owner, so
there is never ambiguity about who is expected to move it forward.

**Nothing enters `main` unreviewed.** Every change is reviewed by at least one
other member. This spreads knowledge of the design across the team and means no
single person is the only one who understands part of the project.

**Asynchronous by default.** Team members should not be blocked waiting for
someone in a different time zone or with different commitments. Written updates
and clear issue descriptions make this possible.

---

## 2. Issues: planning and tracking work

Issues are the team's single list of what needs doing, what is in progress and
what is finished.

**Creating an issue.** Anything that needs doing is raised as an issue using the
[task template](../.github/ISSUE_TEMPLATE/task.md), including documentation
sections, diagram updates and questions the team must resolve. Defects in
existing content use the [bug report template](../.github/ISSUE_TEMPLATE/bug_report.md).

A good issue states what is required and how the team will know it is done. "Fix
the architecture doc" is not actionable; "Add the Security Group rules table to
the architecture document, showing source and destination for each tier" is.

**Assignment.** Each issue is assigned to exactly one owner. Assign yourself
before starting, so nobody duplicates your work.

**Labels.** The team uses labels to make the list readable at a glance:

| Label | Meaning |
|---|---|
| `documentation` | Changes to files in `docs/` or the README |
| `diagram` | Work on the Figma diagram or its exports |
| `question` | A decision the team needs to make together |
| `blocked` | Cannot proceed until something else is resolved |
| `bug` | Something in the repository is incorrect |

**Status.** An issue is open until the work is merged. Issues are closed by the
pull request that completes them, using `Closes #12` in the description, so the
issue list always reflects reality.

**Decisions.** When the team agrees something in a meeting or a chat — an AWS
service, a subnet layout, a naming convention — one person records it in the
relevant issue and comments with the outcome. That comment becomes the record
the team can point to later.

---

## 3. Branch ownership

**Each branch has one owner: the person who created it.**

Only the owner commits to that branch. If another member wants to change work in
progress, they comment on the pull request or the issue rather than pushing to
someone else's branch. Unexpected commits on a branch you are working on cause
confusion and can overwrite local work.

**Branch naming** follows the prefixes in [`CONTRIBUTING.md`](../CONTRIBUTING.md):
`docs/`, `diagram/`, `fix/` and `chore/`. Names describe the work, not the
person, so the branch list reads as a list of work in progress.

**Avoiding conflicts.** Two people editing the same file at the same time is the
main source of merge conflicts in a documentation repository. The team avoids
this by:

- Assigning each document a section owner where two people must work in parallel.
- Keeping branches short-lived — days, not weeks.
- Pulling the latest `main` before creating a branch and before opening a pull
  request.

**Handing over.** If you cannot continue with a branch, say so in a comment on
the issue, push what you have, and reassign the issue. Work in progress that is
only on someone's laptop is invisible and cannot be picked up.

---

## 4. Pull requests

A pull request is how work moves from a branch into `main`, and it is also the
record of what changed and why.

**Opening one.** Use the [pull request template](../.github/pull_request_template.md)
and complete every section. Link the issue with `Closes #12`. If the work is not
ready, open it as a **draft** so the team can see progress without being asked
to review unfinished content.

**Size.** Keep pull requests small enough to review carefully. A pull request
touching one document with a clear purpose will be reviewed within a day; one
rewriting five files at once will sit unreviewed.

**Responding to review.** Address every comment, either by making the change or
by explaining your reasoning. Push updates to the same branch — the pull request
updates automatically. Do not open a new pull request to answer review comments.

**Merging.** Merge only after approval, then delete the branch. See
[`CONTRIBUTING.md`](../CONTRIBUTING.md) for the full merge conditions.

---

## 5. Reviews

**Who reviews.** At least one team member other than the author. Rotate
reviewers rather than always asking the same person, so knowledge of the design
is shared across the whole team.

**Turnaround.** Aim to review within one working day. If you cannot, say so on
the pull request so the author can ask someone else instead of waiting.

**What a reviewer checks.**

- The content is accurate and technically correct.
- It is consistent with the architecture document and the diagram.
- It uses only the AWS services on the agreed list.
- No credentials, keys, `.pem` files or `.env` files have been committed.
- The language is clear and appropriate for assessment.
- The pull request template has been completed.

**How to comment.** Be specific and refer to the content, not the person.
"This section says two subnets but the diagram shows four" is useful.
"This is wrong" is not. Where something is unclear rather than incorrect, ask a
question — the answer often belongs in the document itself.

**Approving.** Approve when you are satisfied the change is correct and
consistent. Approving without reading is worse than not reviewing, because it
creates a false record of scrutiny.

---

## 6. Progress updates

Because the team works remotely and asynchronously, written updates replace the
awareness that comes from sitting in the same room.

**Weekly team check-in.** A short scheduled call where each member covers what
they completed, what they are working on next, and anything blocking them.
Decisions taken are written into the relevant issues immediately afterwards by a
nominated note-taker.

**Written updates between calls.** If work on an issue runs across several days,
post a brief comment on that issue. Two or three sentences is enough:

```
Progress: subnet boundary section drafted, Security Group table still to do.
Blocked on: whether we document the exact database port or refer to it
generically — raised as #18.
Next: finish the table and open the pull request tomorrow.
```

**Flagging blockers early.** If you are blocked, say so the same day. Label the
issue `blocked` and state clearly what you are waiting for and from whom. A
blocker that surfaces at the weekly call has already cost the team several days.

**Availability.** Team members note their general availability and time zone in
the team channel, so others know when to expect a response and when to ask
someone else instead.

---

## 7. Evidence of collaboration

The repository is itself the evidence that the team worked together. By the end
of the project it should show:

- Issues describing the work, assigned across the team.
- Branches created by different members.
- Pull requests with completed templates.
- Review comments and approvals from members other than the author.
- A commit history in which contributions are spread across the team.

Following the workflow produces this record as a by-product. Working outside it —
committing directly to `main`, merging without review, or agreeing things only
in chat — leaves the team with a design but no demonstration of how it was
produced.
