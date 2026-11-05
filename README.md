# auto-merge-actions

Central, public GitHub Actions runner for enabling repository auto-merge settings.
It does not merge or queue any pull request.

## Dispatch

The `enable-auto-merge.yml` workflow runs only by manual/API dispatch. From the
Actions tab, select **Enable repository auto-merge**, then **Run workflow**.
The default credential is `GH_CLASSIC`; `COPILOT_GITHUB_TOKEN` is also supported.
The default targets are `aozq/expo-native-core aozq/Voice-Flow-Engine`.

An authorized session can dispatch and monitor it using its existing GitHub CLI
authentication (never supply organization secret values in inputs):

```sh
gh workflow run enable-auto-merge.yml \
  --repo aozq-team/auto-merge-actions --ref main \
  -f credential=GH_CLASSIC \
  -f repositories='aozq/expo-native-core aozq/Voice-Flow-Engine'

gh run list --repo aozq-team/auto-merge-actions \
  --workflow enable-auto-merge.yml --event workflow_dispatch --limit 5
gh run watch RUN_ID --repo aozq-team/auto-merge-actions --exit-status
gh run view RUN_ID --repo aozq-team/auto-merge-actions --log
```

For REST API dispatch:

```sh
gh api --method POST \
  repos/aozq-team/auto-merge-actions/actions/workflows/enable-auto-merge.yml/dispatches \
  --input - <<'JSON'
{
  "ref": "main",
  "inputs": {
    "credential": "GH_CLASSIC",
    "repositories": "aozq/expo-native-core aozq/Voice-Flow-Engine"
  }
}
JSON
```

Other workflows can use the same CLI/API dispatch with their own authorized
credential supplied through `GH_TOKEN` from an Actions secret. Their default
`GITHUB_TOKEN` is scoped to their own repository and normally cannot dispatch
this central repository's workflow. Dispatch requires write access and suitable
Actions permissions here (fine-grained tokens/GitHub Apps: Actions write;
classic PATs: the appropriate repository scope). Reading runs requires Actions
read access. Publishing workflow changes also requires workflow write permission.
No token is passed in the dispatch payload.

## Secret access and behavior

Organization Actions secrets `GH_CLASSIC` and `COPILOT_GITHUB_TOKEN` must be
accessible to this **public** repository. In organization **Settings > Secrets
and variables > Actions**, set each needed secret's repository access to
**All repositories**, or **Selected repositories** including
`aozq-team/auto-merge-actions`. **Private repositories** excludes this repository.
If the organization plan does not support organization secrets for this
repository, an organization administrator must resolve that restriction.
Do not copy secret values into chat, workflow inputs, source files, or logs.

Inputs are validated and supplied to the script through environment variables.
Targets must be space-separated `owner/repository` names. There is no target
allow-list: the selected credential's permissions determine access.

For every target, the workflow attempts
`gh repo edit <repo> --enable-auto-merge`, then independently verifies with
`gh api repos/<repo> --jq .allow_auto_merge`, even if the edit fails.
It continues to all targets and fails if any edit or verification command fails,
or any verified value is not `true`. Exact GitHub command output, error messages,
exit codes, and per-target results appear in logs and the run summary.
An unavailable selected secret fails explicitly before any edits.
Failed targets also include read-only REST/GraphQL settings, credential
permissions, and the authenticated account's reported plan in logs and summaries.
Diagnostic failures are reported with their exact errors and exit codes; they
never turn a failed target into a success.

A successful edit or reported `admin=true` is not proof that the setting changed.
Treat the subsequent `allow_auto_merge` value as the actual result; if it remains
false, inspect the recorded GitHub errors and repository/organization policies
instead of assuming another token scope change will fix it.

GitHub Free supports auto-merge only for public repositories. Private repositories
require GitHub Pro (personal ownership), Team, or Enterprise; hosting this runner
in a public repository does not change the targets' eligibility. See
[GitHub's auto-merge availability](https://docs.github.com/en/pull-requests/how-tos/merge-and-close-pull-requests/automatically-merging-a-pull-request).
Compare REST `allow_auto_merge` with GraphQL `autoMergeAllowed` when diagnosing
an accepted edit that does not persist. An absent REST field is not success.
