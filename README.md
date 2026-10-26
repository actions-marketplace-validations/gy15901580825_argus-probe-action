# Argus agent red-team scan

This action runs [Argus](https://github.com/gy15901580825/Argus) red-team probes against your AI agent from inside your CI job. It installs the `argus-probe` CLI, runs `argus-probe run --local` against the target you describe in a JSON file, and writes a report. With the default SARIF format it uploads the report to GitHub code scanning, so findings show up in the Security tab and on pull requests. The job fails when a probe fails at or above the `fail-on` severity. The SARIF upload happens before the job fails, so you still see the findings.

## Usage

```yaml
name: agent-red-team
on:
  pull_request:
  push:
    branches: [main]

permissions:
  contents: read
  security-events: write

jobs:
  argus:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: gy15901580825/argus-probe-action@v1
        env:
          ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
        with:
          target: .argus/target.json
          fail-on: high
```

A minimal `.argus/target.json` for an OpenAI-compatible endpoint:

```json
{
  "kind": "openai_compat",
  "endpoint_url": "https://my-agent.example.com/v1/chat/completions",
  "model": "my-agent",
  "api_key": "${MY_AGENT_KEY}"
}
```

`${VAR}` references are read from the environment at run time, so put the secret in the step's `env:` and keep it out of the file. The [target spec cookbook](https://github.com/gy15901580825/Argus/blob/main/docs/onboarding/target-spec-cookbook.md) covers the other target kinds.

## Inputs

| Input | Default | Description |
|---|---|---|
| `target` | (required) | Path to the target spec JSON file. |
| `probes` | `owasp_01_prompt_injection_basic,owasp_07_system_prompt_leakage` | Comma-separated probe ids, or `all`. |
| `format` | `sarif` | Report format: `sarif`, `html` or `junit`. |
| `output` | `argus.sarif` | Path of the report file. |
| `fail-on` | `critical` | Fail the job when a probe fails with this severity or higher (`info`, `low`, `medium`, `high`, `critical`). Set it to `''` to turn the gate off. |
| `per-run-cap` | `0.50` | Maximum LLM spend in USD for one scan. The CLI refuses to start a scan whose estimate is above the cap. |
| `upload-sarif` | `true` | Upload the report to code scanning. Ignored unless `format` is `sarif`. |
| `argus-version` | `0.1.1` | Version of `argus-probe` to install from PyPI. |
| `install-spec` | `''` | Space-separated pip specs to install instead of `argus-probe[local]==<argus-version>`, for example a git URL or a wheel path. |
| `python-version` | `3.11` | Python version passed to `actions/setup-python`. |

## Outputs

| Output | Description |
|---|---|
| `exit-code` | Exit code of `argus-probe run`. 0 means no gated failure, 1 is an error, 2 means local mode is unavailable, 3 means the `fail-on` gate tripped. |
| `report` | Path of the report file. |

## Notes

- The judge needs an API key. Set `ANTHROPIC_API_KEY`, or set `REDTEAM_MODEL_PROVIDER=azure` plus the Azure OpenAI variables. Without a key the scan exits with code 1.
- Pull requests from forks and from Dependabot get no repository secrets. The judge key is empty there, so the job fails. Skip the job for those events, for example with `if: github.event_name != 'pull_request' || (github.event.pull_request.head.repo.full_name == github.repository && github.actor != 'dependabot[bot]')`.
- `browser_use` targets do not work in local mode. Use an HTTP or gRPC target kind.
- Every scan costs money on your judge key. The default cap is $0.50 per scan. Raise `per-run-cap` when you run `probes: all`.
- The action fails the job on any nonzero exit code, not only on 3. The error message says whether the gate tripped or the scan itself failed.
- Uploading SARIF needs `security-events: write`. On private repositories code scanning also needs GitHub Advanced Security.

## License

Apache-2.0. See [LICENSE](LICENSE).
