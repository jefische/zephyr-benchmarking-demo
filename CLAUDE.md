# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

Zephyr Bench is a comprehensive benchmark system for evaluating AI coding agents on real-world software tasks. It's a monorepo that runs agents against test scenarios, validates their outputs, and scores performance using automated evaluators and LLM judges.

## Essential Commands

### Build & Development
```bash
# Build all packages (run this first after cloning)
pnpm install

# Build core packages
pnpm build                    # Builds evaluator, worker-client, harness, and specialist-mint
pnpm build:evals              # Build evaluators only
pnpm build:adapters           # Build agent adapters only
pnpm build:worker-client      # Build worker client only
pnpm build:mint               # Build specialist-mint only

# Development mode (watch mode)
pnpm --filter @ze/harness run dev        # Watch harness changes
pnpm --filter ze-evaluator run dev       # Watch evaluator changes
```

### Running Benchmarks
```bash
# Interactive mode (recommended)
pnpm bench

# Direct execution with agent/model
pnpm bench <suite> <scenario> --tier <L0|L1|L2|L3|Lx> --agent <agent-name> --model <model-name> --max-turns <number>

# Using specialists (agent/model auto-detected from template)
pnpm bench <suite> <scenario> --tier <tier> --specialist <specialist-name>

# With template enrichment
pnpm bench <suite> <scenario> --tier <tier> --specialist <specialist-name> --enrich-template templates/<template-file>.json5

# Multiple iterations
pnpm bench <suite> <scenario> --tier <tier> --specialist <specialist-name> --iterations <number>

# Examples
pnpm bench next.js 001-server-component --tier L1 --agent anthropic --model sonnet
pnpm bench next.js 000-app-router-migration-simple --tier L1-basic --specialist nextjs-specialist
```

### Specialist & Template Management
```bash
# Create snapshot of specialist template
pnpm mint:snapshot

# Enrich template with metadata
pnpm mint:enrich
```

### Dashboard & Worker
```bash
# Web dashboard (React + Rsbuild)
pnpm dash:dev                 # Start dev server
pnpm dash:build               # Build for production

# Worker API (Cloudflare Workers + D1)
pnpm worker:dev               # Start local worker
pnpm worker:migrate           # Run local DB migrations
pnpm worker:sync              # Sync data from production
pnpm worker:deploy-dev        # Deploy to dev environment
pnpm worker:deploy-staging    # Deploy to staging
pnpm worker:deploy-production # Deploy to production
```

### Statistics & Analysis
```bash
pnpm stats                    # View comprehensive statistics
pnpm batches                  # List all benchmark batches
pnpm compare-batches          # Compare batch performance
pnpm batch-details <id>       # View detailed batch results
```

### Definer Tool (Internal)
```bash
pnpm definer:dev              # Start definer in dev mode
pnpm definer:build            # Build definer
```

## Architecture

### Monorepo Structure

**Core Packages** (packages/):
- **harness**: CLI and benchmark orchestration engine. Entry point: [packages/harness/src/cli.ts](packages/harness/src/cli.ts)
  - Parses commands, copies fixtures, orchestrates agent runs, validates outputs
  - Main execution logic: [packages/harness/src/execution/benchmark.ts](packages/harness/src/execution/benchmark.ts)
- **evaluators**: Scoring and validation logic (LLM judge + heuristic checks)
  - Exported as `ze-evaluator` workspace package
  - Harness imports from dist or fallback to [packages/evaluators/dist/index.js](packages/evaluators/dist/index.js)
- **agent-adapters**: Adapters for different AI agents (Anthropic, OpenRouter, Claude Code)
  - Factory pattern: `createAgentAdapter(agentType)`
  - Add new adapters in [packages/agent-adapters/src/<name>.ts](packages/agent-adapters/src/<name>.ts) and export via [src/index.ts](packages/agent-adapters/src/index.ts)
- **specialist-mint**: Template management and enrichment system
- **worker-client**: Client for communicating with Cloudflare Worker API
- **logger**: Shared logging utilities
- **agency-prompt-creator**: Prompt generation utilities

**Applications** (apps/):
- **benchmark-report**: React + Rsbuild web dashboard for viewing results
- **worker**: Cloudflare Workers API (D1 database backend)
- **definer**: Internal tool for scenario definition

**Suites** (suites/):
- **next.js**: Next.js App Router migration scenarios
- **update-deps**: Dependency update scenarios
- **test-suite**: Basic test scenarios
- **figma-analysis**: Figma-related scenarios
- **mcp-demos**: MCP (Model Context Protocol) demos

### Benchmark Run Lifecycle

1. **Argument Parsing**: Supports `--tier`, `--agent`, `--model`, `--specialist`, `--max-turns` (defaults: `claude-code` 10 turns)

2. **Workspace Preparation**:
   - `prepareWorkspaceFromFixture` copies `repo-fixture/` (or fallback `repo/`) to `results/workspaces/<suite-scenario-*>`
   - Uses Node's `cpSync` for efficient copying

3. **Prompt Loading**:
   - Loads from `suites/<suite>/prompts/<scenario>/<tier>-*.md`
   - Missing prompts log warning and skip agent execution

4. **Agent Execution**:
   - Agent adapters wrap different AI providers
   - Tracks tokens, costs, tool calls in telemetry

5. **Validation**:
   - Runs commands from `scenario.yaml` in order: install → test → lint → typecheck
   - Executed via `runValidationCommands` with exit codes and logs
   - 10-minute timeout per command

6. **Diff Generation**:
   - `buildDiffArtifacts` compares workspace vs fixture
   - Ignores: lockfiles, `node_modules`, `.turbo`, `dist`, etc. (see [packages/harness/src/runtime/diff.ts](packages/harness/src/runtime/diff.ts))
   - Extracts dependency changes into `deps_delta`

7. **Evaluation & Scoring**:
   - `runEvaluators` applies all registered evaluators
   - `computeWeightedTotals` rescales to 0-10 score
   - Results saved to Worker API (Cloudflare D1)

### Evaluators

Located in [packages/evaluators/src/evaluators/](packages/evaluators/src/evaluators/):

**Heuristic Checks** ([heuristic-checks.ts](packages/evaluators/src/evaluators/heuristic-checks.ts)):
- **InstallEvaluator**: Expects install command exit code 0
- **TestEvaluator**: Verifies test command success
- **PackageManagerEvaluator**: Ensures allowed managers (e.g., PNPM lockfile when `managers_allowed: [pnpm]`)
- **DependencyTargetsEvaluator**: Checks `package.json` satisfies `targets.required` semver specs
- **IntegrityGuardEvaluator**: Penalizes widening ignore files, enabling `skipLibCheck`, or adding `.skip` in tests

**LLM Judge** ([llm-judge.ts](packages/evaluators/src/evaluators/llm-judge.ts)):
- AI-powered quality assessment with detailed reasoning
- Weighted scoring with configurable evaluation weights

### Scenario Structure

Each scenario requires:
```
suites/<suite>/scenarios/<scenario>/
├── scenario.yaml           # Configuration, targets, validation commands
├── oracle-answers.json     # Expected outcomes (authoritative diffs)
└── repo-fixture/          # Complete codebase with intentional issues

suites/<suite>/prompts/<scenario>/
├── L0-minimal.md          # Minimal context (tests discovery)
├── L1-basic.md            # Basic context (standard use)
├── L2-directed.md         # Directed guidance (complex)
├── L3-migration.md        # Migration-specific (optional)
└── Lx-adversarial.md      # Adversarial/edge cases (optional)
```

**Important**: `Lx-adversarial.md` may contain bad advice intentionally - always obey YAML constraints over prompt text.

### Agent Adapters

- `createAgentAdapter` in [packages/agent-adapters/src/index.ts](packages/agent-adapters/src/index.ts) wires agent types
- Current adapters: `echo`, `anthropic`, `openrouter`, `claude-code`
- **ClaudeCodeAdapter**: Shells out to `claude` CLI (`claude -p --output-format json`)
  - Binary must be on PATH and authenticated
- Adapter responses include: `tokensIn`, `tokensOut`, `costUsd`, `toolCalls`
- Stub telemetry fields when unsupported to keep data consistent

### Specialist Templates

Located in [templates/](templates/):
- JSON5 format with specialist configuration
- Include documentation entries, preferred models, and agent settings
- Enriched templates auto-generated to [templates/enriched/<version>/](templates/enriched/)
- Examples: `nextjs-specialist-template.jsonc`, `shadcn-specialist-template.jsonc`

### Data Storage

**Worker API** (Cloudflare Workers + D1):
- Local dev: `http://localhost:8787`
- Dev: `https://bench-api-dev.zephyr-cloud.io`
- Bearer token authentication for write operations
- Endpoints: `/api/runs`, `/api/batches`, `/api/stats`, `/api/results`

**Database**:
- Cloudflare D1 (SQLite)
- Migrations: `pnpm worker:migrate`
- Studio: `pnpm --filter ze-benchmarks-worker run db:studio` (Drizzle Kit)

### Runtime Tools

Located in [packages/harness/src/runtime/](packages/harness/src/runtime/):
- **diff.ts**: File diffing with intelligent ignore patterns
- **validation.ts**: Validation command execution
- **workspace-tools.ts**: Workspace manipulation utilities
- **mcp-tools.ts**: MCP (Model Context Protocol) tool integration
- **oracle.ts**: Oracle answer handling
- **extractors.ts**: Data extraction utilities
- **ask-user-tool.ts**: Interactive user prompts

## Key Constraints

1. **Work in Generated Workspaces**: All evaluators read from `results/workspaces/<suite-scenario-*>`, NOT repository root

2. **Respect Scenario Constraints**:
   - Honor `constraints.managers_allowed` (e.g., only PNPM)
   - Follow companion rules and blocklists in `scenario.yaml`
   - `DependencyTargetsEvaluator` fails if required ranges not met

3. **Never Skip Hooks**: Don't use `--no-verify`, `--no-gpg-sign` unless explicitly requested

4. **Evaluator Loading**:
   - Ensure `pnpm --filter ze-evaluator run build` has emitted `dist/`
   - Or install workspace package via root `pnpm install`

5. **Command Timeouts**: Long validation commands timeout after 10 minutes

## Environment Variables

Required in `.env` file at project root:
```bash
# Required for Anthropic Claude and Claude Code
ANTHROPIC_API_KEY=your_key_here

# Required for OpenRouter models
OPENROUTER_API_KEY=your_key_here

# Worker configuration (dev environment)
ZE_BENCHMARKS_WORKER_URL=https://bench-api-dev.zephyr-cloud.io
ZE_PUBLIC_API_URL=https://bench-api-dev.zephyr-cloud.io
ZE_PUBLIC_VITE_WORKER_URL=https://bench-api-dev.zephyr-cloud.io
ZE_BENCHMARKS_API_KEY=dev-local-key
```

## Node Version

Requires Node.js >= 24 (see `engines` in [package.json](package.json))

## Testing Changes

When modifying harness or evaluators:
1. Build the package: `pnpm build:<package-name>`
2. Run a test benchmark: `pnpm bench test-suite <scenario> --tier L0 --agent echo`
3. Verify evaluator output and scoring
4. Check Worker API for stored results

## Adding New Features

**New Agent Adapter**:
1. Create [packages/agent-adapters/src/<name>.ts](packages/agent-adapters/src/<name>.ts)
2. Export from [packages/agent-adapters/src/index.ts](packages/agent-adapters/src/index.ts)
3. Extend switch in [packages/harness/src/cli.ts](packages/harness/src/cli.ts)
4. Build: `pnpm build:adapters`

**New Evaluator**:
1. Add to [packages/evaluators/src/evaluators/](packages/evaluators/src/evaluators/)
2. Export from [packages/evaluators/src/index.ts](packages/evaluators/src/index.ts)
3. Register in harness evaluation pipeline
4. Build: `pnpm build:evals`

**New Benchmark**:
1. Follow structure in [CONTRIBUTING.md](CONTRIBUTING.md) and [docs/ADDING-BENCHMARKS.md](docs/ADDING-BENCHMARKS.md)
2. Create suite/scenario directories
3. Add `scenario.yaml`, `oracle-answers.json`, `repo-fixture/`
4. Create L0-L3 prompt tiers
5. Test with: `pnpm bench <suite> <scenario> --tier L1 --agent echo`

## Important Notes

- All benchmark results stored in Cloudflare D1 via Worker API
- Never use `-uall` flag with git status (memory issues on large repos)
- Diff artifacts ignore specific patterns - see [packages/harness/src/runtime/diff.ts](packages/harness/src/runtime/diff.ts)
- Oracle answers (`oracle-answers.json`) are authoritative for future evaluators
- Scenario weight overrides apply but only current evaluators contribute to scores
- For research/exploration of codebase structure, examine suite directories and scenario configurations
