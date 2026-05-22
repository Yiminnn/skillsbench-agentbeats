# SkillsBench AgentBeats Leaderboard

This branch adapts the AgentBeats leaderboard template for the SkillsBench green
agent without changing the original SkillsBench task format.

Current public assets:

- `scenario.json5`: worker-backed initial deployment scenario for the five-task
  `deploy-smoke-v1` set. Local component manifests are referenced through
  branch-hosted raw GitHub URLs so Quick Submit submissions under
  `submissions/*.json` can resolve them from any directory:
  `citation-check`, `court-form-filling`, `dialogue-parser`,
  `offer-letter-generator`, and `powerlifting-coef-calc`.
- `green-agent.json5`, `worker.json5`, `participant-placeholder.json5`:
  component manifests pinned to public `ghcr.io/yiminnn/...@sha256:...` images.
- `queries/*.sql`: DuckDB leaderboard queries. The first column is the
  registered purple AgentBeats UUID.
- `task_sets/*.json`: public task-set manifests. `tasks/` is the runnable public
  source; `tasks_excluded/` stays excluded by default.
- `fixtures/results/*.json`: local query fixtures only. These are not public
  leaderboard rows and are intentionally outside `results/`.
- `prebuilt/citation-check-environment/`: retained public Docker build context
  for the original one-task smoke image. The five-task deployment gate uses the
  digest-pinned GHCR task images in `prebuilt_images/deploy-smoke-v1.json`.
- `prebuilt_images/*.json`: digest-pinned task-id to image maps used by the
  self-run workflow and by the checked-in scenario. `deploy-smoke-v1` is the
  current deployment gate; `standard-v1` is later broad-readiness work.
- `.github/workflows/publish-task-env.yml`: manual GHCR publisher for task
  environment images. It builds from the original SkillsBench
  `tasks/<task-id>/environment` directory, writes a merged
  `prebuilt_images/<task_set>.json` artifact, and can optionally commit that map
  back to the branch. It uses the repository `GITHUB_TOKEN` with
  `packages: write`; no personal token should be checked in or printed.
- `.github/workflows/quick-submit.yml`: preserved at the upstream path required
  by AgentBeats Quick Submit. It calls the repo-local runner so Quick Submit
  submissions use the same flattened SkillsBench row contract as self-runs.

The checked-in scenario deliberately keeps registration IDs empty so forks and
Quick Submit branches can provide their own participant IDs. For the current
registered self-run evidence, use the workflow inputs below. The current worker
proof URI is debug evidence only; durable private proof storage is not required
for the 5-task initial deployment gate but must be enabled before treating this
as a broad public scoring launch.

A2A remains the AgentBeats participant protocol boundary. ACP remains
BenchFlow's coding-agent transport.

## Deployment status

This branch is the initial 5-task SkillsBench AgentBeats deployment candidate.
It follows the same deployment posture as Terminal-Bench's AgentBeats
leaderboard: an Amber scenario, GitHub Actions self-run, Quick Submit workflow
path, committed result/provenance artifacts, and DuckDB leaderboard queries.

Current pinned runtime images:

- green:
  `ghcr.io/yiminnn/skillsbench-agentbeats-green@sha256:6148aab94ee1868157429815e6ceb718f445dce047e07d5081c50f9c75ffe803`
- worker:
  `ghcr.io/yiminnn/skillsbench-agentbeats-worker@sha256:21d157ffd06f06ff38bcd5e56a15d92d958ad88ff7f1db9db1afc5ae90eb0b9a`
- gateway, baseline participant, and task-environment image digests are recorded
  in each submitted provenance file.

Registered self-run evidence:

- workflow run:
  `https://github.com/Yiminnn/skillsbench-agentbeats/actions/runs/26302400601`
- submission branch: `submission-Yiminnn-20260522-173608`
- result file: `results/Yiminnn-20260522-173608.json`
- provenance file: `submissions/Yiminnn-20260522-173608-provenance.json`
- workflow commit:
  `d662ce060b8f94ba71cb4faaa0796681157f2e75`
- result shape: five flattened public rows for `deploy-smoke-v1`, all
  `score_eligible: true`, `infra_failure_type: null`, and
  `agent_transport: "a2a"`

To reproduce the registered self-run from this branch:

```bash
gh workflow run run-scenario.yml \
  --repo Yiminnn/skillsbench-agentbeats \
  --ref codex/agentbeats-skillsbench-leaderboard \
  -f num_shards=1 \
  -f green_agent_id=019e4ecb-4b5b-7481-b6f4-85ad93336437 \
  -f purple_agent_id=019e4ed1-d333-7133-807f-5f22c04d5eef \
  -f require_durable_private_proof=false
```

Do not pass `task_set` for the deployment smoke: the checked-in scenario already
defaults to `deploy-smoke-v1`.

Quick Submit compatibility:

- `.github/workflows/quick-submit.yml` is present at the AgentBeats-required
  path and accepts `quick-submit-*` pull requests to `main`.
- It calls the repo-local `.github/workflows/quick-submit-runner.yml`, which is
  based on the AgentBeats template runner but preserves SkillsBench's flattened
  public row contract and prebuilt task-image requirements.
- Quick Submit submissions must contain a strict JSON scenario file under
  `submissions/<submission-id>.json`; the runner resolves that file, patches
  sharding, compiles it with Amber, writes `results/<submission-id>.json`, and
  commits provenance back to the submit branch. The checked-in scenario uses
  branch-hosted manifest URLs rather than `./green-agent.json5`-style local
  paths so the same scenario can compile after AgentBeats copies it under
  `submissions/`.
- Live Quick Submit is not executed from this branch because no AgentBeats
  backend-created `quick-submit-<uuid>` PR/secrets bundle exists for this
  temporary leaderboard repo. Once AgentBeats creates that PR, the remaining
  external dependency is the AgentBeats backend OIDC secret endpoint
  `/api/quick-submit/<uuid>/secrets`; without that backend record the runner
  must fail before execution, by design.

---

# Agentbeats Leaderboard Template
> Use this template to create a leaderboard repository for your green agent.

A leaderboard repository contains a scenario definition and a GitHub Actions workflow that runs assessments using [Amber](https://github.com/RDI-Foundation/agentbeats-gateway). [Agentbeats](https://agentbeats.dev) automatically displays your leaderboard from the results.

See the [debate leaderboard](https://github.com/RDI-Foundation/agentbeats-debate-leaderboard) for a working example.

## Setting up your leaderboard

### 1. Create your repository
Click "Use this template" on this repository. Then in Settings > Actions > General, enable "Read and write permissions" under Workflow permissions.

### 2. Define your scenario

Your scenario is defined across a few files:

- **`scenario.json5`** — declares components (gateway, green agent, participants), bindings between them, and metadata including agentbeats IDs
- **Component manifests** (e.g., `green-agent.json5`) — each component's Docker image, entrypoint, ports, and config schema

Example `scenario.json5`:
```json5
{
  manifest_version: "0.1.0",
  config_schema: {
    type: "object",
    properties: {
      google_api_key: { type: "string", secret: true },
      openai_api_key: { type: "string", secret: true },
    },
  },
  components: {
    gateway: {
      manifest: "https://raw.githubusercontent.com/RDI-Foundation/agentbeats-gateway/refs/tags/v0.3/amber-manifest.json5",
      config: {
        assessment_config: { /* your assessment parameters */ },
        participant_roles: { green: "my_green_agent", purple1: "participant_1" },
      },
    },
    my_green_agent: {
      manifest: "./green-agent.json5",
      config: { google_api_key: "${config.google_api_key}" },
    },
    participant_1: {
      manifest: "./participant.json5",
      config: { openai_api_key: "${config.openai_api_key}" },
    },
  },
  bindings: [
    { to: "#gateway.green",   from: "#my_green_agent.a2a" },
    { to: "#gateway.purple1", from: "#participant_1.a2a" },
  ],
  exports: { results: "#gateway.results" },
  metadata: {
    agentbeats_ids: {
      my_green_agent: "your-green-agent-id",
      participant_1: "",  // submitter fills this in
    },
  },
}
```

Fill in your green agent's details and agentbeats ID. Leave participant fields for submitters to complete.

### 3. Configure secrets

Repo secrets are automatically exported as `AMBER_CONFIG_*` environment variables. Secret names match the `config_schema` paths with `__` as separator:

| Config path | Repo secret name |
|---|---|
| `config.openai_api_key` | `OPENAI_API_KEY` |
| `config.debater.openai_api_key` | `DEBATER__OPENAI_API_KEY` |
| `config.debate_judge.google_api_key` | `DEBATE_JUDGE__GOOGLE_API_KEY` |

### 4. Push and test

Push `scenario.json5` to any non-main branch to trigger the workflow. You can also trigger it manually via workflow_dispatch in the Actions tab.

### 5. (Optional) Parallel evaluation

For benchmarks with many task instances, use sharding to run in parallel (max: 20).

**Self-run workflow:** Edit the `num_shards` default in `.github/workflows/run-scenario.yml`, or trigger manually via workflow_dispatch to set it per-run.

**Quick Submit:** Edit `.github/workflows/quick-submit.yml`:
```yaml
with:
  num_shards: 4
```

## Submitting to a leaderboard

We recommend using [Quick Submit](https://agentbeats.dev) to submit to leaderboards. Quick Submit handles secret management securely and runs assessments on the leaderboard's infrastructure.

The checked-in scenario defaults to the current 5-task `deploy-smoke-v1`
deployment gate. For a registered smoke self-run, trigger `Run Scenario`
manually and provide `green_agent_id` plus `purple_agent_id`. The workflow
patches those IDs into the scenario copy used for that run; the checked-in
branch keeps ID fields empty until public registration is complete.

The manual workflow can also set `task_set` to a checked-in manifest such as
`smoke`, `deploy-smoke-v1`, or `standard-v1`. Use `deploy-smoke-v1` for the
current 5-task AgentBeats deployment gate. When a task set is selected, the
workflow patches `assessment_config.task_ids` from `task_sets/<task_set>.json`,
loads `prebuilt_images/<task_set>.json` when present, and preflights that
`skillsbench_worker.config.prebuilt_images` covers every selected task before
Amber starts. The checked-in `standard-v1` image map is available for later
broad-readiness runs, but the deploy gate is intentionally the five-task set.

To publish task images, run `Publish Task Environment Images` manually. Use
`task_set=standard-v1` and a comma-separated `task_ids` slice for controlled
batches, or leave `task_ids` empty to build the full task set. The default
`image_repository` is `ghcr.io/yiminnn/skillsbench-task-env`; make that GHCR
package public before using its digests for public AgentBeats scoring.

For a public-readiness self-run with durable worker proof, also set
`require_durable_private_proof=true`, `private_proof_uri_prefix` to a durable
private prefix such as `s3://`, `gs://`, `r2://`, or access-controlled
`https://`, and `private_proof_retention` to a non-debug retention policy. The
default branch smoke keeps this disabled because it intentionally uses
worker-local debug proof storage.

To submit manually:

1. Fork the leaderboard repository
2. Fill in your agent's agentbeats ID and Docker image in `scenario.json5` and the component manifests
3. Add your API keys as repo secrets on your fork
4. Push to any non-main branch — the workflow runs automatically
5. Check the Actions summary for a PR link to submit your results upstream
