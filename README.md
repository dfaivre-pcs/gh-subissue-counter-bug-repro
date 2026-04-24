# GitHub sub-issue counter bug: `subIssuesSummary` frozen after first add

## TL;DR

On github.com, the denormalized sub-issue counter on `Issue` —
`subIssuesSummary.total` / `.completed` (GraphQL) and
`sub_issues_summary.total` / `.completed` (REST) — **freezes at `{total: 1,
completed: 0}` after the first sub-issue is linked**. Subsequent adds, removes,
and child-close events are computed correctly in the mutation response but
never persist.

The enumerated connection `Issue.subIssues.totalCount` is always correct. Only
the denormalized summary is stuck.

This repository contains four reproducible scenarios, each exercising a
different API write path. **All four reproduce the bug identically.** The write
path does not matter.

## Where the stale counter is visible

Both surfaces read the broken denormalized counter:

- **GitHub native issue list** — the parent issue displays a "0 / 1" progress
  badge next to the title even when it has 3 open children. See the issue list
  at
  <https://github.com/dfaivre-pcs/gh-subissue-counter-bug-repro/issues> — the
  four parent issues (#1, #5, #9, #13) each show "0/1".
- **Projects v2 backlog** — same parents added to
  <https://github.com/users/dfaivre-pcs/projects/7> also show "0/1".

Any tooling or dashboard reading `sub_issues_summary` (REST) or
`subIssuesSummary` (GraphQL) receives wrong data.

## Environment

- github.com (not GHES)
- Reproduced on a brand-new public repo (`dfaivre-pcs/gh-subissue-counter-bug-repro`)
- Also reproduced on a separate org repo with the same 4 write paths — bug is
  not per-repo or per-org
- Reproduced through both `gh` CLI (v2.91.0) and direct `curl` against the
  REST/GraphQL APIs

## The four scenarios

Each scenario creates one parent and three children. After every child is
linked, the **mutation/REST response returns the correct live-computed value**
(e.g. `total: 2`, then `total: 3`), but a subsequent re-query against REST
`GET /issues/{n}` or GraphQL `issue(number).subIssuesSummary` returns
`total: 1` — every time.

| Scenario | Write path | Parent | Children | Expected `subIssuesSummary.total` | Actual (persisted) |
|---|---|---|---|---|---|
| **A** | REST `POST /repos/{o}/{r}/issues/{n}/sub_issues` (link existing) | [#1](https://github.com/dfaivre-pcs/gh-subissue-counter-bug-repro/issues/1) | [#2](https://github.com/dfaivre-pcs/gh-subissue-counter-bug-repro/issues/2) [#3](https://github.com/dfaivre-pcs/gh-subissue-counter-bug-repro/issues/3) [#4](https://github.com/dfaivre-pcs/gh-subissue-counter-bug-repro/issues/4) | `3` | **`1`** |
| **B** | GraphQL `addSubIssue` (link existing) | [#5](https://github.com/dfaivre-pcs/gh-subissue-counter-bug-repro/issues/5) | [#6](https://github.com/dfaivre-pcs/gh-subissue-counter-bug-repro/issues/6) [#7](https://github.com/dfaivre-pcs/gh-subissue-counter-bug-repro/issues/7) [#8](https://github.com/dfaivre-pcs/gh-subissue-counter-bug-repro/issues/8) | `3` | **`1`** |
| **C** | GraphQL `createIssue(input: {parentIssueId, …})` (one-shot create + link) | [#9](https://github.com/dfaivre-pcs/gh-subissue-counter-bug-repro/issues/9) | [#10](https://github.com/dfaivre-pcs/gh-subissue-counter-bug-repro/issues/10) [#11](https://github.com/dfaivre-pcs/gh-subissue-counter-bug-repro/issues/11) [#12](https://github.com/dfaivre-pcs/gh-subissue-counter-bug-repro/issues/12) | `3` | **`1`** |
| **D** | [`yahsan2/gh-sub-issue`](https://github.com/yahsan2/gh-sub-issue) extension — `gh sub-issue create --parent N` (wraps Scenario C) | [#13](https://github.com/dfaivre-pcs/gh-subissue-counter-bug-repro/issues/13) | [#14](https://github.com/dfaivre-pcs/gh-subissue-counter-bug-repro/issues/14) [#15](https://github.com/dfaivre-pcs/gh-subissue-counter-bug-repro/issues/15) [#16](https://github.com/dfaivre-pcs/gh-subissue-counter-bug-repro/issues/16) | `3` | **`1`** |

`Issue.subIssues.totalCount` (enumerated) returns `3` for every parent above.

## Raw evidence per scenario

### Scenario A — REST `POST /sub_issues`

```text
--- LINK child 1 (POST body returns sub_issues_summary) ---
{"completed":0,"percent_completed":0,"total":1}
--- requery ---
{"completed":0,"percent_completed":0,"total":1}    ✓

--- LINK child 2 ---
{"completed":0,"percent_completed":0,"total":2}    ← response shows 2
--- requery ---
{"completed":0,"percent_completed":0,"total":1}    ✗ stuck at 1

--- LINK child 3 ---
{"completed":0,"percent_completed":0,"total":3}    ← response shows 3
--- requery ---
{"completed":0,"percent_completed":0,"total":1}    ✗ stuck at 1

--- GraphQL cross-check ---
subIssues.totalCount = 3                           ← enumerated, correct
subIssuesSummary = { total: 1, completed: 0 }      ← stored, stuck
```

### Scenario B — GraphQL `addSubIssue`

```text
--- LINK child 1 (mutation response) ---
subIssues.totalCount = 1, subIssuesSummary.total = 1
--- REST requery ---
{"total":1,"completed":0}                          ✓

--- LINK child 2 (mutation response) ---
subIssues.totalCount = 2, subIssuesSummary.total = 2   ← response shows 2
--- REST requery ---
{"total":1,"completed":0}                          ✗ stuck

--- LINK child 3 (mutation response) ---
subIssues.totalCount = 3, subIssuesSummary.total = 3   ← response shows 3
--- REST requery ---
{"total":1,"completed":0}                          ✗ stuck
```

### Scenario C — GraphQL `createIssue(parentIssueId: …)`

```text
--- CREATE child 1 ---
--- REST requery ---  {"total":1,"completed":0}    ✓
--- CREATE child 2 ---
--- REST requery ---  {"total":1,"completed":0}    ✗ expected 2
--- CREATE child 3 ---
--- REST requery ---  {"total":1,"completed":0}    ✗ expected 3

--- GraphQL cross-check ---
subIssues.totalCount = 3
subIssuesSummary = { total: 1, completed: 0 }
```

### Scenario D — `yahsan2/gh-sub-issue` extension

```text
$ gh sub-issue create --parent 13 --title "scenario D: child 1"    ← now #14
REST requery: {"total":1,"completed":0}    ✓
$ gh sub-issue create --parent 13 --title "scenario D: child 2"    ← now #15
REST requery: {"total":1,"completed":0}    ✗ expected 2
$ gh sub-issue create --parent 13 --title "scenario D: child 3"    ← now #16
REST requery: {"total":1,"completed":0}    ✗ expected 3

subIssues.totalCount = 3
subIssuesSummary = { total: 1, completed: 0 }
```

## Additional observations (from prior investigation)

Observed in a separate private repo. Not re-reproduced here to keep the public
artifacts pristine, but trivially reproducible by extending any scenario above.

1. **`subIssuesSummary.completed` is also frozen.** Closing children on a stuck
   parent leaves `.completed` at `0` while `totalCount` of closed children
   increases. The *entire summary object* is frozen, not just `.total`.
2. **Remove-to-zero does not unstick the counter.** After the 3 adds, calling
   `removeSubIssue` three times drops `subIssues.totalCount` back to `0`, but
   `subIssuesSummary.total` stays at `1`. The counter is permanently frozen
   after the first `0 → 1` transition.
3. **The mutation response always returns the correct live-computed value.**
   Confirmed for `addSubIssue`, `removeSubIssue`, and the REST
   `POST /sub_issues` response body. The value is derivable server-side — the
   write-back to the stored field is the step that never fires after the first
   transition.

## Reproduce from scratch

Prereqs:
- `gh` CLI authenticated with `repo` scope
- Optional for Scenario D: `gh extension install yahsan2/gh-sub-issue`

```bash
# 1. Create a throwaway repo
gh repo create my-subissue-repro --public --add-readme

REPO="<your-login>/my-subissue-repro"
```

### Scenario A — REST POST /sub_issues

```bash
PARENT=$(gh issue create --repo $REPO --title "A: parent" --body "" | tail -1 | grep -oE '[0-9]+$')
for i in 1 2 3; do
  eval "C$i=$(gh issue create --repo $REPO --title "A: child $i" --body "" | tail -1 | grep -oE '[0-9]+$')"
done

for num in $C1 $C2 $C3; do
  ID=$(gh api /repos/$REPO/issues/$num --jq '.id')
  # NOTE: use -F (raw field) — REST expects integer, not string.
  gh api -X POST /repos/$REPO/issues/$PARENT/sub_issues -F sub_issue_id=$ID --jq '.sub_issues_summary'
  gh api /repos/$REPO/issues/$PARENT --jq '.sub_issues_summary'   # ← persisted; watch this one
done
```

### Scenario B — GraphQL addSubIssue

```bash
PARENT=$(gh issue create --repo $REPO --title "B: parent" --body "" | tail -1 | grep -oE '[0-9]+$')
PARENT_NODE=$(gh api /repos/$REPO/issues/$PARENT --jq '.node_id')

for i in 1 2 3; do
  C_URL=$(gh issue create --repo $REPO --title "B: child $i" --body "" | tail -1)
  C_NUM=$(echo "$C_URL" | grep -oE '[0-9]+$')
  C_NODE=$(gh api /repos/$REPO/issues/$C_NUM --jq '.node_id')
  gh api graphql -f query="mutation { addSubIssue(input: {issueId: \"$PARENT_NODE\", subIssueId: \"$C_NODE\"}) { issue { subIssues(first:10) { totalCount } subIssuesSummary { total completed } } } }"
  gh api /repos/$REPO/issues/$PARENT --jq '.sub_issues_summary'
done
```

### Scenario C — GraphQL createIssue with parentIssueId

```bash
REPO_NODE=$(gh api /repos/$REPO --jq '.node_id')
PARENT=$(gh issue create --repo $REPO --title "C: parent" --body "" | tail -1 | grep -oE '[0-9]+$')
PARENT_NODE=$(gh api /repos/$REPO/issues/$PARENT --jq '.node_id')

for i in 1 2 3; do
  gh api graphql \
    -f query='mutation($repoId: ID!, $parentId: ID!, $title: String!) { createIssue(input: {repositoryId: $repoId, parentIssueId: $parentId, title: $title}) { issue { number } } }' \
    -f repoId=$REPO_NODE -f parentId=$PARENT_NODE -f title="C: child $i"
  gh api /repos/$REPO/issues/$PARENT --jq '.sub_issues_summary'
done
```

### Scenario D — yahsan2/gh-sub-issue

```bash
gh extension install yahsan2/gh-sub-issue   # one-time

PARENT=$(gh issue create --repo $REPO --title "D: parent" --body "" | tail -1 | grep -oE '[0-9]+$')
for i in 1 2 3; do
  gh sub-issue create --repo $REPO --parent $PARENT --title "D: child $i" --body ""
  gh api /repos/$REPO/issues/$PARENT --jq '.sub_issues_summary'
done
```

### Expected output (any scenario)

After the second and third add, `subIssuesSummary.total` should increment to
`2` and then `3`. Instead, it remains `1` permanently.

## Suggested fix area

The denormalized summary on `Issue` is written on the first `0 → 1`
transition but the subsequent update hook appears missing or guarded on a
stale "already-has-children" check. The mutation pipeline can compute the
correct value (proven by the mutation/REST response payloads), but the
persistence step is skipped after the first write.

Possible areas:
- A write-once-on-first-link path distinct from the general recompute path
- A stale cache / tombstone on the counter row that prevents subsequent
  updates
- A missing index/trigger on the child link table that only fires when the
  parent transitions from `has_children=false` to `has_children=true`

## Related / resources

- [Projects v2 board showing the 0/1 badges](https://github.com/users/dfaivre-pcs/projects/7)
- [All issues in this repo](https://github.com/dfaivre-pcs/gh-subissue-counter-bug-repro/issues)
  — each parent visibly shows "0/1" on the native issue list
- [yahsan2/gh-sub-issue](https://github.com/yahsan2/gh-sub-issue) — the
  extension used in Scenario D
