# Mintigate guest tasks

This repository receives guest task requests. An invited guest asks, in one issue, for one small change to one of the public repositories they were authorised for. The answer is one comment on that issue: the result of the checks and, when it is verified, the patch. Nothing is pushed to any repository, and no pull request is opened for you.

The pilot is not open yet. Until it is, every request is refused before anything runs.

## Who can ask

Only invited guests, each for the public repositories listed in their invitation, until the invitation's end date and within its quotas (a number of runs over the pilot and per day, and a daily total for all guests). Each request is launched at most once: to try again, open a new issue.

A request from anyone else, on another repository, after the end date or over a quota, gets a refusal comment that says why. Nothing else runs for it.

## How to ask

Open an issue with the **Guest task** form and fill in every field:

| Field | What to write |
|---|---|
| Repository | The public repository to change, as `owner/name`. |
| Branch | The branch to start from (default `main`). Its current head is used. |
| Commit | Optional: the full commit you expect that head to be. If the branch has moved, the request is refused. |
| Task | What should change, in plain words, 2,000 characters at most. Describe a change to make: a read-only review is not available. |
| Files to change | One path per line. Only these files are sent to the model, and only these files may change. Files under `knowledge/` or `qa/` are not accepted. No file can be created or deleted. |
| Acceptance criteria | One per line, 10 at most. |
| Checks | The npm scripts of your repository to run on the patch, one name per line, 5 at most (for example `test` and `lint`). |
| Confirmations | Tick both boxes. |

Never put a secret, token, password or private information in a request. A request that looks like it contains a credential is refused before anything is sent.

## What happens

1. **Intake.** The request is read as data and checked against the invitations and quotas. No model is called, nothing is cloned and nothing runs. An accepted request is recorded as launched before the next step starts.
2. **Proposal.** The repository is cloned without credentials at the branch head. One call is made to the Claude API (Messages API), with the text below, your task and the full text of the files you listed, and nothing else. The answer may only edit those files, with exact text replacements; anything else is refused and nothing is written. There is no retry, no repair and no second call. None of your repository's scripts run in this step.
3. **Checks.** In a separate job that holds no credential, a fresh clone receives the patch. Before anything runs, the files are compared with the accepted change: the listed files must hold exactly the accepted content and nothing else may differ. Dependencies are installed with `npm ci --ignore-scripts` when a `package-lock.json` exists (so no install script runs), then each requested check runs as `npm run <name>`, as a separate unprivileged user, in an environment built from nothing. Your scripts have network access and run on a hosted runner. After the checks, the files are compared with the accepted change again.
4. **Report.** The check results are tied to this run, this commit and this patch; the patch is applied to another fresh clone and must give exactly the accepted content. Then one comment is posted on your issue.

## The text the model receives

The model receives this text, word for word, then your task and files. It receives no other instruction, document or file.

```text
You are a code-change assistant working on one task in a public repository.
You receive the task, its acceptance criteria, the names of the checks that will run, and the full text of the only files you may change. Use only that material.
Return one JSON object that follows the response schema. Propose changes only to the files you were given, as MODIFY operations with exact text edits: set content to null and give edits as [{find, replace}]. Copy every find string character for character from the given file, so that it matches exactly once; prefer the shortest unique single line. Never rewrite a whole file, never create or delete a file, and never invent a file you were not given.
If you cannot make a correct exact edit, propose no edit for it and explain why in uncertainties.
Do not claim that you ran, tested or verified anything: the checks run later, apart from you.
Keep the change minimal and limited to what the task asks.
```

```text
Required keys: schema_version ("1.0"), capability_id ("website.frontend"), summary (one string), findings, proposed_changes, verification_plan, risks, uncertainties, source_refs, human_decisions_required, handoff_notes (each an array of strings except proposed_changes), review_outcome ("NOT_APPLICABLE"), reviewed_change_paths ([]).
Each proposed_changes item is {path, operation: "MODIFY", rationale, instructions, content: null, edits: [{find, replace}], verification: [strings]}.
source_refs may only name files you were given.
```

The request also carries a JSON schema of that same answer format.

## What you receive

One comment on your issue, with one of these results:

- **ACCEPTED**: every requested check passed on the patch, and the patch holds exactly the accepted content. The patch is in the comment; apply it with `git apply`.
- **NOT_ACCEPTED**: the run produced a result that is not accepted (a check failed, did not exist or timed out, or a file changed during the checks). The findings say why. When the patch itself was verified, it is shown too.
- **REFUSED**: the request was refused before anything ran; no model was called.
- **FAILED**: the run stopped before it produced a patch.
- **WITHHELD**: the result is held back for a manual review before anything is published.

The comment also gives the model's own summary and uncertainties, and its token usage. Logs and the system's own records stay private.

The patch in the comment is exactly the one that was checked, byte for byte; runner paths are masked in the explanations only, never in the patch. When the patch cannot be published exactly as it is (it names a private path of the run, carries a passage of a protected text, is not plain UTF-8 text with Unix line endings, or is too long), the comment says it is withheld and why; it is never edited to make it publishable.

No result is a review or an approval. Applying the patch, opening a pull request and merging it are your decisions.

## Limits

- Public GitHub repositories only, and only those of your invitation.
- Checks are npm scripts of your repository; without a `package-lock.json`, no dependency is installed.
- One request, one model call, one comment. A patch longer than 50,000 characters is withheld, not cut.
- The task, the criteria and the listed files are sent to the Claude API. Everything in the issue, and the result, is public.
