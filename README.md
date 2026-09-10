# Gloom

A personal, Windows-first Loom replacement: reliable local recording, upload during capture, and inexpensive sharing through your own Cloudflare R2-backed player.

**YouTube is an optional export destination, not the primary hosting path.**

**Status:** specifications, preparation guides, and configuration templates only. The recorder, service, deployment scripts, and integrations are not implemented yet.

## Primary workflow

```text
Record screen + microphone + optional system audio
  -> Keep a recoverable local recording
  -> Upload completed video segments to private R2 while recording
  -> Stop, finish the remaining uploads, and commit the playlist
  -> Share a stable Gloom link
  -> Viewers watch through Gloom's player and Cloudflare's cache
```

Gloom must work without a Google account. Ordinary recordings stay on Gloom unless you explicitly choose another destination. Local only is available when nothing should leave your PC. System audio is off by default.

The target is a playable link soon after Stop when the connection keeps up, not a broken player behind an immediately allocated URL. Offline/slow uploads remain visible and recoverable.

## Optional YouTube workflow

```text
Select a recording -> Save to YouTube
  -> Upload directly from the PC as private
  -> Review and approve later in YouTube Studio
  -> Optionally choose YouTube playback for the existing Gloom link
  -> Separately decide whether to retain or remove the R2 copy
```

Uploading, approving publication, switching the playback backend, and deleting a copy are separate decisions. None happens implicitly because another finishes. Some course recordings may move to YouTube playback; other recordings can remain on Gloom indefinitely.

Google API publication restrictions must be validated for the optional exporter. They must never block recording, R2 upload, or Gloom sharing. The supported manual-upload fallback is described in the preparation guide.

## Start here

| Document | Purpose |
| --- | --- |
| [Owner preparation](docs/preparation.md) | Account/billing/domain setup, exact credential types, Windows access, optional Google setup, and the pre-handoff checklist |
| [Agent handoff](docs/agent-handoff.md) | Ready-to-paste implementation task, expected deliverables, and completion report |
| [Owner config template](config/owner.example.toml) | Non-secret inputs/defaults to fill in a private copy |
| [Secret names](.env.example) | Credentials needed by the trusted execution environment; no real values |
| [Product and technical specification](docs/specification.md) | R2-first architecture, recording, sharing, export, security, retention, costs, and acceptance criteria |
| [Implementation plan](docs/implementation-plan.md) | Build order, automated provisioning/deployment requirements, and tests |
| [Agent constraints](AGENTS.md) | Repository-wide product and safety invariants |

Complete the owner-only preparation, supply the private config and scoped credentials securely, then give an implementation agent the handoff task. The agent should implement/provision/build/test the rest; you should not need to hand-build Workers, a database schema, the player, or an upload pipeline.

Windows permission prompts, your Google consent when enabled, and manual approval of actual videos remain owner actions. The implementation report must distinguish implemented, deployed, live-tested, and still-blocked items.

**Current specification:** v0.2, reviewed 2026-09-10. This supersedes the earlier YouTube-first specification in repository history.
