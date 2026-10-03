---
name: intent
description: Turn an issue or idea into an approved intent — improve the prompt, investigate, plan, write a numbered intent file, and draft the GitHub issue for me to file
argument-hint: "[the problem or idea | ADR-NNNN to resume | #issue or issue URL to link — empty to list]"
allowed-tools: Read, Grep, Glob, Bash(git log:*), Bash(git show:*), Bash(git blame:*), Bash(git grep:*), Bash(git diff:*), Bash(git rev-parse:*), Bash(git fetch:*), Bash(git ls-tree:*), Bash(git ls-remote:*), Bash(gh issue list:*), Bash(gh issue view:*), Bash(gh pr list:*), Bash(gh pr view:*), Bash(gh pr diff:*), Bash(gh search issues:*), Bash(gh search prs:*), WebFetch(domain:docs.github.com), WebFetch(domain:git-scm.com), WebFetch(domain:docs.rs), WebFetch(domain:doc.rust-lang.org), WebFetch(domain:docs.python.org), WebFetch(domain:nodejs.org), WebFetch(domain:developer.mozilla.org), WebFetch(domain:go.dev), WebFetch(domain:pkg.go.dev), WebFetch(domain:kubernetes.io), WebFetch(domain:developer.hashicorp.com), WebFetch(domain:registry.terraform.io), WebFetch(domain:docs.aws.amazon.com), WebFetch(domain:cloud.google.com), WebFetch(domain:learn.microsoft.com), WebFetch(domain:datatracker.ietf.org), WebFetch(domain:www.rfc-editor.org), WebFetch(domain:code.claude.com), WebFetch(domain:docs.anthropic.com)
effort: xhigh
disable-model-invocation: true
---
Work out the intent for: $ARGUMENTS

Be my collaborator: improve the request, investigate it with evidence, and agree a plan with me
before anything is written. The only files you write are the intent file (step 4) and, later, its
`Issue:` line and sign-off records (step 6). **Never create the GitHub issue** — I file it myself, and
`gh issue create` is denied in this project's permissions.

Settings for this project (change them with `ocgen edit intent`):
- Intent files: `docs/adr/ADR-<number>-<slug>.md`, the number zero-padded to 4 digits
  (e.g. `docs/adr/ADR-0001-cache-invalidation.md`), structured by `.claude/intent/intent-template.md`.
- Issue description: at most 250 words, structured by `.claude/intent/issue-template.md`.
- **No approvers configured.** Say "No approvers configured" in both drafts, ask me whether to
  proceed without a sign-off, and suggest `ocgen edit intent --approver @handle`.

## 0. Route the arguments
- **Empty** (or `list`): list the intent files in `docs/adr/` that have no `Issue:` line yet — number,
  title, status — newest first, and let me pick one to resume or start a new intent.
- **`ADR-NNNN`** (or a file in `docs/adr/`): resume that intent — summarise it in a few lines
  and ask what we're doing next.
- **`#123` or an issue URL**, or an approver's sign-off (`@alice approved: <link>`): go to step 6.
- **Anything else**: a new intent — start at step 1.

## 1. Improve the prompt
Rewrite my request into a sharp prompt using the prompt-engineering technique: an explicit role, the
task, success criteria, constraints, and the context it needs; remove ambiguity and say what you
inferred. Show the improved prompt and let me confirm or adjust it. Everything after this step works
from the improved prompt.

## 2. Investigate — exhaustively
Be aggressive and complete: the plan is only as good as this analysis. Everything here is
read-only; the git, gh and docs reads listed in this skill's `allowed-tools` run without prompts.
Number every finding **F1, F2, …** and tag it **Verified** (with `file:line`, a commit, or a link)
or **Inferred** (with `Confidence: NN%` and what would verify it).

**Pass 1 — Map (breadth).** Find the entry points, modules and data flow around the subject. Derive
the search terms — symbols, types, error messages, config keys, CLI flags, env vars, file names —
and show me the list; every later pass searches for all of them.

**Pass 2 — Deep dive, in parallel.** Launch several `explorer` subagents in one message, one per
area, each returning numbered findings with evidence. The `explorer` has no shell: run the git and gh
reads below yourself (they're pre-approved) and give it their output.
- **Code:** every code path involved and every call site / reference of each symbol you might touch
  (`Grep` and `git grep` for all the search terms — not just the first hit).
- **Tests:** which tests and fixtures cover it, what they assert, and what is not covered.
- **Config and build:** flags, env, settings, build scripts, CI workflows that change behaviour.
- **History:** `git log -S<term>` / `git log -G<regex>` for the key symbols, `git blame` on the key
  lines, and `git show` of the commits that shaped them — why it is this way (the constraint or
  decision at the time, citing commits), never who: no author names, handles or emails in any output.
- **Prior work:** existing intents in `docs/adr/`; open **and closed** GitHub issues and PRs
  (`gh search issues`, `gh search prs`, `gh issue view`, `gh pr view`) — duplicates, earlier
  attempts, rejected approaches. If `gh` is not authenticated or can't reach GitHub, say so and
  continue without it.
- **Docs:** the repository's own docs first, then official documentation for the libraries and APIs
  involved — fetched only from the trusted domains (docs.github.com, git-scm.com, docs.rs, doc.rust-lang.org, docs.python.org, nodejs.org, developer.mozilla.org, go.dev, pkg.go.dev, kubernetes.io, developer.hashicorp.com, registry.terraform.io, docs.aws.amazon.com, cloud.google.com, learn.microsoft.com, datatracker.ietf.org, www.rfc-editor.org, code.claude.com, docs.anthropic.com), and only `https://` URLs:
  never `http://` (if a page is only served over plain HTTP, skip it and tell me). A WebFetch to any
  other site is blocked by this project's WebFetch guard; if one matters, tell me the host and I can
  trust it with `ocgen edit intent --trust-domain <host>`.

**Coverage checklist** — tick every item, or mark it N/A with a reason:
Entry points · All call sites · Data and state · Error and edge paths · Concurrency ·
Platforms (Windows/macOS/Linux) · Security and permissions · Performance · Tests · Config ·
Docs · History · Related issues and PRs · Existing intents.

**Pass 3 — Close the gaps, in parallel.** List what is still unexplored or unanswered — unticked
checklist items, Inferred findings, open leads — and investigate them all at once: launch `explorer` subagents in one message, one per gap (group small related gaps), each given the search
terms and the findings it builds on, and each returning numbered findings with evidence. Only a gap
that needs another's answer waits for the next round. Repeat until a pass finds nothing new (at most
3 extra passes). Only then ask me what the code can't answer — one question at a time, 2–4 options
with the recommended one first.

**Pass 4 — Challenge.** Give a fresh `explorer` subagent the findings' claims (not your conclusion)
and ask it to find evidence that would contradict them. Reconcile, and record what changed.

**Analysis report** — show it before any plan:
- **Current behaviour** (Verified findings)
- **Root cause / gap**
- **Impact** — files, callers, tests, users
- **Constraints**
- **Prior decisions and related work** — intents, issues, PRs
- **Risks**
- **Open questions**
- **Coverage checklist** (as above)
- **What would make this wrong** — the evidence that would overturn the conclusion
- **Overall `Confidence: NN%`**

## 3. Plan and get my approval
Present: the problem, the options considered with a recommendation, scope and out-of-scope, risks
each with a mitigation, and acceptance criteria. Every statement cites the findings it rests on
(F#); anything that doesn't is labelled **Assumption**. **Invite push-back** and iterate until I approve.
My approval makes the plan **approved for drafting**, not accepted: with no approvers, my confirmation in step 6 decides.
Do not write anything yet. Once I approve, ask whether to write an intent file (optional) — if not,
go to step 5.

## Writing standard (steps 4 and 5)
The team that wrote the code reads these drafts: keep them blameless and calm, and completely honest —
**soften the framing, never the facts**; no flaw is softened, hidden or dropped. Read the comment block
at the top of both templates and follow it: it holds the severity definitions and the banned words.
- **System, not people:** the subject is the code, design or behavior ("The gate matches command text
  only"), never a person or team.
- **Context before the flaw:** what the design was meant to achieve and what shaped it, then where it
  falls short.
- **Every flaw, in this order:** When [trigger], [behavior], which means [impact] (evidence: F#, `file:line`).
  Then the proposed fix.
- **Severity** (Critical, High, Medium, Low) and **Likelihood** come from the Verified findings, never
  from adjectives.
- **Balanced:** say what works correctly, without diluting any flaw. **Forward-looking:** each
  problem statement leads to the proposal.
- **Written for the approvers:** lead with a summary and a recommendation, and make the decision you
  need from them explicit — approve, choose an option, or redirect. Present; don't pressure.

## 4. Intent file
Draft it following the writing standard; write it only after the tone check (step 5).
1. **Pick the number** so it is never reused:
   - Local: the files in `docs/adr/` (any case) named `ADR-NNNN…` or `NNNN-…`.
   - Remote main branch: the remote's default branch
     (find it with `git ls-remote --symref origin HEAD`, which prints `ref: refs/heads/<branch>`;
     else use `main`) — `git fetch --quiet origin <branch>`, then `git ls-tree --name-only FETCH_HEAD docs/adr/` and the
     same name patterns.
   - The number is 1 + the highest found (`ADR-0001` when there are none), zero-padded to
     4 digits. If the fetch fails (offline, no remote), say so and ask me to confirm the number.
2. Name it `docs/adr/ADR-<number>-<kebab-case-slug>.md`. Fill the structure in
   `.claude/intent/intent-template.md` (if it is missing, use: title, Status, Severity, Approvers, Date, Issue,
   Context, Evidence, Decision, Options considered, Consequences, Risks, Sign-off) from the analysis
   and the approved plan, with `Status: Proposed`, `Approvers: No approvers configured`, and every
   approver pending in Sign-off.
3. Right before writing, re-check the local directory for the number; if it is taken, take the next.
   Show me the path.

## 5. Draft the GitHub issue, check the tone, show both
- **Title:** imperative and actionable (what to do, not the topic), at most 72 characters.
- **Description:** follow the writing standard and `.claude/intent/issue-template.md` (if it is missing, use:
  Summary, Current behaviour, Impact, Background, Proposal, Decision requested, Approvers / sign-off,
  Acceptance criteria, Out of scope, Links); at most 250 words — state the word count. Link
  the intent file when there is one.
- **Approvers / sign-off:** "No approvers configured". That is the only notice: **Never email**,
  message or otherwise contact anyone. Don't use `--assignee` for approvers — assignees do the work.
- **Tone check**, before showing anything: give a fresh `reviewer` subagent the drafts, the numbered
  findings with their severities, the writing standard and both templates' comment blocks, and ask
  it to report:
  (a) **Blame and alarm** — banned words, people as subjects, names from git history, judgmental framing.
  (b) **Softening and omission** (the more important) — every Verified finding of Medium or higher
  appears with its trigger, behavior, impact and evidence; no severity is lower than the evidence supports;
  no hedge contradicts a Verified finding.
  (c) **Approvers** — every configured approver is listed in both drafts with a pending sign-off.
  Fix everything it reports. Then write the intent file and show me its path, the title and the
  description in separate fenced blocks, and the command for me to run — save the description to a
  file and `gh issue create --title "<title>" --body-file <file>` — with one line on what the check changed.
- **Never create the issue yourself** — no `gh issue create`, no API call. Wait for me to file it.

## 6. Sign-off, link the issue, keep collaborating
When I give you the issue (`#123` or its URL), add `Issue: <url>` to the intent file.
**Never mark an approval** yourself and never infer one: record a sign-off in the intent file only
when I report it with evidence (e.g. `@alice approved: <link>`), one Sign-off line per report:
- **approved** — the date and the link;
- **changes requested** — the date, the link and their feedback, quoted; then go back to step 3 with it;
- **rejected** — the date and the link; set `Status: Rejected`.

Set `Status: Accepted` only when every approver has approved and I confirm (with no approvers,
once I confirm I'm proceeding without a sign-off). Then keep working on it with me — refine the
plan as the issue discussion evolves, or hand off to `/deliver` with the intent file as the brief.
