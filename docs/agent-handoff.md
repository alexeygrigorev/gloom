# Implementation-agent handoff

**Updated:** 2026-09-10  
**Use after:** [owner preparation](preparation.md).  
**Scope:** Implement the tool, not another set of specifications.

The private owner configuration and secrets must be available in the actual execution environment. Do not paste their values into the task. This repository currently contains documentation/templates only; build/provision/test commands below are requirements for the implementation, not existing features.

## Copy this task to the implementation agent

```text
Implement Gloom in this repository as a complete personal Windows recording
and sharing tool. Read AGENTS.md, docs/specification.md (v0.2),
docs/implementation-plan.md, and docs/preparation.md first. Read the private
owner configuration from GLOOM_OWNER_CONFIG_FILE or config/owner.local.toml,
when supplied. Credentials are in the execution environment's secret facility,
not in this prompt.

The PRIMARY path is Windows recording -> concurrent upload of completed media
segments -> private Cloudflare R2 -> Gloom's player on a stable share link.
YouTube is SECONDARY: explicit Save to YouTube -> private resumable upload ->
manual review/publication later. No Google account is required for the primary
product. Do not implement the superseded YouTube-first design.

Deliver the Rust-based desktop app, a tested capture/packaging pipeline,
persistent local recording and upload recovery, the Worker/static player,
D1 metadata/migrations, R2 upload and authorized cached playback, basic library
and retention controls, and the optional YouTube exporter. Reuse established
capture/encoding/player components; do not build a SaaS platform or editor.

Prove concurrent segment output on Windows before polishing the UI. OBS
start/stop plus upload of a finished file does not meet the normal workflow.
Keep local originals recoverable. System audio must be independently optional
and off by default. Show backlog honestly when instant sharing is impossible.

Start with redacted preflight checks. Implement idempotent setup, build,
deployment, smoke-test, backup/restore, and data-preserving rollback commands.
Provide an executable Windows build/installer or portable distribution with
checksums and clear runtime dependencies, not only source files.

Implement in an implementation branch and open a PR when repository access
permits. Use the owner configuration to determine whether live changes are
authorized and which environments/resources are in scope. Provision/deploy
only the named Gloom resources with the supplied credentials. Do not buy
products, change subscriptions/nameservers, alter unrelated resources, or
publish/delete actual course material. If live deployment is not authorized,
finish the implementation and deployment automation without making live changes.

Keep R2 buckets private and validate access before returning cached media.
Separate application runtime credentials from deployment credentials. Never
commit/log secrets or recordings. Preserve existing secrets and data on reruns.
Use synthetic fixtures and development resources; a real YouTube test upload
requires the explicit allow_live_test_upload setting and is always private.

Ordinary recordings must make no YouTube requests unless export was selected.
Upload completion never publishes a YouTube video, switches the existing Gloom
link to YouTube, or deletes the R2 copy. Manual approval, playback-backend
selection, and cleanup are independent operations. Implement the official
manual-upload fallback for API publication restrictions without blocking Gloom.

Make reasonable engineering decisions from the specs instead of repeatedly
asking about details already settled. Record meaningful deviations. Missing
credentials, unavailable Windows hardware, or a Google audit must not prevent
all other implementation and automated testing. Do not work around missing
permission or pretend a mocked integration was tested live.

Run the available unit/integration/browser/build tests and authorized live
checks. Include evidence of segments uploaded during capture, stop-to-playable
measurements, recovery tests, authorization on cache hits, and primary operation
with Google entirely disconnected. Do not stop at a scaffold or happy-path demo.

Finish with a concise completion report containing the PR/commit, Windows
artifact, deployed URL and resource inventory when applicable, exact tested
commands/results, current cost assumptions, and remaining owner-only actions.
Clearly separate implemented, built, deployed, live-tested, and blocked items.
Never label unavailable tests or provider approvals complete.
```

## Inputs the agent should discover, not ask you to repeat

Read these from the prepared config/secret environment: account and zone identifiers, allowed hostnames, development/production bucket names and jurisdiction, actual Workers plan, allowed environments, deployment authorization, soft budget, local storage paths, retention defaults, Windows test access, and optional YouTube client/channel/test consent.

Use the example config's defaults for ordinary preferences unless your private file overrides them. Empty account/hostname identifiers are not working values. The agent should report missing live inputs once, proceed with work that can be done safely, and never guess credentials or resource ownership.

The agent generates application owner credentials/signing keys securely, discovers/creates authorized D1/Worker IDs, installs secrets and bindings, and writes a local inventory. You should not need to hand-write a database schema, Worker configuration, HLS command, player integration, or deployment workflow.

## Required deliverables

| Deliverable | Evidence |
| --- | --- |
| Source implementation | Reviewable branch/PR, pinned dependencies, no secret/media leakage |
| Windows distribution | Actual build artifact and checksum; signed/unsigned status; dependency/install notes |
| Working primary service | Authorized deployment URL, private R2 origin, D1 schema, tested player/API/cache |
| Repeatable automation | Working preflight/provision/build/deploy/test/backup/restore/rollback commands |
| Recorder reliability | Real capture/audio and concurrent-upload results where Windows access is available |
| Access and retention | Tests for cache-hit authorization, token renewal/revocation, protected cleanup |
| Optional YouTube feature | Implemented disabled/configured states, mocked tests, and live results only when consent/setup allows |
| Operations guide | Exact commands, credentials rotation/recovery, resource inventory, usage/budget notes |
| Honest closeout | Complete versus blocked/unverified capability list, not a blanket everything-is-ready claim |

A source-code-only delivery does not satisfy a deployment-authorized task with working credentials. Conversely, code completion does not authorize cloud changes when your configuration forbids them.

## Completion report template

```text
Commit / PR:
Windows artifact and checksum:
Production URL (or not deployed and exact reason):
Private deployment inventory location:

Capability                     Implemented  Built/tested  Live-verified  Status
Windows capture/audio
Concurrent R2 upload
Gloom playback + seeking
Cache-hit authorization
Offline/restart recovery
Retention + restore/rollback
YouTube private export
YouTube manual approval observation
Explicit playback-backend switch

Measured stop-to-playable results and test conditions:
Commands actually run and results:
Actual environments/resources changed:
Expected recurring cost assumptions and usage limits:
Known defects/unsupported combinations:
Owner-only actions still needed (and why):
```

Populate the report with observed results, not checkmarks copied from the specification. An untested feature can be implemented without being verified; those are different claims.

## Expected owner interaction after coding

Even with all pre-work done, you may need to approve an installer/UAC or microphone/screen prompt, import the protected application credential, and perform an actual recording test on your PC. With YouTube enabled, you also complete the app's browser consent and any account/API review required by Google. Approving actual course videos later is intentional behavior.

The agent must automate the surrounding engineering and avoid unnecessary setup chores, but must not claim these owner/account decisions can be eliminated by code. YouTube setup or audit delays must never disable the working R2-first application.
