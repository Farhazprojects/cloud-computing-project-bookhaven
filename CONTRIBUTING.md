# Contributing to BookHaven

This guide explains how to make a change to this repository. Every team member
follows the same process so that all contributions are visible, reviewed and
recorded.

At this stage the repository contains **documentation only**. Please do not add
application code, deployment scripts, Terraform, CloudFormation, or AWS services
beyond those listed in the README.

---

## Before you start

Make sure the change you intend to make is captured in an issue and assigned to
you. This prevents two people working on the same file at the same time, which
is the most common cause of merge conflicts in a documentation repository.

If your change would alter the agreed architecture or introduce a new AWS
service, raise it as an issue and get team agreement **first**. A pull request
is for reviewing how something was written, not for deciding whether it should
exist.

---

## 1. Create a feature branch

Always branch from an up-to-date `main`. Never commit directly to `main`.

```bash
git checkout main
git pull origin main
git checkout -b docs/architecture-security-section
```

Use the branch prefixes agreed by the team:

| Prefix | Used for |
|---|---|
| `docs/` | Documentation changes |
| `diagram/` | Diagram exports and updates |
| `fix/` | Corrections to existing content |
| `chore/` | Repository housekeeping |

Name the branch after the work, not after yourself, so that anyone reading the
branch list can tell what is in progress.

---

## 2. Make focused changes

Keep each branch to one piece of work. A branch that edits the security section
of the architecture document should not also reorganise the README.

Focused changes are easier to review, easier to discuss and much easier to undo
if the team disagrees with them. If you notice something else that needs doing
while you are working, open an issue for it rather than adding it to the current
branch.

Before committing, check what you are about to include:

```bash
git status
git diff
```

Confirm that no credentials, keys, `.pem` files or `.env` files appear in the
list.

---

## 3. Commit using clear messages

Write commit messages that explain the change to someone reading the history in
three months' time.

**Format**

```
Short summary in the imperative, under 72 characters

Optional body explaining what changed and why, wrapped at 72
characters. Reference the issue this commit relates to.

Relates to #12
```

**Good**

```
Add Security Group rules table to architecture document
Correct Availability Zone count in high availability section
Explain how CloudWatch alarms trigger Auto Scaling
```

**Avoid**

```
update
fixed stuff
changes as discussed
```

Commit at logical points rather than saving everything until the end, so the
history shows how the work developed.

---

## 4. Push the branch

```bash
git push -u origin docs/architecture-security-section
```

The `-u` flag is only needed the first time you push a branch. After that,
`git push` is enough.

---

## 5. Open a pull request

Open the pull request on GitHub from your branch into `main`. The repository
template will load automatically — complete every section:

- **Summary** — what changed and why.
- **Files changed** — the files you touched.
- **Testing or verification** — how you checked the change is correct.
- **Assessment requirement addressed** — which criterion this supports.
- **Reviewer checklist** — leave for the reviewer to complete.

Link the pull request to its issue by writing `Closes #12` in the description,
so the issue closes automatically when the pull request is merged.

If the work is not finished, open the pull request as a **draft**. This lets the
team see progress without implying the work is ready for review.

---

## 6. Request team review

Add at least one other team member as a reviewer, and post the link in the team
channel so it is not missed.

**If you are the author:** respond to every comment, either by making the change
or by explaining why you have not. Push updates to the same branch; the pull
request updates automatically.

**If you are the reviewer:** aim to respond within one working day. Check that
the content is accurate, that it is consistent with the diagram and the rest of
the documentation, that it uses no services outside the agreed list, and that no
sensitive information has been committed. Ask questions where something is
unclear rather than assuming it is wrong.

Reviews are about the work, not the person. Keep comments specific and
constructive.

---

## 7. Merge only after approval

A pull request may be merged when:

- At least one other team member has approved it.
- All review comments have been resolved.
- The pull request template is fully completed.
- The documentation and the diagram are consistent with each other.

Do not merge your own pull request without an approval, even for a small change.
The review record is part of how the team demonstrates collaboration for the
assessment.

After merging, delete the branch and close any related issue that has not closed
automatically.

---

## Security note

Never commit:

- Passwords or database credentials
- AWS access keys or secret access keys
- `.pem` key files or any other private keys
- `.env` files or local configuration containing secrets
- Personal data of any kind

Database credentials for BookHaven are stored in AWS Secrets Manager and
retrieved at runtime by the application through an IAM role, as described in
[`docs/architecture.md`](docs/architecture.md). They never appear in this
repository.

If a credential is committed by mistake, tell the team immediately. Removing the
file in a later commit does not remove it from the history, and the credential
must be treated as compromised and rotated.
