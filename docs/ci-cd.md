# CI/CD Integration

Shannon runs headlessly and non-interactively, so it fits ephemeral CI environments. This guide covers credentials, artifact paths, SARIF upload, and the runtime and cost characteristics that shape where a Shannon job belongs in a pipeline.

Everything here is part of Shannon Open Source. None of it is gated behind a commercial edition.

> [!WARNING]
> Shannon actively executes exploits. Point CI jobs at ephemeral preview environments, staging, or disposable test deployments that you own. Do not run Shannon against production.

## Headless Requirements

A CI run needs three things:

- **Docker**, for the worker container. GitHub-hosted runners already provide it.
- **Node.js 18+**, for the `npx` workflow.
- **Provider credentials as environment variables**, so no interactive setup step runs.

`npx @keygraph/shannon setup` is the interactive credential wizard and is not used in CI. Export the variables instead:

```bash
export ANTHROPIC_API_KEY=...
```

`SHANNON_AI_MODEL` selects the provider and model as `<provider>:<model-id>`. Left unset, Shannon uses its default Claude model, so a job that exports only `ANTHROPIC_API_KEY` runs without further configuration. See [AI providers](ai-providers.md) for other providers, gateways, and custom base URLs.

> [!NOTE]
> Anthropic and OpenAI apply real-time safeguards to cyber-security workloads, which can interrupt a scan mid-run. Complete their guidance for legitimate security testers before wiring Shannon into a pipeline. See [cyber safeguards](ai-providers.md#cyber-safeguards-do-this-before-your-first-scan).

## How a Scan Runs in CI

`shannon start` launches the scan in a detached worker container and returns as soon as the run registers. It does not block until the pentest finishes.

`shannon logs <workspace>` streams the run's log and returns when the scan reports `COMPLETED` or `FAILED`. Pair the two commands to make a CI step wait for results:

```bash
npx @keygraph/shannon start -u "$TARGET_URL" -r . -c shannon.yaml -w ci-run -o ./shannon-results
npx @keygraph/shannon logs ci-run
```

Pass an explicit workspace name with `-w` so the `logs` command has a deterministic name to attach to. Without it, Shannon generates a name from the hostname and a timestamp.

## Output Artifacts

`-o <path>` copies the run's deliverables out of the workspace and into a directory the rest of your pipeline can read:

| File | Contents |
| --- | --- |
| `report.sarif` | SARIF 2.1.0 log. Written only when `report.sarif` is enabled and the run is exploitative. |
| `report.json` | Structured findings emitted by the report agent. The Markdown report is rendered from it. |
| `Security-Assessment-Report.pdf` | The human-facing report. |
| `comprehensive_security_assessment_report.md` | The assembled Markdown report. |

`report.sarif` and `Security-Assessment-Report.pdf` are also surfaced at the workspace root, so a step that reads from the workspace directly can rely on a stable path.

## SARIF Output

SARIF 2.1.0 is the OASIS standard interchange format for static analysis results. Any tool that reads SARIF ingests `report.sarif` unchanged, so the GitHub Actions example below is one consumer among many, not a requirement.

SARIF is opt-in. Enable it in a configuration file:

```yaml
# shannon.yaml
report:
  sarif: "true"
```

SARIF requires an exploitative run, which is the default. Shannon does not write a SARIF log for analysis-only runs (`exploit: "false"`).

Each finding becomes one SARIF result, filed under a rule per vulnerability class (`shannon/injection`, `shannon/xss`, `shannon/auth`, `shannon/authz`, `shannon/ssrf`) and tagged with its OWASP Top Ten 2025 category. Severity maps onto SARIF's three levels: `critical` and `high` become `error`, `medium` becomes `warning`, everything else becomes `note`.

See [Configuration](configuration.md#sarif-output) for the full mapping.

## GitHub Actions

```yaml
name: Shannon Pentest
on: [pull_request]

jobs:
  pentest:
    runs-on: ubuntu-latest
    permissions:
      contents: read
      security-events: write
    steps:
      - uses: actions/checkout@v4

      - name: Run Shannon
        run: |
          npx @keygraph/shannon start \
            -u ${{ vars.TARGET_URL }} \
            -r . \
            -c shannon.yaml \
            -w ci-${{ github.run_id }} \
            -o ./shannon-results

          npx @keygraph/shannon logs ci-${{ github.run_id }}
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}

      - name: Upload results
        uses: github/codeql-action/upload-sarif@v3
        with:
          sarif_file: ./shannon-results/report.sarif
```

`security-events: write` is required for `upload-sarif` to publish into GitHub code scanning.

## Gating Merges

Shannon does not currently fail the job based on what it finds. `logs` returns successfully whether the scan completed or failed, so a passing step means the pipeline ran, not that the target is clean.

Two options for turning findings into a gate:

- **GitHub code scanning**: once the SARIF is uploaded, use code scanning's own pull request checks and severity rules to block a merge.
- **Your own check**: read `report.json` in a follow-up step and exit non-zero on the findings you care about.

Filter before you gate. `report.min_severity` drops findings below a severity threshold at report time, so both the SARIF and the JSON carry only what you want to act on:

```yaml
# shannon.yaml
report:
  min_severity: high
  sarif: "true"
```

Because Shannon reports only vulnerabilities it has produced a working proof-of-concept for, a gate built on these results fires on proven exploitation rather than speculative alerts.

## Authenticated Targets

Most useful CI targets sit behind a login. Describe the login flow, test credentials, and rules of engagement in the same configuration file you pass with `-c`, and supply secrets through environment variables rather than committing them. See [Configuration](configuration.md).

## Runtime and Cost

A full run can take roughly 1 to 1.5 hours and incurs LLM API costs that scale with model pricing and application complexity. That shapes where the job belongs:

- Scheduled runs against a staging environment, or a manual `workflow_dispatch`, fit the runtime better than a check on every pull request.
- If you do run per pull request, scope it: limit `vuln_classes`, or trigger only on changes to security-sensitive paths.
- Give the job a generous `timeout-minutes`. GitHub-hosted runners default to a six-hour job limit, but the step will inherit whatever you set.
- Use `-w` with a stable workspace name to resume an interrupted run rather than restarting it from the first agent. See [Workspaces and resuming](workspaces.md).

## Other CI Systems

Nothing in the workflow is GitHub-specific. Any runner with Docker and Node.js 18+ can run the same two commands, export the same credentials, and collect the same artifacts from the `-o` directory. SARIF consumers other than GitHub code scanning read `report.sarif` unchanged.
