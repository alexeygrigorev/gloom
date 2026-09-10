# Gloom: product and technical specification

**Version:** 0.1  
**Date:** 2026-09-10  
**Status:** Proposed implementation specification; not implemented or validated on the owner's machine.

## 1. Purpose and priorities

Gloom is a personal Windows recording tool, not a commercial Loom competitor. It should make recording course lessons and short explanations predictable, keep an independently recoverable local copy, and remove unnecessary publishing work.

The central workflow is **record locally, automatically upload privately to YouTube, review later, then explicitly release the video**. YouTube should host and deliver most approved recordings. Gloom should not pay to store and distribute the same video by default.

Priorities, in order:

1. Never silently lose a recording or publish an unapproved recording.
2. Make recording and recovery simple and dependable.
3. Automate private uploads while keeping publication under the owner's control.
4. Minimize implementation complexity and recurring infrastructure costs.
5. Add alternative hosting only for a demonstrated need.

### 1.1 Confirmed requirements

The owner uses Windows, records course material and informal clips, wants independently selectable system audio, values a simple recording workflow, and prefers minimal/serverless infrastructure. Rust is desirable but is not a reason to rewrite working capture or encoding software. Some recordings must remain available long-term; others may expire or be deleted. Most recordings should upload automatically to YouTube, with manual approval later.

### 1.2 Proposed defaults

These are design choices, not additional requirements attributed to the owner:

| Decision | Initial choice |
| --- | --- |
| Platform | Windows 11 x64 first; validate other Windows versions separately |
| Recording engine | Existing OBS installation controlled by a small Rust application |
| Persistent application state | Local SQLite database and a managed recordings directory |
| Destination | YouTube, uploaded directly from the desktop |
| Upload privacy | Always private for the automatic-upload workflow |
| Approval interface | YouTube Studio in the system browser |
| Intended course visibility | Suggest unlisted at review; never apply it automatically |
| System audio | Off unless explicitly enabled for that recording |
| Cloud infrastructure | None required for the MVP |
| R2 | Deferred, optional backup or alternative-hosting adapter |

## 2. The first feasibility gate: YouTube publication

An API project audit is a dependency, not an implementation detail to leave until release. New unaudited API projects can produce videos that are locked private. YouTube documents that those uploads cannot simply be changed to public/unlisted in Studio; its supported recovery is re-uploading through its official site/app or a verified API service. The service can also seek an API audit. [S1][S2]

Before relying on this architecture for student-facing videos, test a disposable recording against the actual Google project and channel. Record whether the project supports publication after a private API upload. Do not assume that OAuth consent-screen verification, a channel's longer-upload verification, and the YouTube API compliance audit are interchangeable approvals.

Gloom must offer two explicitly labeled modes:

- **Automatic private upload:** the desired workflow, with publication dependent on the validated project capability.
- **Manual YouTube upload:** retain/export the completed file, open the official upload interface, and allow the owner to attach the resulting video URL. This works as a fallback workflow but is not advertised as automatic upload.

Do not build browser automation, cookie extraction, or private/undocumented upload endpoints to evade the gate. If publication is blocked, explain it and preserve the file. Do not automatically pay for or publish to R2 instead.

## 3. Scope

### 3.1 MVP

Provide screen or window capture, microphone selection, optional system audio, start/stop controls and a hotkey, a local recording library, file import, private resumable uploads, visible upload status, retry/recovery, review in Studio, link copying after approval, and separate local/remote deletion controls.

No camera overlay is required initially. The system must support importing an existing recording so the upload workflow can be tested before the custom recorder UI exists.

### 3.2 Not in the MVP

No team accounts, billing, public signup, video editor, AI summaries, transcription pipeline, analytics warehouse, mobile/macOS/Linux capture, cloud transcoding, live broadcasting, DRM, enrollment synchronization, or custom video-streaming player.

A branded share domain, YouTube playlists, in-app publication, camera overlays, and R2-hosted instant sharing are later features. Do not provision AWS, Cloudflare, or a hosted database merely to begin development.

## 4. End-to-end experience

### 4.1 Setup

The owner selects a recordings directory, checks OBS connectivity, chooses a microphone, connects the intended YouTube channel, reviews upload metadata defaults, and explicitly enables automatic private uploading. The chosen channel name and identity must remain visible in settings and the upload queue.

Automatic upload is disabled until consent is established. Once enabled, it is the default for ordinary recordings, with a prominent per-recording **Local only** override. Private upload still sends the content to Google before human review; the UI must make that distinction clear.

### 4.2 Record and stop

Before Record, show the capture target, microphone input meter, system-audio toggle, destination policy, and free disk space. During capture, show an unmistakable recording indicator and elapsed time.

On Stop, commit the recording to the local library, finalize/remux it when needed, and enqueue a validated file automatically. No browser tab, cloud connection, metadata form, or approval prompt should be needed to save the recording.

### 4.3 Review later

The library separates local recordings, active uploads, private recordings awaiting review, published recordings, and failures requiring attention. Each item offers local preview, title/description editing where applicable, Open in Studio, retry/pause/cancel, and destination information.

In the MVP, the owner reviews the video and metadata in Studio, selects the desired visibility there, and saves. Gloom refreshes remote status and records an observed external publication as the manual approval action. It must not rewrite Studio edits with stale local metadata.

Unlisted means anyone with the link can view and forward it; it is not enrollment-based protection. The review screen must explain this. Public is an explicit alternative, never the automatic outcome of finishing an upload. [S3]

### 4.4 What ready means

| Milestone | Meaning |
| --- | --- |
| Saved locally | A durable local recording exists; finalization/recovery may still be running |
| Local preview ready | The completed or recovered file can be reviewed locally |
| Uploaded | YouTube accepted the file and Gloom has persisted its video ID |
| Processed | Provider processing succeeded; final playback quality still needs validation |
| Awaiting review | The uploaded item remains private and has not been released |
| Ready to share | Approval is recorded and the intended remote visibility/availability is verified |

**Proposed MVP trade-off:** the earlier desire for an immediately working link on Stop becomes an immediately saved local recording plus automatic publication preparation. Upload and processing are asynchronous operations performed by the running app/provider, not an instant remote-playback promise. A private video URL is not a student-ready link.

The MVP uploads a finalized file after Stop. Concurrent upload of an unfinished recording and live-to-VOD tricks are deliberately deferred. For illustration, 1.35 GB takes about 18 minutes over a 10 Mbps uplink before overhead and provider processing. This is a calculation, not a latency estimate for the owner's connection.

## 5. Recording requirements

**REC-01 — Capture controls.** Support one selected display or window, start/stop from the application and a configurable hotkey, and visible capture state. Region capture is optional after the first usable version.

**REC-02 — Audio consent.** Provide independent microphone and system-audio controls. System audio starts off for each new recording unless the owner has explicitly configured a different recording profile. Do not silently substitute a different microphone when the selected device disappears.

**REC-03 — Recoverable output.** Write to disk during capture. The initial OBS implementation should use a recoverable recording container, such as MKV, and remux to the upload format afterward when required. OBS recommends MKV to avoid losing the entire recording on an ungraceful stop. [S4]

**REC-04 — Quality.** Begin testing with 1080p, 30 fps, hardware H.264 encoding where available, and AAC audio in the upload MP4. These are proposed starting settings, not a fixed bitrate promise. Validate small code text, scrolling, cursor motion, and audio synchronization after YouTube processing. Preserve a configurable higher-quality preset.

**REC-05 — Local processing.** Reuse the existing encoder; avoid a second lossy encode when a remux is sufficient. Never stream the entire recording into application memory. A capture master and derived upload file must have distinct lifecycle records.

**REC-06 — Failures.** Low disk, device loss, capture failure, sleep, and application interruption must leave visible state and any recoverable bytes. A recovered/incomplete recording requires review before being automatically uploaded.

**REC-07 — Process lifecycle.** Closing the main window may minimize to the tray after an explicit preference is set. Exiting the uploader or sleeping/shutting down the PC pauses local upload work; it resumes on the next launch. Do not imply that a remote server keeps uploading when the PC is off.

## 6. Architecture

```text
Windows desktop
  Minimal UI / tray / hotkeys
          |
     Rust coordinator
       /     |       \
 Capture   Library   Persistent upload worker
 adapter   SQLite          |
    |         |       Google OAuth tokens in OS secret storage
   OBS        |            |
    |         |       Direct resumable upload to YouTube
 Recoverable local file    |
    |                      v
 Local remux/validation   YouTube processing
    |                      |
 Upload-ready file      Private review in Studio
                           |
                      Manual publication
                           |
                    Direct YouTube link
                    Optional future embed page
```

OBS already exposes WebSocket-based remote control; Gloom should use that rather than invent an inter-process control protocol. Keep the control connection local and authenticated. [S5]

Recommended module boundaries are `capture`, `media`, `library`, `jobs`, `youtube`, `secrets`, and `ui`. Capture and destination adapters should be replaceable, but no general plugin framework is required. An OBS dependency is acceptable for a personal tool; evaluate native Windows capture only if it demonstrably improves the everyday workflow.

SQLite is the source of truth for local workflow state. Files use stable recording IDs, not titles, as identifiers. The UI must remain responsive while recording, hashing, remuxing, or uploading. A single-instance/job-lease mechanism prevents two processes from uploading the same queued revision.

## 7. YouTube integration

### 7.1 Authentication and consent

Use a Desktop OAuth client, the system browser, a supported loopback redirect, state validation, and PKCE. Keep access/refresh tokens in Windows-protected secret storage rather than source files or the public repository. Request offline access where supported and handle revoked/expired credentials through reauthorization. An installed-app client credential is not a substitute for protecting the owner's tokens. [S6]

Start with `youtube.upload` for uploads and the read permission needed to inspect the owner's channel and private video status, such as `youtube.readonly`; validate the exact scope set in the feasibility test. Studio-based approval avoids requesting publication/delete permissions in the first version. Add broader permissions only for explicit later features, using the relevant method's current requirements. [S1][S7]

Surface the destination, title, description, and privacy choices, and let the owner disable an upload destination. Validate the integration against YouTube's required minimum functionality rather than assuming a personal application has no platform obligations. [S8]

### 7.2 Upload operation

**UP-01.** Only enqueue a completed, validated file revision whose upload policy permits YouTube. Freeze its size and content fingerprint for the job.

**UP-02.** Use the documented resumable protocol. Persist the session reference securely; query the provider for the acknowledged offset after interruptions rather than trusting the number of bytes the client attempted to send. Respect protocol chunk alignment and retry behavior. [S9]

**UP-03.** Set `status.privacyStatus=private` explicitly and `notifySubscribers=false` on the initial upload. Do not set `publishAt` or enqueue a delayed visibility change. Owner-confirmed audience and other applicable disclosure settings belong in the metadata profile. [S1]

**UP-04.** Record the video ID transactionally when the provider confirms completion. Never create a second upload just because the client lost the final response. Reconcile the resumable session first; if the outcome cannot be determined, stop in `completion_unknown` for owner-assisted resolution.

**UP-05.** Use bounded retries with exponential backoff and jitter. Distinguish network/transient failures from authentication, quota, policy, metadata, and local-file failures. Retryable jobs survive application and machine restarts.

**UP-06.** One concurrent upload is the default. Provide pause/resume, cancel, progress, and an optional bandwidth limit. Capture takes priority over remux/upload work.

**UP-07.** Poll only tracked video IDs for provider status, with backoff and a manual refresh action. The owner-authorized `videos.list` endpoint exposes processing information; do not substitute unauthenticated scraping. [S7]

**UP-08.** Cancellation stops further transfer but is not a promise that YouTube has deleted a partially or fully created asset. Show known remote state and provide a Studio link.

### 7.3 Quotas and channel capability

At the review date, YouTube documents separate default daily buckets of 100 upload calls and 100 search calls, plus 10,000 units for other endpoints; quotas reset at midnight Pacific Time. Treat the actual project's console as authoritative and keep limits configurable. Do not copy older per-upload quota estimates into the implementation. [S10]

Pause on quota exhaustion and explain when retry will be attempted. Do not create extra projects to bypass quotas. Avoid `search.list` for routine polling.

Validate that the channel can upload recordings longer than 15 minutes. YouTube documents an upper limit of 256 GB or 12 hours, whichever comes first; longer-upload eligibility is a separate setup check. [S11]

## 8. Approval and publication safety

**PUB-01.** Upload completion, successful processing, a retry, a timer, restarting the application, and reconnecting OAuth must never approve or publish a video.

**PUB-02.** Review decisions refer to a specific recording revision and remote video ID. Replacing/re-uploading media invalidates approval for the new asset.

**PUB-03.** MVP approval occurs in Studio. Store the observed visibility and the time it was refreshed. Gloom may recognize the owner's external change to unlisted/public as approval, but a successful upload alone is not evidence of review.

**PUB-04.** Only enable the normal Copy share link action after the intended visibility is observed. Provide a separately labeled private-owner/Studio link while review is pending. Warn that actual playback and course embedding should be checked before distributing a lesson.

**PUB-05.** `provider_blocked` is distinct from `pending_review`. Do not infer a locked-private reason merely because ordinary private visibility is observed. Use setup validation, documented errors, and owner confirmation where the API does not expose a definitive reason.

**PUB-06.** Keep-private and reject are non-publication outcomes. Reject does not automatically delete the original or a remote asset. Deletion requires its own action.

**PUB-07.** Future in-app publication must require an explicit confirmation showing the exact video, title, channel, and target visibility. It must not bypass the feasibility gate. Remote updates must preserve unrelated metadata and be verified after completion.

## 9. Persistent model and state

Use separate state dimensions rather than one overloaded `ready` flag.

| Entity | Essential fields |
| --- | --- |
| Recording | ID, creation time, title, description, course/tag, content revision, capture state, duration, resolution, audio configuration, destination-policy snapshot, retention class |
| Local artifact | Recording/revision, role (master/upload/thumbnail), path, size, checksum, finalization state, existence/validation time |
| Upload job | ID, recording/revision, destination/channel, state, protected session reference, acknowledged offset, attempt count, next retry time, remote ID, structured last error |
| Remote asset | Recording/revision, provider/video ID, upload status, processing status, observed visibility, embeddability when available, last verified time, setup/policy gate state |
| Review event | Recording/revision, remote ID, decision, intended visibility, time, provenance (Studio observation or explicit Gloom action) |
| Backup record | Artifact/revision, destination, checksum verification, completion time, last restore test |

Suggested capture states: `recording`, `finalizing`, `local_ready`, `interrupted`, `failed`.

Suggested upload states: `queued`, `uploading`, `paused`, `retry_wait`, `auth_required`, `quota_wait`, `completion_unknown`, `uploaded`, `failed`, `cancelled`, `manual_upload_required`.

Keep provider processing (`unknown`, `pending`, `succeeded`, `failed`) separate from review (`pending`, `approved`, `kept_private`, `rejected`) and actual visibility (`private`, `unlisted`, `public`, `unknown`). A local approval never overrides the provider's actual visibility.

State transitions must be persisted before dependent jobs run. On startup, scan unfinished jobs and incomplete local artifacts, reconcile them, and display recovery actions. An immutable file revision and one active upload job per revision/channel are local deduplication rules, not an exactly-once guarantee from the remote API.

## 10. Distribution and course integration

The cheapest MVP copies the approved YouTube URL into the course manually. Gloom must not depend on a Maven API, student database, or automated enrollment synchronization.

A later branded share page can map a stable Gloom ID to an approved provider ID. For YouTube content, use YouTube's supported embedded player, not extracted media URLs or a player that removes its controls/branding. Follow the embed requirements and include a direct Watch on YouTube fallback. [S12]

The public mapping may expose only approved publication metadata. Do not deploy a JSON catalog containing private video IDs, titles, review notes, or local paths. Updates to a stable link must not unexpectedly replace course content without owner confirmation.

Unlisted YouTube does not meet a strict enrolled-students-only requirement. Such content must use another explicitly selected destination/access model. Private does not mean end-to-end encrypted: YouTube explains that its systems and reviewers may inspect private videos. [S3]

Test playback from the actual student environments. No design can promise that YouTube is reachable on every network. R2 or another host is an opt-in alternative, not a silent fallback that redistributes content.

## 11. Retention and backup

| Classification | Default behavior |
| --- | --- |
| Permanent course material | No automatic deletion; retain an independent recoverable copy and publication mapping |
| Ordinary recording | Keep the original until the owner explicitly enables a cleanup policy |
| Temporary clip | Offer an explicit 30- or 90-day policy, with a Keep override |
| Failed/incomplete capture | Retain for recovery; cleanup only after review or a separately enabled policy |
| Derived upload file | May be removed after successful upload/validation if an independently usable master remains |

Uploading to YouTube does not satisfy Gloom's independent-backup requirement. Before enabling automatic original cleanup, require a verified second copy, such as an external drive or optional private object storage. Backup is not the same as public hosting.

Protect pending uploads, ambiguous completion states, unpublished course masters, and failed backups from automated cleanup. Show a dry-run deletion list and separate Delete local copy from Delete on YouTube. Remote deletion remains a Studio action in the MVP.

A temporary local expiration must not imply that a published YouTube video has expired. Provider deletion/access changes have separate policies and confirmations. Display broken or missing remote assets without deleting the remaining source.

Do not implement infrequent-access/archive tiers initially. Reconsider them only after measured storage costs justify the recovery complexity.

## 12. Optional R2 extension

R2 is not part of the default upload path. Never upload every recording to R2 just to copy it to YouTube.

Two distinct later uses are allowed:

**Private backup.** Store selected original files privately, optionally encrypted before upload. No CDN/player is required. Backup failure must not prevent a local recording or completed YouTube upload from existing.

**Alternative sharing.** For explicitly selected non-YouTube clips, add a player, authorization where required, local media preparation, and caching configuration. Keep backups and shareable media separate. Validate cache limits, CORS, content types, and seeking before choosing MP4 versus segmented HLS.

The earlier near-instant sharing idea belongs here: produce playable segments and upload completed segments during capture. It is a separate engineering milestone with its own readiness/approval rules. It must not be smuggled into the YouTube MVP as a required HLS pipeline.

## 13. Operating-cost model

These are architecture budgets, not provider guarantees. USD before tax; development time, electricity, connectivity, disk purchases, code signing, domains, and independent backups are excluded unless selected.

| Configuration | Estimated incremental Gloom cloud hosting |
| --- | --- |
| Desktop + direct YouTube uploads + direct YouTube links | Target $0/month: no Gloom-managed cloud component |
| Optional static YouTube embed/share site | Can fit free static hosting; domain and dynamic functions are separate |
| Optional 40.5 GB private R2 backup | Approximately $0.50/month storage with unused free allowance; operations extra beyond allowances |
| Optional 100 GB private R2 backup | Approximately $1.35/month storage on the same assumptions |
| Optional authenticated R2 video delivery | Separate future budget; not required for YouTube-hosted viewing |

The R2 examples use Standard storage at $0.015/GB-month after 10 GB-month free. R2 includes 1 million write-class and 10 million read-class operations monthly; egress has no charge, but connected metered services can add costs. Billing rounds usage units. [S13]

Cloudflare Workers Static Assets currently serves static requests without storage/request charges; dynamic Worker execution is priced separately. This is an optional share-page host, not a reason to introduce a backend. [S14]

The 40.5 GB example comes from 30 recorded hours at an assumed average 3 Mbps: `hours * Mbps * 0.45 = decimal GB`. Actual masters may be much larger. YouTube handles delivery in the default design, so student viewing does not generate Gloom CDN traffic. Quotas, account restrictions, policy changes, and access requirements still matter.

## 14. Security and operational requirements

No credentials, tokens, resumable-session URLs, recording media, private metadata, or local databases belong in the public repository. Provide sanitized configuration examples and ignore local state. Tests use synthetic media and mocked HTTP responses, not real account secrets.

Bind local control endpoints to loopback, authenticate OBS control, restrict filesystem access to configured locations, sanitize filenames, and render titles/descriptions as text rather than HTML. Do not execute shell commands constructed from metadata.

Logs should contain IDs, state transitions, error categories, and timings, but redact credentials and private URLs. A diagnostics export must be previewable. Telemetry is off by default.

Expose actionable failures: disk full, source disappeared, connection offline, reauthorization needed, quota exhausted, upload completion uncertain, processing failed, and publication blocked. Prefer recoverable state over automatic deletion or re-upload.

Keep migrations reversible where practical, back up the library database before upgrades, and export recording/provider mappings in a documented format. Review dependency and external-platform changes before release. Re-audit bundled software licensing before distributing binaries beyond the owner's machine.

## 15. Acceptance criteria

The first personal release is acceptable only when:

- A real recording succeeds with microphone only, with microphone plus system audio, and with both disabled; system audio never appears when off.
- Screen and window selection, a long coding lesson, small text, cursor motion, and audio synchronization pass review on the owner's hardware and in YouTube playback.
- Stop creates a durable local record without a network connection or publication prompt.
- Every automatic upload starts private and cannot become public/unlisted from queue completion, retries, or application restart.
- Offline recording and interrupted uploads survive application restart; uploads resume or enter an explicit recoverable state.
- A lost final upload response does not cause an automatic duplicate upload.
- Manual approval and observed remote visibility apply to the correct content revision and channel.
- A policy-locked upload is not presented as an ordinary pending-review item that a visibility toggle will fix.
- A publication/audit-blocked installation still has a functioning local recorder and clearly labeled official manual-upload fallback.
- The library distinguishes saved, uploading, processing, review-pending, and shareable recordings.
- Original cleanup cannot remove the only recoverable course copy or an in-flight upload source.
- No cloud account other than the chosen YouTube integration is required for the default workflow, and no credentials appear in logs or Git.

Targets for startup speed, stop-to-local-preview latency, and audio drift should be set from measured prototype results. Do not describe untested targets as achieved reliability.

## 16. References and change control

Primary sources checked on 2026-09-10. External prices, quotas, and platform rules are snapshots; recheck them during the feasibility milestone. Product requirements above are proposed design decisions unless explicitly identified as platform constraints.

- **S1:** [YouTube videos.insert: upload, authorization, privacy, notifications, and audit restriction](https://developers.google.com/youtube/v3/docs/videos/insert)
- **S2:** [YouTube: videos locked as private and supported recovery](https://support.google.com/youtube/answer/7300965?hl=en)
- **S3:** [YouTube: private, unlisted, and public visibility](https://support.google.com/youtube/answer/157177?hl=en)
- **S4:** [OBS: standard recording output and recoverable container recommendation](https://obsproject.com/kb/standard-recording-output-guide)
- **S5:** [OBS: remote control guide](https://obsproject.com/kb/remote-control-guide)
- **S6:** [Google: OAuth for desktop applications](https://developers.google.com/identity/protocols/oauth2/native-app)
- **S7:** [YouTube videos.list: owner-authorized status and processing inspection](https://developers.google.com/youtube/v3/docs/videos/list)
- **S8:** [YouTube API required minimum functionality](https://developers.google.com/youtube/terms/required-minimum-functionality)
- **S9:** [YouTube resumable upload protocol](https://developers.google.com/youtube/v3/guides/using_resumable_upload_protocol)
- **S10:** [YouTube current quota buckets and method costs](https://developers.google.com/youtube/v3/determine_quota_cost)
- **S11:** [YouTube: longer uploads, verification, and size/duration limits](https://support.google.com/youtube/answer/71673?hl=en)
- **S12:** [YouTube supported embedded player](https://developers.google.com/youtube/iframe_api_reference)
- **S13:** [Cloudflare R2 pricing and billing allowances](https://developers.cloudflare.com/r2/pricing/)
- **S14:** [Cloudflare Workers Static Assets billing](https://developers.cloudflare.com/workers/static-assets/billing-and-limitations/)
