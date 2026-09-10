# Gloom: product and technical specification

**Version:** 0.2  
**Date:** 2026-09-10  
**Status:** Implementation requirements, not implemented or validated behavior.

## 1. Product decision and precedence

Gloom is a personal, Windows-first Loom replacement. **Gloom-hosted recording and sharing is the primary path: local recording -> Cloudflare R2 -> Gloom's player and stable share link. YouTube is an optional, explicitly selected export destination.**

This revision supersedes the earlier YouTube-first specification. In particular, it restores concurrent uploads and near-immediate sharing to the core scope. YouTube credentials, upload capability, API audits, and availability must never be dependencies of ordinary recording or Gloom playback.

The owner should be able to use Gloom indefinitely without connecting Google. Some recordings remain on Gloom only. Others, especially course material, can be exported to YouTube privately, approved later, and optionally served through YouTube instead of R2.

Priorities: preserve recordings; make the Record/Stop/share workflow reliable and simple; minimize recurring costs; keep destinations under explicit owner control; avoid building a multi-user commercial product.

### 1.1 Confirmed requirements

Windows recording, screen/window capture, microphone capture, independently optional system audio, durable local copies, upload during recording, working links soon after Stop, global playback, permanent and temporary recordings, and inexpensive serverless hosting. Rust is preferred where useful. YouTube export must upload privately and require manual approval for publication.

### 1.2 Proposed implementation defaults

| Concern | Default |
| --- | --- |
| Desktop | Rust coordinator and a small Tauri UI; Windows x64 first |
| Capture engine | Reuse an established engine if it passes the concurrent-output proof; do not require a custom codec implementation |
| Cloud | Cloudflare R2 Standard, one Worker with static assets, one D1 metadata database per environment |
| Playback | An established open-source HLS player, such as Video.js; no paid player service |
| Media preparation | Local encoding/packaging, one tested 1080p/30 fps rendition initially |
| Upload | Completed short segments uploaded directly to R2 with bounded, short-lived upload authorization |
| R2 visibility | Private buckets; all viewer access goes through the authorized Worker path |
| Sharing | Revocable anyone-with-the-link sharing, with an owner-only option |
| YouTube | Disconnected/off by default; explicit Save to YouTube action |
| System audio | Off by default, independent of microphone selection |
| Retention | No automatic deletion until a policy is explicitly enabled |
| Infrastructure | No AWS, VM, container host, cloud transcoder, Cloudflare Stream, or hosted database server |

These are proposed technical choices, not claims that the owner selected a UI framework, Windows version, bitrate, or particular capture dependency. The agent may change implementation details with a short decision record, but must not demote R2 sharing or remove concurrent upload to make implementation easier.

## 2. Recording and destination behavior

### 2.1 Primary workflow

```text
Select screen/window, microphone, and optional system audio
  -> Record to recoverable local storage
  -> Produce and upload completed playback segments while recording
  -> Stop and commit the remaining segments/manifest
  -> Copy a stable Gloom link
  -> Viewer opens the Gloom player; media is served from R2 through Cloudflare
```

After the owner's initial setup consent, Gloom hosting is enabled for normal recordings. No repeated hosting approval dialog is required. The destination indicator remains visible. Local only disables all cloud upload for that recording. Owner only permits R2 storage but withholds viewer access.

A share link may be allocated at recording start, but before completion it must show an honest not-ready state. No public live broadcast is implied. Do not expose an in-progress lesson merely because a URL exists.

### 2.2 Secondary workflow

```text
An existing recording -> Save to YouTube
  -> Confirm destination channel and private upload metadata
  -> Persistent background upload performed by the running desktop app
  -> YouTube processing -> Awaiting YouTube approval
  -> Owner reviews/releases it in YouTube Studio
  -> Gloom observes the approved visibility
  -> Optional, separate Use YouTube for this Gloom link action
```

Pressing Save to YouTube authorizes transfer to Google, not public/unlisted publication. A per-recording Also upload privately to YouTube choice may be selected before recording for convenience; it defaults off. A course tag alone must never enable it. A reusable course preset requires explicit opt-in and must show that destination before capture.

No implicit YouTube requests, synchronization, or export jobs may run for Gloom-only/local-only items. Account connection, restarting, retries, and editing a title are not export consent.

### 2.3 Three independent decisions

| Decision | Example | What it must NOT imply |
| --- | --- | --- |
| Where copies exist | Local plus R2, optionally a YouTube copy | Permission to publish the YouTube copy |
| What the share page plays | R2 by default; approved YouTube asset after explicit switch | Permission to delete R2 or the master |
| What may be deleted | Explicit local/R2 retention policy | Deletion from another destination |

Export completion never changes the existing Gloom link, its access mode, or its playback backend. A private or policy-blocked YouTube video must not replace a functioning R2 video.

## 3. Meaning of ready and measurable targets

Distinct states must be visible: recording, saved locally, finishing upload, Gloom ready, export queued/uploading, YouTube processing, awaiting approval, approved, and needs attention. YouTube status is a secondary badge, not the main recording state.

**Primary readiness:** the local recording is recoverable, all referenced media exists remotely, the final playlist is committed, and an authorized viewer can start playback and seek. A URL pointing to a spinner or a partially missing playlist is not ready.

Upload during capture is mandatory for the intended low-latency workflow. Uploading a single completed multi-GB MP4 after Stop is acceptable for file import/recovery, but does not satisfy normal recording acceptance.

**Proposed target to validate:** Stop-to-playable-Gloom-link <=10 seconds in ten consecutive representative recordings, with a healthy connection, no upload backlog at Stop, and sustained uplink at least twice the measured average encoding rate. Report the actual hardware, codec, duration, bitrate, network conditions, and measured times. This is an acceptance target, not an achieved guarantee.

Offline or insufficient-bandwidth recording must remain safe. Show remaining upload work and resume later; never promise instant remote playback under those conditions. Closing the app, sleep, or PC shutdown can stop local upload work. Tray operation is opt-in and visible; there is no always-running remote uploader.

## 4. Capture and local media pipeline

### 4.1 Required controls and output

Provide display/window selection, a microphone meter, independent microphone/system-audio toggles, Record/Stop, elapsed time, and a configurable hotkey. Show the actual destination and recording indicator. Do not silently switch microphones after device loss. Camera overlay, region capture, and editing are deferred.

Use hardware encoding where available and preserve readable code text, cursor motion, and synchronized audio. Start testing with H.264 video and AAC audio, but choose quality from real recordings rather than enforcing a universal 3 Mbps bitrate. One HLS rendition does not provide adaptive quality; a lower-bitrate rendition is a later, optional local-encoding improvement.

Produce a recoverable local master or an equivalently recoverable complete local segment set throughout capture. Do not rely on finalizing ordinary MP4 as the only protection against process failure. A reusable capture engine such as OBS and an established packager such as FFmpeg are preferred over writing codecs. OBS exposes remote control, and FFmpeg supports HLS/fMP4 packaging. [R1][R2]

### 4.2 Mandatory engine proof

Before polishing the UI, prove that the selected Windows capture path can simultaneously preserve recoverable local media and emit completed, timestamp-correct, keyframe-aligned segments while capture is still running.

**OBS WebSocket start/stop plus a watcher that waits for the final file is not sufficient.** Do not assume that remuxing/tailing a growing file works reliably without testing EOF behavior, buffering, discontinuities, restarts, and audio/video timestamps. If the OBS integration cannot meet the contract simply, use another supported capture/output adapter and document the decision. Keep Rust coordination and the user workflow stable.

Suggested packaging is approximately six-second independently addressable HLS segments. Emit temporary files first and atomically mark them complete so the uploader cannot read half-written segments. Use bounded memory and keep capture ahead of upload/remux work. The exact segment duration is adjustable based on tests and request cost.

### 4.3 Local recovery and import

Persist the recording, artifact inventory, and retry queue in local SQLite. Use recording ID plus content revision as identifiers, not filenames/titles. Hash and validate completed artifacts. Keep each upload source unchanged while its job exists.

On restart, reconcile unfinished captures and jobs. Recovered/incomplete recordings require owner review before sharing. Support importing existing files through the same packaging and upload pipeline. Derive a standalone upload MP4 locally for YouTube only when needed; avoid an unnecessary lossy re-encode.

## 5. Serverless architecture

```text
Windows recorder (Rust + UI + capture adapter)
  |-- recoverable originals/segments + local SQLite queue
  |-- small authenticated requests --> Gloom Worker API
  |                                  |-- D1 metadata/state
  |                                  |-- issue scoped upload URLs
  |-- completed segments --------------------------> private R2
  |
  `-- explicitly selected export ------------------> YouTube

Viewer --> Worker custom domain --> static share page/player
                  |
                  |-- validate share capability / playback authorization
                  |-- serve authorized manifest and media from edge cache
                  `-- cache miss --> private R2 binding
```

Use one origin for the share page/API/media initially to simplify browser behavior. Development and production have separate buckets, databases, credentials, and resource names. The agent must generate repeatable provisioning, bindings, migrations, deployment, and rollback scripts rather than leave a list of dashboard operations for the owner.

D1 stores small authoritative remote metadata, not video bytes. It is serverless and has included usage on Workers plans. Local SQLite remains the durable source for local jobs; remote state must not depend on the PC remaining online after upload. [R3]

Workers should authorize and stream, not encode long recordings or buffer complete videos. Upload bytes normally go directly from desktop to R2. Runtime access uses Worker secrets/bindings; deployment credentials never ship in the desktop installer or website.

## 6. Upload and finalization contract

An initial API surface, subject to sensible implementation refinements:

| Operation | Required semantics |
| --- | --- |
| Create recording | Owner-authenticated, idempotent create with content revision, access mode, retention, and policy snapshot |
| Authorize segment batch | Bound to this recording/revision, specific keys, allowed types, sizes, and short expiration |
| Acknowledge segment batch | Verify remote existence/length and supported integrity evidence; record acknowledged inventory idempotently |
| Complete revision | Verify complete inventory and manifest references, then atomically mark ready; safe to retry |
| Update metadata/access | Owner-authenticated; optimistic concurrency and no implicit destination changes |
| Create/rotate/revoke share | Separate from upload; reject unauthorized access immediately at session creation |
| Issue playback session | Validate capability/access/ready state, return narrowly scoped expiring playback authorization |
| Select playback backend | Require explicit owner choice and verified eligible YouTube asset when switching away from R2 |
| Schedule deletion | Record explicit scope; protect references, pending jobs, backups, and grace periods |

Use immutable, revisioned media paths and cryptographically random share capabilities. Validate paths against the known recording namespace; no arbitrary object-key signing, arbitrary URLs, or unauthenticated uploads. Rate-limit mutation endpoints, constrain job/segment counts and expected bytes, and reject malformed/oversized input.

R2 S3 presigned URLs use the S3 endpoint, not the Worker/custom domain. Their expiration and object scope must be tested. Browser CORS is relevant only to browser-origin S3 access; it is not authorization and does not replace private-bucket controls. [R4]

Persist acknowledgements while recording. Batch remote verification so Stop does not trigger thousands of sequential requests or exceed Worker subrequest limits. Confirm finalization with explicit state rather than inferring success from a client timeout. Checksums must use a tested provider-supported mechanism; do not assume an ETag is a SHA-256 checksum.

Retry transient failures with bounded exponential backoff and jitter. Expired upload URLs should be renewed only for still-authorized jobs. Restart recovery must avoid duplicate recording creation. A cancelled or abandoned revision is not automatically shared; later cleanup must account for outstanding URL lifetimes.

## 7. Playback, cache, and access

### 7.1 Private origin and caching

Keep R2 public access and `r2.dev` disabled. Attach the production hostname to the Worker, **not directly to the protected bucket**. Use the Worker R2 binding and Cache API for media delivery. The R2 Cache API example requires a custom domain/route; `workers.dev` is suitable for development but not proof of this production caching behavior. [R5][R6]

Authorization must run before returning any cached media. Cache immutable bytes under canonical internal keys after validation; do not globally cache the entire authorization response or let another cache layer bypass access checks. Never cache user-specific metadata, credentials, or signed manifests as shared public responses. Test both cache hits and misses with missing, expired, and revoked credentials.

Use appropriate content types, lengths, cache directives, and range handling. Implement explicit diagnostics for cache hits in the development test path rather than assuming one particular response header proves the cache implementation. Demonstrate a same-location hit that avoids an R2 read. Global caching is on demand, not permanent replication of every file to every location.

### 7.2 Access modes

**Anyone with the link** is the simple default, not enrollment-based security. The link contains a high-entropy capability separate from the recording ID. Prefer a fragment-carried capability exchanged by the share-page JavaScript so it is not included in ordinary server request URLs/referrers. The backend stores capability hashes; the desktop protects recoverable link material or offers rotation when it cannot recover an existing link.

**Owner only** disables viewer sessions and remains usable for owner preview. **Local only** means no R2 or YouTube transfer. These names must not be conflated.

Suggested playback-session lifetime is ten minutes, scoped to one recording/revision. Session refresh checks current share/access state; media requests validate signatures/expiration before cache access. Revocation prevents new sessions immediately and expires existing access within the documented session lifetime plus clock skew. It cannot erase bytes already downloaded. Test token refresh during multi-hour playback and seeking.

The agent must explicitly propagate playback authorization to every playlist/init/segment resource; a query string on a master playlist is not automatically inherited by its children. Test native Safari HLS as well as the JavaScript player path. Do not require students to install software.

Protect private metadata from unauthenticated enumeration. Use no-referrer, noindex, a restrictive content-security policy, and no unnecessary third-party scripts on R2 share pages. These are defense-in-depth, not substitutes for authorization.

### 7.3 Player functionality

Use a maintained player rather than implementing demuxing/decoding. Include play/pause, seeking, volume, playback speed, full screen, responsive layout, a useful error/retry state, title, local resume position, and timestamp sharing. Do not add a mandatory paid video service. Verify Chrome/Edge/Firefox on desktop and Safari/mobile playback with actual encoded test media. [R7]

## 8. YouTube export: optional, implemented separately

### 8.1 Consent and authentication

Gloom launches and records normally with no Google configuration. Save to YouTube prompts connection only when selected. Show the channel, video revision, metadata, and private-transfer intent. A disabled/unconfigured integration is a valid primary-product installation, not an onboarding failure.

Use a Desktop OAuth client, system browser, loopback redirect, PKCE, state validation, and OS-protected tokens. Request the minimum upload/read scopes required; start by validating `youtube.upload` and `youtube.readonly`. Never use an ordinary service account for a personal channel. Keep Google refresh tokens on the desktop, not in R2 metadata, Worker configuration, CI artifacts, or Git. [R8]

### 8.2 Upload and review

Upload directly from a finalized local artifact with the documented resumable protocol. R2 is not used as an unnecessary staging hop. If only Gloom-hosted segments remain, reconstruct a valid upload artifact through the owner's authorized download path; never extract media from YouTube. Persist provider-confirmed offsets and video IDs. If the final response is lost, reconcile the session before retrying; use `completion_unknown` rather than automatically creating a duplicate. [R9]

Always create exports as private with subscriber notifications disabled. No timer, successful upload, app restart, course tag, or processing completion may publish the video. Review occurs in YouTube Studio initially; Gloom refreshes observed status without overwriting Studio edits. Approval is bound to the specific remote video and content revision. [R10]

A manual decision to publish in Studio and an observed eligible visibility can mark the export approved. Public/unlisted visibility must be verified before enabling Use YouTube for this link. Unlisted links are forwardable, not enrollment restrictions. YouTube's embedded player must be used for YouTube-backed playback. [R11][R12]

### 8.3 Provider gates do not block Gloom

Unaudited API-project uploads may be locked private. Manual visibility changes in Studio do not bypass that lock; the documented fallback is upload through the official YouTube site/app or a verified API service, or pursue an API audit. Record the actual project's tested capability. Do not label a locked-private asset as merely waiting for a publication toggle. [R10][R13]

OAuth consent verification, OAuth publishing/testing state, channel longer-upload eligibility, and YouTube API audit are separate concerns. External OAuth projects in Testing can issue seven-day refresh tokens for these scopes. Handle reauthorization and explain the setup state; changing OAuth to production is not proof of API audit approval. [R14]

Confirm the actual channel's longer-upload eligibility and the project's current quotas at setup. Use configurable retry/quota handling rather than embedding remembered quota numbers. Test a disposable export with explicit owner permission; automated CI must not upload to the real channel.

## 9. Playback migration and retention

The stable Gloom URL initially resolves to R2 playback. After YouTube approval, an explicit Use YouTube for this link action may change the backend without changing the URL. Keep the prior mapping for rollback and do not switch owner-only material to forwardable YouTube access without an explicit access-policy confirmation.

Show R2 copy retained after a backend switch. Removing it is a separate opt-in action after a grace period, a playback check, and preservation of an independent recoverable master for permanent lessons. Keeping both copies costs R2 storage; YouTube saves delivery requests only when the link actually plays YouTube. Do not silently fall back to another host after access changes or deletion.

| Category | Initial retention behavior |
| --- | --- |
| Permanent course material | No automatic expiry; independent recoverable copy required before source cleanup |
| Ordinary recording | Keep until explicitly deleted or a policy is enabled |
| Temporary recording | Offer 30/90-day expiry and Keep override; no hidden default deletion |
| Interrupted/failed capture | Retain for review/recovery |
| Abandoned uploaded segments | Reconcile first; scheduled cleanup with a documented safe grace period |
| Derived export MP4 | Removable after success when a usable master remains |

Separate local deletion, Gloom/R2 deletion, link revocation, and YouTube deletion. Remote YouTube deletion remains a Studio operation initially. Revoking a Gloom URL does not revoke a separately distributed YouTube URL. A YouTube copy is not an independent original-file backup.

Do not use archive/infrequent-access tiers for the only playable copy. Defer archive-tier automation until measured storage savings justify it. Retention jobs must be idempotent and protect in-flight jobs, permanent material, and assets still referenced by active playback mappings.

## 10. Persistence, security, and operations

Persist recording IDs/revisions, policy snapshots, artifact roles/checksums, upload acknowledgements, desired versus observed remote state, primary playback backend, hashed share capabilities, export consent, review provenance, retention class, and backup verification. Keep capture, R2 readiness, YouTube upload, YouTube visibility, approval, and deletion as separate state dimensions.

The desktop owner credential is a high-entropy, revocable application credential generated during setup and stored with OS protection. The Worker stores its verifier and server-side signing keys as secrets. Do not use the Cloudflare deployment API token as the application's runtime login. No public signup or exposed unauthenticated administrative UI is required.

Restrict desktop IPC and local control to necessary capabilities; authenticate OBS control and bind it locally. Never interpolate user metadata into shell commands. Validate filenames and render titles as text. Log structured states/errors without tokens, presigned URLs, private titles, or secret-bearing request bodies. Telemetry is off by default.

Provide database migrations, configuration validation, sanitized diagnostics, usage estimates, resource inventory export, credential rotation, deployment rollback, and a tested restore/export path. Never delete data during a deployment rollback. Immutable originals should survive an application upgrade. Cloud resource names must distinguish development from production.

## 11. Operating costs and budget discipline

Planning assumptions, USD before tax, checked 2026-09-10:

- R2 Standard storage is $0.015/GB-month after 10 GB-month of free storage. It includes monthly operation allowances and has no egress charge. [R15]
- Workers Paid starts at $5/month with included requests and CPU; Free can be used for development but has daily limits. Static assets and D1 have their own included allowances. Cache hits in this authenticated design still require the chosen authorization/request path. [R3][R16]
- At an illustrative 3 Mbps, 30 recorded hours is about 40.5 GB (`hours * Mbps * 0.45`). That is approximately $0.50/month of R2 storage before extra renditions/backups and operation overages. With a $5 Worker plan, a $6-10/month initial planning budget is reasonable at modest usage, not a fixed-price or unlimited-service promise.

Primary traffic is R2-hosted unless an owner switches the playback backend. Count stored copies, segment requests, database queries, CPU, logs, and retries. Track development plus production against shared account allowances. Do not create another paid subscription, cloud transcoder, streaming product, or third-party service without authorization.

Budget alerts and configuration limits are not a guaranteed billing hard cap. Any application upload/storage limits must be described precisely; playback request charges and abusive traffic still need monitoring. Do not delete lessons automatically to enforce a cost target.

## 12. Release acceptance

The implementation is not complete until evidence covers:

1. With no Google credentials, a real Windows recording becomes a playable Gloom link; service-only recordings produce no Google network calls or export jobs.
2. Completed segments demonstrably reach R2 during capture, and the readiness target in section 3 is tested rather than replaced with upload-after-Stop behavior.
3. Microphone-only, microphone plus system audio, and silent recording behave as selected. Multi-hour recordings preserve readable text, synchronization, and seeking.
4. Network interruption, restart, disk-full/device-loss scenarios, expired upload authorization, and ambiguous completion preserve recoverable state and avoid silent duplication.
5. Unauthorized metadata/manifest/segment access fails, including on cache hits. A custom-domain cache test avoids redundant R2 reads. Revocation and long-video token refresh meet their documented bounds.
6. Save to YouTube exports only the selected revision privately; approval and backend switching are separate. A blocked export leaves the existing Gloom recording usable.
7. Local/R2/YouTube deletion and expiry are independent. Cleanup cannot remove the only permanent-course copy or an active upload source.
8. The agent delivers a Windows build, deployed service when authorized, repeatable setup/deploy scripts, tests, resource inventory, usage notes, and exact remaining owner actions. Mock-only tests are not labeled real Windows/provider validation.

Detailed sequencing and owner prerequisites are in [implementation-plan.md](implementation-plan.md), [preparation.md](preparation.md), and [agent-handoff.md](agent-handoff.md).

## References

Primary sources checked 2026-09-10; recheck during implementation. Architecture, defaults, safety limits, and acceptance targets above are proposed Gloom design choices.

- **R1:** [OBS remote control](https://obsproject.com/kb/remote-control-guide)
- **R2:** [FFmpeg formats and HLS packaging](https://ffmpeg.org/ffmpeg-formats.html)
- **R3:** [Cloudflare D1 pricing](https://developers.cloudflare.com/d1/platform/pricing/)
- **R4:** [R2 presigned URLs](https://developers.cloudflare.com/r2/api/s3/presigned-urls/)
- **R5:** [R2 with the Workers Cache API](https://developers.cloudflare.com/r2/examples/cache-api/)
- **R6:** [Worker custom domains](https://developers.cloudflare.com/workers/configuration/routing/custom-domains/)
- **R7:** [Video.js](https://videojs.com/)
- **R8:** [Desktop OAuth](https://developers.google.com/identity/protocols/oauth2/native-app)
- **R9:** [YouTube resumable uploads](https://developers.google.com/youtube/v3/guides/using_resumable_upload_protocol)
- **R10:** [YouTube videos.insert](https://developers.google.com/youtube/v3/docs/videos/insert)
- **R11:** [YouTube visibility](https://support.google.com/youtube/answer/157177?hl=en)
- **R12:** [YouTube embedded player](https://developers.google.com/youtube/iframe_api_reference)
- **R13:** [YouTube locked-private uploads](https://support.google.com/youtube/answer/7300965?hl=en)
- **R14:** [Google OAuth token expiration](https://developers.google.com/identity/protocols/oauth2)
- **R15:** [R2 pricing](https://developers.cloudflare.com/r2/pricing/)
- **R16:** [Workers pricing](https://developers.cloudflare.com/workers/platform/pricing/)
