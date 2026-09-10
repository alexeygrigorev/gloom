# Gloom implementation plan

**Updated:** 2026-09-10  
**Status:** Planned implementation, not completed work.  
**Product source of truth:** [specification.md](specification.md), version 0.2.

The primary deliverable is **record on Windows -> upload during recording -> R2-backed Gloom share link**. YouTube is an optional export that cannot block the primary release. This plan replaces the earlier YouTube-first build order.

Read [preparation.md](preparation.md) for owner-only setup, [agent-handoff.md](agent-handoff.md) for the executable assignment, and the private owner configuration for live-deployment authorization. Work in dependency order, not to an assumed calendar estimate.

## Milestone 0 — Preflight and truthful capability report

Implement configuration parsing and a redacted preflight before creating remote resources.

Check repository access, toolchains, Windows execution/test capability, secret presence, Cloudflare account/bucket/jurisdiction access, allowed environments, hostnames/zone, actual plan, and resource-name collisions. Validate inputs without printing secrets or exposing private media. An active deployment token is not proof of every required permission.

Report readiness separately for:

- Local implementation and mocked tests.
- Windows build and interactive capture/audio tests.
- Development deployment and custom-domain cache testing.
- Production deployment.
- Optional live YouTube export/publication testing.

Missing Google configuration is informational for the primary product. Missing Cloudflare credentials blocks live deployment, not coding. Missing interactive Windows access blocks claims about real capture/audio behavior, not all implementation work. Gather genuinely missing owner-only actions into one actionable report rather than repeatedly asking questions already answered in the config.

**Exit:** validated configuration, a scoped resource plan, and an explicit list of capabilities that can be tested in the available environment. No subscription changes, DNS migrations, real-media publication, or destructive operations.

## Milestone 1 — R2 playback vertical slice from synthetic media

Implement the smallest useful deployed path before the recorder UI.

### Work

1. Create the Rust workspace/local SQLite model and a small TypeScript Worker/static player project. Use current supported dependencies, pin the validated versions, and commit lockfiles.
2. Add local Cloudflare development support, D1 migrations, and separate development/production configuration. Bind only the approved private buckets.
3. Implement owner authorization, idempotent recording creation, scoped upload authorization, acknowledged segment batches, finalization, share capabilities, and metadata/access updates.
4. Package a synthetic/local test video into HLS, upload directly to R2, and publish the final manifest only when its referenced inventory is verified.
5. Implement the player/share page with authorization, manifest/segment access, basic controls, and recoverable error states.
6. Deploy to the authorized environment and test the custom-domain cache path. No public R2 endpoint may bypass the Worker.

**Exit:** an authorized viewer can play and seek a synthetic Gloom-hosted recording while the desktop uploader is closed. Unauthorized access fails on both cache misses and hits. No Google account, YouTube API call, paid streaming service, or cloud transcoding job exists in this path.

## Milestone 2 — Prove concurrent Windows capture and packaging

This is a core requirement, not an optional performance enhancement.

### Work

- Test an existing capture engine first, with OBS as a candidate and a dedicated Gloom profile where applicable.
- Demonstrate that completed, keyframe-aligned playback segments become available and are uploaded while capture is still in progress, while a recoverable local copy is preserved.
- Validate system-audio OFF/ON independently of microphone input, timestamps, file completion signaling, and encoder load on the actual target machine.
- Test network loss, encoder/capture interruption, repeated start/stop, and an interrupted local media container.
- If the initial adapter cannot satisfy simultaneous recoverable recording and segmentation reliably, implement a supported alternative capture/output adapter. Record the measured reason. Do not silently replace the requirement with a completed-file uploader.

Do not assume a growing-file tailer or one particular Windows audio flag works without a real proof. FFmpeg/OBS integration, process buffering, device availability, and EOF behavior are implementation responsibilities, not owner pre-work.

**Exit:** a repeatable Windows proof produces usable R2 segments during capture and preserves local recovery. Add a short decision record with the selected engine/output topology, measured constraints, and rejected approach where relevant.

## Milestone 3 — Everyday desktop experience and persistent recovery

### Work

- Add the minimal tray/window UI: source selection, microphone meter, separate audio toggles, Record/Stop, hotkey, timer, destination/access indicators, and Copy Gloom link.
- Add a local library with preview, title editing, import, upload progress, retry/pause/cancel, and distinct local/R2/YouTube status.
- Snapshot the destination and access policy per recording. Gloom hosting is default; Local only causes no cloud transfer; YouTube export remains off unless explicitly selected.
- Persist upload acknowledgements and job leases; retry expired authorization safely. An app crash or lost finalization response must not create duplicate recordings or lose the source.
- Show finishing-upload state honestly when the connection cannot keep up. Opening a not-ready link must not produce a broken player or leak unfinished content.
- Implement first-run owner-token installation and OS-protected local secrets. Closing/minimizing/tray behavior must be explicit, with no hidden system service.

**Exit:** the owner can use normal Record/Stop/share without configuring OBS during everyday operation. With a healthy connection, measure the stop-to-playable target from the specification over repeated recordings. Offline recording works and resumes uploads after connectivity returns.

## Milestone 4 — Access, retention, and operational hardening

### Work

- Test every manifest/init/segment path for missing/expired authorization, correct revision scoping, cache-hit enforcement, and token refresh during multi-hour playback.
- Implement link rotation/revocation, owner-only mode, and non-enumerable metadata. Protect capability material from logs, referrers, and static assets.
- Implement permanent/ordinary/temporary classifications, Keep, independent-backup records, separate deletion scopes, and dry-run cleanup. No deletion policy is enabled merely by deploying.
- Add bounded, idempotent cleanup of safely abandoned upload artifacts. Never break a current playback mapping or remove an in-flight upload source.
- Add configuration/credential rotation, diagnostics, inventory export, local/metadata backup and restore, deployment rollback, and usage reporting.
- Validate readable code text, audio synchronization, playback speed, seeking, and start-up behavior across available browser/device tests. State the actually tested compatibility matrix.

**Exit:** the failure/recovery and access tests below pass, with actual versus mocked coverage clearly distinguished. The primary R2-hosted application is release-capable independently of YouTube.

## Milestone 5 — Optional Save to YouTube integration

Implement this feature as part of the requested tool, but make live configuration/testing optional.

### Work

- Add a disabled/unconfigured state that does not interfere with recording, and an explicit Save to YouTube action.
- Implement Desktop OAuth, OS-protected tokens, channel confirmation, local upload-file preparation, resumable uploads, polling, and review state.
- Create only private uploads; no publication schedule or automatic visibility change. Handle revocation, quota limits, network interruptions, and completion ambiguity without silent duplicates.
- Support Open in Studio, manual refresh, and official manual-upload/attach-link fallback.
- With owner-supplied credentials and explicit live-test consent, test the actual API project's publication capability. Do not confuse OAuth consent/testing state, channel eligibility, and API audit restrictions.
- Implement a separate explicit Use YouTube for this Gloom link action only for a verified approved asset. Preserve the R2 mapping for rollback and keep R2 deletion independent.
- Test a per-recording opt-in export choice; course tags must never silently cause transfers to Google.

**Exit:** mocked export tests pass with no Google setup. When live setup is supplied, report exactly which live upload/processing/publication checks passed. A Google policy block must leave the Gloom link functional and be labeled as a secondary-integration limitation, not a primary-app failure.

## Milestone 6 — Package, deploy, and hand over

Deliver a usable Windows artifact, not only source code or instructions to install a compiler. Build an installer or a clearly documented portable distribution with all non-system runtime dependencies available. State whether it is signed and supply checksums. Do not bundle credentials.

Complete authorized deployment using the provided environment and named Gloom resources. Verify TLS, static assets, D1 schema/bindings, private R2 access, runtime owner credential, upload authorization, cache behavior, and remote playback with the desktop stopped. Run the data-preserving rollback/restore exercise.

Deliver an accurate installation/operation guide and a completion report with separate columns for implemented, automated-tested, live-tested, deployed, and owner action needed. Include the resource inventory, application URL, build artifact location, measured timing results, expected cost model, and outstanding limitations. Do not mark hypothetical commands or unrun checks complete.

## Required automation interfaces

The following are deliverables for the agent to implement; they do not exist in this documentation-only repository yet. Equivalent clearly documented commands are acceptable when they are simpler, but a manual-only dashboard recipe is not sufficient.

| Proposed command | Contract |
| --- | --- |
| `scripts/preflight.ps1 -OwnerConfig <path>` | Read/validate config and permissions; redact all secrets; show distinct capability/blocker states |
| `scripts/bootstrap.ps1 -OwnerConfig <path> -Environment development -Plan` | Default-safe plan of named resources and generated configuration; no live mutations |
| `scripts/bootstrap.ps1 ... -Apply` | Idempotent authorized provisioning, D1 setup, bindings, secret installation, and private inventory; reject unauthorized environments |
| `scripts/build-windows.ps1` | Repeatable build/package with pinned dependencies and no embedded secrets |
| `scripts/deploy.ps1 -OwnerConfig <path> -Environment <name>` | Validate authorization, apply safe migrations, publish the named service, and retain rollback information |
| `scripts/smoke-test.ps1 -OwnerConfig <path> -Environment <name>` | Synthetic recording upload/playback/access/cache checks; cleanup only its own disposable test artifacts |
| Backup/export and restore commands | Preserve local library, provider mappings, and independent-source recovery; verify a restore |
| Rollback command | Revert application deployment without deleting video or blindly reversing destructive database migrations |

Generate environment-specific Worker config from the private owner file, with valid real IDs. Do not commit filled production secrets or fabricate resource IDs. Installing secrets must preserve prior values on rerun unless rotation was explicitly requested.

## Acceptance and failure matrix

| Test | Required result |
| --- | --- |
| Google setup completely absent | Primary app builds, runs, records, uploads to R2, and shares |
| Gloom-only or local-only recording | No YouTube network call or queued export; local-only also has no R2 transfer |
| Record remains active | Completed segments already exist in R2 |
| Stop with healthy uplink/no backlog | Target measured honestly; complete authorized playback, not merely a URL |
| Offline/slow connection | Durable local source, visible backlog, recoverable upload |
| System audio off while another app plays sound | Recording contains no system audio |
| Microphone removed | Visible error/warning, no silent device replacement |
| Upload URL expires or response is lost | Safe renewal/reconciliation, no false readiness or duplicate recording |
| Capture/app crash or disk full | Preserve recoverable bytes; show recovery, not silent success |
| Two app instances | No competing upload job for the same artifact revision |
| Unauthorized media request with a warm edge cache | Denied before cached bytes are returned |
| Long playback session | Authorization refresh and seeking continue to work |
| Share revoked or owner-only selected | New sessions blocked; existing sessions expire within the specified bound |
| Restart after finalization | Ready media still plays with the PC offline |
| YouTube selected for one item | Only that revision uploads privately; publication remains manual |
| YouTube locked-private/auth/quota error | Actionable export status; original Gloom link remains functional |
| YouTube becomes approved | Existing Gloom playback does not switch until owner selects it |
| Playback backend changes | Stable link remains; R2 copy is not automatically deleted |
| Cleanup runs | Does not delete protected originals, active jobs, or still-referenced playback |
| App/cloud rollback | Media and useful metadata survive |
| Package/log/repository scan | No real credentials, private recordings, or secret URLs included |

Use synthetic fixtures for automated tests and owner-approved disposable live media. Test long files, non-ASCII paths, cancellation, preflight permission failures, wrong environment/channel configuration, and configuration upgrades. Actual capture/audio tests require an interactive Windows environment; report them as pending when unavailable instead of inferring success from mocks.

## Suggested repository layout after implementation

```text
AGENTS.md
README.md
config/owner.example.toml
docs/
  preparation.md
  agent-handoff.md
  specification.md
  implementation-plan.md
  operations.md             # agent delivers validated commands/results
  decisions/                # short engineering decision records
crates/                     # Rust core, capture, desktop, optional export modules
web/                        # static player and share page
worker/                     # API/media authorization, bindings, D1 migrations
scripts/                    # setup/build/deploy/test/backup helpers
tests/                      # synthetic fixture generation and automated checks
.github/workflows/          # build/test; deployment only with explicit scope
```

Start with fewer crates/modules when simpler. Do not build a generic plugin system, public SaaS platform, editor, transcription service, team system, enrollment service, or Kubernetes deployment.

## Definition of done

A real Windows recording can be safely saved, uploaded during capture, and shared through Gloom's own R2-backed player; the owner can optionally export selected recordings privately to YouTube and approve them later. Provisioning/build/deployment are reproducible, local recovery and access control are tested, and all claimed live behavior has evidence. External account approval or unavailable hardware is reported precisely, never hidden behind a claim that everything was verified.
