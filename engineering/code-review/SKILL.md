---
name: code-review
description: Review a GitHub pull request for high-confidence bugs and for violations of the repository's own written rules (AGENTS.md, CLAUDE.md, CONTRIBUTING.md and equivalents), validating every finding before it is reported. Reports to the user by default and posts inline PR comments only when asked. Use when asked to review a pull request, code-review a PR, or review and comment on a PR.
compatibility: Requires a shell, git, a local clone of the repository under review, and an authenticated GitHub CLI (gh). Uses subagents and per-pass model selection when the agent provides them; neither is required.
---

# Pull request code review

Review one GitHub pull request and report only issues that are certainly real: code that will not compile, logic that is wrong for every input, a security hole the change certainly opens, and clear violations of the rules the repository wrote down in its instruction files (`AGENTS.md`, `CLAUDE.md`, `CONTRIBUTING.md` and the like; step 2 lists them). A reviewer who is right every time gets read. One false positive costs more trust than a missed nitpick earns, so when unsure, leave it out.

## Inputs

- **Pull request**: a number or URL. If none is given, use the pull request of the current branch (`gh pr view` with no argument resolves it).
- **Mode**: `report` (default) or `comment`. In `report` mode nothing is written to GitHub. Use `comment` mode only when the user explicitly asked for the review to be posted, for example "review and comment" or a `--comment` argument.

## Requirements

- git, an authenticated `gh` CLI, and a local clone of the repository under review. Run every command from the root of the clone. Use `gh` for everything on GitHub rather than fetching github.com pages: it carries the user's authentication and returns structured data.
- Review the pull request as it is at its head commit, `headRefOid` from step 1. Read every file, code and instruction files alike, through git at that commit: `git show <headRefOid>:<path>`, adding `| cat -n` when you need line numbers, and `git grep <pattern> <headRefOid>` to search. Do not read or change the user's working tree, which may be on another commit or hold local changes, and do not switch branches in it.
- A pull request that touches many files (more than 15, say) makes one shell call per file the slow part of the review. For one of those you may check the head commit out once, into a temporary directory outside the clone: `git -c core.hooksPath=/dev/null worktree add --detach <temp-dir> <headRefOid>`. The passes then read and search `<temp-dir>` with the agent's own file tools, which can run side by side, in place of `git show` and `git grep`. The `-c` option switches git hooks off for that command, so that checking the pull request out cannot run code it brought with it. Run that command yourself, once, and hand `<temp-dir>` to the passes that may read beyond the diff: a pass never makes a checkout of its own. Change nothing in `<temp-dir>`, or later passes will review code the pull request does not contain, and remove it with `git worktree remove --force <temp-dir>` before you finish, however the review ends.
- Assume `gh` and git work. Do not probe or test tools first and make no exploratory calls; every command should serve a step below.
- If a command fails for a reason you cannot fix from its error message (authentication, missing access, no network), stop and report it instead of working around it.

## How the work is split

Steps 1 and 2 are lookups: run them yourself. Steps 3 to 5 are written as passes: bounded pieces of work, each with its own inputs and its own result.

- If your agent can start subagents, give each pass to its own subagent and start the passes marked *parallel* together. Separate contexts are the point: a reviewer that has seen another reviewer's findings tends to repeat them, and a validator that has seen the finder's reasoning tends to agree with it.
- Keep at most 6 subagents running at once, across all the pull requests when you are reviewing several, and start the rest as those finish. A larger burst runs into the model provider's concurrency limits and comes back as timeouts, which costs more time than the queue does.
- If it cannot, do the passes yourself, in order. Judge each pass on the diff itself and set aside what the pass before concluded.
- A subagent begins knowing nothing. Hand each one what its pass needs: the pull request title and description (they show what the author intended), the command that prints its diff (`git diff <baseOid>..<headRefOid>` from step 2, with the commits filled in), `<temp-dir>` if you made one, the instruction files that apply, the Requirements above, and the "What to flag" and "False positives" sections. Tell it to do its pass directly, without loading this skill and without starting agents of its own: a subagent that picks this skill up again starts the whole review inside itself.
- Each pass names a model tier. *Standard* is a mid-tier model, for summarising and for checking code against written rules. *Deep* is the strongest reasoning model available, for finding and confirming bugs, where a wrong call is expensive. Use the tier if your agent lets you choose a model per pass; otherwise use the model you have.

If your agent has a task list or planning facility, enter the steps in it before starting.

## Steps

### 1. Gate

```
gh pr view <PR> --json number,url,title,body,state,isDraft,author,headRefOid,baseRefName,baseRefOid
gh pr view <PR> --json comments,reviews --jq '[.comments[].body, .reviews[].body] | map(select(contains("<!-- code-review-skill -->"))) | length'
```

Keep `url` (it gives `<owner>/<repo>`), `number`, `headRefOid` (the full SHA of the head commit), `baseRefName` (the branch the pull request targets) and `baseRefOid` (the commit on that branch GitHub compares it with). Later steps use them.

Stop here, and tell the user which condition applied, if any of these is true:

- `state` is not `OPEN`.
- `isDraft` is true.
- The pull request is a mechanical one that needs no review, such as a dependency bump or a release opened by a bot. The title, the description and `author` show this.
- The second command prints a number above 0. This skill has already reviewed the pull request: everything it posts carries the marker `<!-- code-review-skill -->`, which identifies an earlier run whatever account or agent posted it.

A pull request whose code an AI agent wrote is not mechanical and still gets reviewed.

### 2. Find the instruction files

Take the diff from the clone, not from `gh pr diff`, which GitHub refuses for a pull request of more than 300 files. First make sure both ends of the change are in the clone, then find the commit the pull request branched from:

```
git cat-file -e '<headRefOid>^{commit}' && git cat-file -e '<baseRefOid>^{commit}'
git merge-base <baseRefOid> <headRefOid>
```

If the first command exits non-zero, a commit is not in the clone yet: run `git fetch <remote> <baseRefName> pull/<number>/head`, where `<remote>` is the remote that points at `<owner>/<repo>` (usually `origin`), then go on. The second command prints `<baseOid>`, and `<baseOid>..<headRefOid>` is the change under review. If it prints nothing, the clone is shallow and the two histories do not meet in it: run `git fetch --unshallow <remote>` and run it again.

A stacked pull request needs one more check. One that is built on another open pull request, yet targets the branch the whole stack will merge into, carries its parent's changes in that range, and reviewing it would review the parent again. When you are reviewing a stack, or the description or the user names an open parent, get the parent's head with `gh pr view <parent> --json state,headRefOid`. If the parent is `OPEN` and `git merge-base --is-ancestor <parent headRefOid> <headRefOid>` exits 0, use the parent's `headRefOid` as `<baseOid>`, so that each pull request is reviewed only for what it introduces. Otherwise keep the merge base: a parent that is not an ancestor has been rewritten since, and is no base for this diff.

Instruction files are where a repository writes down how to work in it: the files addressed to coding agents, and the contribution and coding-standards documents addressed to people. List the changed files, the size of the change and the candidates:

```
git diff --name-only <baseOid>..<headRefOid>
git diff --shortstat <baseOid>..<headRefOid>
git ls-tree -r --full-tree --name-only <headRefOid> | grep -E '(^|/)(AGENTS|CLAUDE|GEMINI)\.md$|^\.github/copilot-instructions\.md$|^\.claude/rules/.*\.md$|^\.cursor/rules/.*\.mdc$|^\.github/instructions/.*\.instructions\.md$|^(\.github/|docs/)?(CONTRIBUTING|CODING_STANDARDS)\.md$'
```

If the third command prints nothing, the repository has no candidates; that is a result, not a failure.

From the candidates, keep the paths of these; the compliance passes read the contents:

- `AGENTS.md`, `CLAUDE.md` and `GEMINI.md` are directory-scoped. Keep the one at the repository root and any in a directory that contains a changed file, or in a parent of one. Each applies to its own directory and everything below it.
- `.github/copilot-instructions.md`, `CONTRIBUTING.md` and `CODING_STANDARDS.md` (at the root, in `.github/` or in `docs/`) apply to the whole repository.
- Files under `.claude/rules/`, `.cursor/rules/` and `.github/instructions/` are glob-scoped: the frontmatter of each (`paths:`, `globs:`, `applyTo:`) lists globs, relative to the repository root, for the files it covers. A file that sets `alwaysApply: true` covers every file, and so does a file under `.claude/rules/` that lists no globs. Any other file that lists none covers no file. Keep them all; the compliance passes read the frontmatter.
- Any file the user names as a rule source.

A directory-scoped file may import another by writing `@<path>` outside a code span, with the path relative to the file itself: a `CLAUDE.md` whose whole content is `@AGENTS.md`, for example. An import is not a rule. Open each directory-scoped file you kept, add every file in the repository that it imports to the list with the importer's scope, and drop a file that holds nothing but imports. Do not follow ordinary links.

If nothing is left, skip the compliance passes and say so in the report.

### 3. Summarise the change (standard)

Read the pull request (`git diff <baseOid>..<headRefOid>`, with the title and description from step 1) and return a summary of what it changes. The summary is for you: it lets you brief the later passes and weigh their findings without holding the whole diff.

If the summary shows a trivial change that is obviously correct, such as a typo or a comment, stop here and tell the user why.

### 4. Review (four passes, parallel)

Each pass returns a list of issues. Every issue has a description, the file and line range at `headRefOid` (the file alone when the issue is something the change left out), and the reason it was flagged: `bug`, or `instruction-file compliance` with the rule quoted and the path of the file it comes from.

- **Passes A and B, compliance (standard).** Audit the changes against the instruction files from step 2. Judge a changed file only against the instruction files whose scope covers it: its directory for a directory-scoped file, its frontmatter for a glob-scoped one. Only a rule the change itself can break counts: a rule about process outside the diff, such as how to branch or what to run before pushing, is not a finding. The two passes do the same work independently, which helps only when each has its own context. If you are running the passes yourself in one context, run this once. When you are reviewing several pull requests at once you may also run it once for each: the repeat is the cheapest pass to give up when they share the 6 subagents.
- **Pass C, bugs in the diff (deep).** Scan for obvious bugs, looking only at the diff and reading no further context. Flag significant bugs only; skip nitpicks and likely false positives. Do not flag anything you cannot confirm from the diff alone.
- **Pass D, problems in the new code (deep).** Look for problems in the code the pull request introduces, such as incorrect logic or a security hole. It may read any surrounding code it needs, but it reports only issues that sit in the changed code.

When the change is large (more than about 1,500 changed lines or 30 files), you may split passes C and D by directory or module, each part going to its own subagent with `git diff <baseOid>..<headRefOid> -- <paths>` as its diff: a reviewer handed all of a large diff reads the end of it less closely than the start. Put every changed file in exactly one part. A part of pass D may still read any code it needs outside its own paths.

All four passes apply "What to flag" and "False positives" below.

### 5. Validate (one pass per issue, parallel)

Merge issues from step 4 that describe the same problem, then run a validation pass for each one. It receives the pull request title and description and the issue as step 4 returned it (description, location and reason), but not the reasoning of the pass that found it, and it must confirm with high confidence that the issue is real. For "variable is not defined", that means reading the code and confirming it is not defined. For a compliance issue, it means confirming that the rule's scope covers the file and that the change really breaks the rule. It may read anything in the repository at `headRefOid`. Use the deep tier for bugs and logic and the standard tier for compliance.

With more than 4 issues, do not give each its own subagent: a long list would start dozens of them to read the same files. Give each subagent up to 4 issues from the same file or module: it validates each one separately and returns a verdict for each. Keep `bug` issues and compliance issues in separate groups, because they run on different tiers, and never put two issues about the same lines in one group, where the verdict on the first would colour the second.

If you are running without subagents, validate by going back to the code and trying to disprove each issue.

### 6. Filter

Drop every issue that step 5 did not confirm. What remains is the review.

### 7. Report

Tell the user the result:

- If issues remain, list each one with its location and a brief description.
- If none remain, say: "No issues found. Checked for bugs and instruction-file compliance." If step 2 found no instruction files, the second sentence is "Checked for bugs; the repository has no instruction files."

In `report` mode, stop here.

In `comment` mode with no issues, write this text to a temporary file, using the same second sentence you gave the user, post it with `gh pr comment <PR> --body-file <file>`, and stop:

```
## Code review

No issues found. Checked for bugs and instruction-file compliance.

<!-- code-review-skill -->
```

In `comment` mode with issues, continue.

### 8. Draft the comments

List every comment you intend to post and read it over. This list is for you: do not post it or show it anywhere.

### 9. Post one review

Post all the comments as a single review, so the author gets one notification and the review lands whole or not at all. Write the payload to a temporary JSON file and send it:

```
gh api repos/<owner>/<repo>/pulls/<number>/reviews --method POST --input <payload-file>
```

```json
{
  "commit_id": "<headRefOid>",
  "event": "COMMENT",
  "body": "## Code review\n\nFound 2 issues, commented inline.\n\n<!-- code-review-skill -->",
  "comments": [
    { "path": "<path>", "line": 42, "side": "RIGHT", "body": "<comment>" },
    { "path": "<path>", "start_line": 10, "start_side": "RIGHT", "line": 14, "side": "RIGHT", "body": "<comment>" }
  ]
}
```

- `line`, and `start_line` for a range, are line numbers in the file at `headRefOid`, and they must be lines that appear in the pull request's diff. If GitHub answers 422, the review was not accepted: read the message, fix what it names (usually a `line` that is not in the diff), and send the request again.
- If the request ends in `502`, `503`, `504` or a timeout, do not send it again yet: GitHub often saves the review even though its gateway gave up, and a blind retry posts it twice. Wait a minute, then run the marker check from step 1 again. A number above 0 means the review is posted: stop. If it prints 0, send the request once more, and if that times out too, go to "When the request keeps timing out" at the end of this step.
- An issue with no line in the diff to anchor to, such as something the pull request should have changed and did not, goes in the review `body`, after the "Found" sentence and before the marker.

For each comment:

- Describe the issue briefly.
- For a small, self-contained fix, include a committable suggestion: a fenced code block whose language is `suggestion`. It replaces exactly the lines the comment is anchored to.
- For a larger fix (6 or more lines, a structural change, or changes in several places), describe the issue and the fix in words, with no suggestion block.
- Include a suggestion only when committing it fixes the issue completely. If anything else would have to follow, describe the fix instead.
- Post one comment per distinct issue, with no duplicates.
- Cite and link what the comment relies on. For a compliance issue, quote the rule and link the instruction file it is in.

Links to code or to an instruction file must have exactly this form, or GitHub will not render the preview:

```
https://github.com/<owner>/<repo>/blob/<full-sha>/<path>#L<start>-L<end>
```

- `<full-sha>` is the complete 40-character commit SHA, written out. The comment is rendered as Markdown exactly as posted, so shell substitution such as `$(git rev-parse HEAD)` does not work. Use `headRefOid`.
- `<owner>/<repo>` is the repository under review.
- The line range follows the file name after `#`, as `L<start>-L<end>`.
- Include one line of context on each side of the lines you mean, where the file has one: for lines 5 and 6, link `L4-L7`.
- For a Markdown file, put `?plain=1` before the `#`. Without it GitHub shows the rendered page and ignores the line range.

**When the request keeps timing out.** After the second timeout, wait a minute and run the marker check once more. If it still prints 0, the pull request is too large for GitHub to place every comment in one request. Post the same review in parts, as a pending review: nobody sees it or is notified until it is submitted, so it still lands whole and with one notification.

```
gh pr view <PR> --json id --jq .id
gh api graphql -f query='query($pr:ID!){node(id:$pr){... on PullRequest{reviews(states:PENDING,first:1){nodes{id comments(first:100){nodes{path line}}}}}}}' -f pr=<pr-id>
gh api graphql -f query='mutation($pr:ID!,$oid:GitObjectID!){addPullRequestReview(input:{pullRequestId:$pr,commitOID:$oid}){pullRequestReview{id}}}' -f pr=<pr-id> -f oid=<headRefOid>
gh api graphql -f query='mutation($review:ID!,$path:String!,$line:Int!,$body:String!){addPullRequestReviewThread(input:{pullRequestReviewId:$review,path:$path,line:$line,side:RIGHT,body:$body}){thread{id}}}' -f review=<review-id> -f path=<path> -F line=<line> -F body=@<comment-file>
gh api graphql -f query='mutation($review:ID!,$body:String!){submitPullRequestReview(input:{pullRequestReviewId:$review,event:COMMENT,body:$body}){pullRequestReview{url}}}' -f review=<review-id> -F body=@<body-file>
```

- The second command lists the pending review the `gh` user has on the pull request, with the comments it holds. Run it before anything else. GitHub allows one pending review per user, so one that is there before you create yours is the user's own draft: stop and report, because submitting it would publish their unfinished comments.
- The third command creates the review. Give it the commit and nothing else: with no `event` it stays pending, and its `body` comes when you submit.
- The fourth attaches one comment; run it for each. Anchor every comment to a single line, the last of its range, which leaves GitHub one line to place per call. A suggestion block replaces only the anchored line here, so a comment whose suggestion covers more than one line loses the block and describes the fix in words.
- Any of these calls can time out after it has taken effect, as the first request may have. Before you repeat one, run the second command again and send only what the review does not hold yet.
- The fifth submits the review with the `body` of the payload above. If it times out, run the marker check before you repeat it.
- If a comment cannot be attached, delete the pending review (`deletePullRequestReview` with its id) and report, rather than submit a review with a comment missing.

## What to flag

Flag an issue only when it is one of these:

- The code will fail to compile or parse: syntax errors, type errors, missing imports, unresolved references.
- The code will produce wrong results whatever the input: clear logic errors.
- The change certainly opens a security hole, such as removing an authorization check or passing user input into a query unescaped.
- A clear, unambiguous violation of an instruction file, where you can quote the exact rule that is broken.

Do not flag:

- Code style or quality concerns.
- Potential issues that depend on particular inputs or state, including a security weakness that only might be reachable.
- Subjective suggestions or improvements.

If you are not certain an issue is real, do not flag it.

## False positives

Steps 4 and 5 treat all of these as false positives:

- Issues that existed before the pull request.
- Something that looks like a bug but is correct.
- Pedantic nitpicks that a senior engineer would not raise.
- Issues a linter will catch. Do not run the linter to check.
- General code quality concerns, such as missing test coverage or security hardening, unless an instruction file explicitly requires them.
- Issues an instruction file mentions but the code explicitly silences, for example with a lint-ignore comment.
