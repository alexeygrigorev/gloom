# Gloom implementation plan

**Date:** 2026-09-10  
**Status:** Planned work, not completed implementation.  
**Source of truth:** [Product and technical specification](specification.md).

Build the smallest working vertical slice first. The default path must remain desktop -> YouTube, without an intermediate cloud bucket. Milestones are ordered by dependency, not by calendar estimates.

## Milestone 0 — Validate the actual YouTube workflow

This milestone is a release gate for automatic uploads intended for students, not a reason to block development of local recording.

### Work

- Create/configure the owner's Google project and Desktop OAuth client outside the repository. Confirm the actual destination channel and its longer-upload capability.
- Use a disposable, non-sensitive file to test desktop authorization, private upload, status polling, and the intended later manual publication flow.
- Check the API-project audit/publication capability described in specification section 2. Record the evidence, date, and remaining steps without committing project credentials or private video IDs.
- Validate token refresh across application restarts and the practical effect of the project's testing/production configuration. OAuth persistence and the API publication gate are separate checks.
- Confirm the console's current quota buckets against specification reference S10 rather than relying on old tutorials.
- Test an unlisted recording from a signed-out browser after explicit approval. Test embedding separately only when that feature is needed.

### Exit criteria

A short setup report classifies the installation as either **automatic private upload with validated later publication** or **publication blocked; official manual-upload fallback required**. A successful private upload alone does not pass the publication gate.

Do not promise an audit completion date, automate Studio to bypass restrictions, or silently replace YouTube with a paid destination. The owner can choose whether pursuing the audit is worth the saved manual upload step.

## Milestone 1 — Imported file to private YouTube upload

Implement a thin Rust command-line or minimal desktop application before adding capture controls. Importing an MP4 exercises the most important new workflow independently of the recorder.

### Work

- Initialize a local SQLite library, managed artifact directory, configuration, and protected secret-store integration.
- Register an existing recording with a stable ID, revision, title, destination, retention class, file size, and checksum.
- Add explicit auto-upload consent and a per-recording local-only override.
- Implement one persistent upload worker, resumable-session handling, privacy-safe metadata, and remote video-ID storage.
- Separate upload, processing, visibility, and review states in the UI/model.
- Add pause/resume/cancel, bounded retries, reauthorization, quota waiting, and a completion-unknown recovery screen.
- Provide local preview, Open in Studio, and the manual-upload/attach-video-URL fallback. Only accept supported YouTube URL forms, normalize the ID, and verify the linked asset where permissions allow.

### Exit criteria

A private upload survives an interrupted network connection and app restart without an automatic duplicate. Upload completion never releases the video. The workflow runs without AWS, Cloudflare, a hosted database, or a remote Gloom process.

## Milestone 2 — Reliable recording and automatic enqueue

### Work

- Add an OBS capture adapter using a dedicated Gloom scene/profile so configuration changes do not overwrite unrelated OBS work.
- Build the minimal tray/window UI: display/window selector, microphone selector and meter, independent system-audio toggle, Record/Stop, timer, destination indicator, and hotkey.
- Write a recoverable local master; finalize/remux and validate a separate upload artifact when needed.
- Atomically register the completed recording and enqueue it according to the captured destination-policy snapshot.
- Make recovery from interrupted capture visible. Never auto-upload an uncertain recovered fragment.
- Define tray behavior explicitly and support graceful exit/resume of the upload queue.

### Exit criteria

The owner can record a real coding explanation without interacting with OBS's configuration interface during normal use. It is saved even offline. The next available upload opportunity requires no extra action when automatic upload is enabled.

Measure CPU/GPU impact, dropped frames, small-text quality, audio synchronization, disk growth, and stop-to-local-preview time on the actual PC. Use these measurements to choose defaults, not an assumed universal bitrate.

## Milestone 3 — Review, publication, and retention

### Work

- Add library filters for uploading, processing, awaiting review, published, local-only, and needs-attention items.
- Implement refresh after Studio review and safe observation of external visibility changes. Never overwrite newer Studio edits from stale local metadata.
- Show unlisted-sharing implications and distinguish private-owner links from student-ready links.
- Bind approvals to the correct video ID and content revision. A replacement recording requires new approval.
- Add permanent/ordinary/temporary classifications, Keep, backup status, and a preview of any proposed cleanup.
- Keep local deletion separate from remote deletion; leave remote deletion in Studio initially.
- Export a portable recording-to-provider mapping and sanitized diagnostics.

### Exit criteria

An uploaded lesson can be reviewed later, manually released, verified, and linked into the course. No retry, restart, timer, or successful processing event can release an unapproved recording. Cleanup cannot delete the only course copy or an active job's source.

## Milestone 4 — Personal-release hardening

Run automated state/HTTP tests and a real Windows acceptance pass. Mock API failures rather than spending live quota on repeated destructive tests. Use a dedicated, non-sensitive live sample for integration checks.

| Test | Required outcome |
| --- | --- |
| System audio disabled while another app plays sound | Playback contains no system audio |
| Microphone disconnected | Visible warning; no silent device substitution |
| Recording starts offline | Local recording remains usable; upload queues |
| Network drops mid-upload | Resume from provider-confirmed progress or show a recoverable state |
| Process exits after final bytes but before response persistence | Reconcile; do not automatically insert a second video |
| Authentication is revoked | Pause safely and request reconnection without losing the job |
| Quota is exhausted | Defer with an actionable message, not an aggressive retry loop |
| Disk becomes full or capture crashes | Preserve recoverable material and surface the failure |
| PC sleeps with uploads pending | Resume after wake/relaunch; do not claim off-device execution |
| Provider accepts file but processing fails | Keep the source; never show it as student-ready |
| Publication gate is blocked | Clear manual-upload fallback; no misleading visibility-toggle advice |
| Studio changes visibility/metadata | Refresh accurately without undoing the owner's edits |
| Media is replaced after approval | New revision remains unapproved |
| Cleanup is enabled during upload or before backup | Protected artifacts are not deleted |
| Diagnostics/configuration are inspected | No tokens, upload-session secrets, or private media in Git/logs |

Also test a multi-hour lesson, repeated start/stop, non-ASCII filenames, two simultaneous app launches, database migration recovery, wrong-channel selection, and linking the wrong manually uploaded video.

### Exit criteria

The owner completes repeated real recording/upload/review sessions and a deliberate recovery exercise. Document remaining defects and measured behavior. A successful happy-path demo is not a reliability claim.

## Deferred extensions — Only after the default workflow is useful

### Stable share pages

Add a small approved-metadata mapping and static YouTube embed pages only when stable Gloom URLs provide value. Keep private-library metadata out of deployable assets. Continue serving YouTube media directly through its supported player, not through Gloom.

### R2 private backup

Add an opt-in backup adapter with checksums and a restore test. Backing up a file must not make it publicly accessible or automatically enable a player/CDN. Track only the selected retained bytes in the cost model.

### R2 alternative/near-instant sharing

Treat this as a separate product slice: local media packaging, upload during capture, cache configuration, player integration, access control, expiry, and a distinct approval policy. First prove that real recordings need this instead of YouTube. No automatic mirror of every YouTube upload.

### In-app publication and other features

Only after the publication gate is validated, consider explicit in-app approval, playlist management, camera overlays, or a native Windows capture adapter. Each addition needs a concrete workflow benefit and its own permission/reliability review.

## Suggested repository organization when implementation starts

```text
README.md
docs/
  specification.md
  implementation-plan.md
  setup.md                 # create from validated setup, no credentials
  decisions/               # short records of tested architectural choices
crates/
  gloom-core/              # library, jobs, policies, state transitions
  gloom-youtube/           # OAuth, resumable transport, remote status
  gloom-capture/           # OBS adapter, later native adapter if justified
  gloom-desktop/           # UI/tray and application composition
 tests/                    # proposed; finalize layout with the workspace
```

The layout is a proposal, not a requirement to create every package immediately. Start with fewer crates if that makes the first working slice simpler. Select the desktop UI toolkit after validating tray/hotkey/media needs; Rust remains the coordinator, not a mandate to rewrite capture or encoding.

## Definition of done

The delivered personal tool supports **record -> durable local file -> automatic private upload -> manual review/publication -> verified share link**, or clearly identifies the supported manual-upload fallback when the YouTube setup blocks publication. It runs without mandatory Gloom-hosted infrastructure, protects originals, and makes failures recoverable and visible.
