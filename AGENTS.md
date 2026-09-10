# Working on Gloom

Read `docs/specification.md`, `docs/implementation-plan.md`, and
`docs/agent-handoff.md` before implementing. Owner pre-work is in
`docs/preparation.md`. The repository currently contains requirements and
configuration examples, not an implemented app.

## Product invariants

- **R2-first:** ordinary recording and sharing use Gloom's own player and
  Cloudflare R2/Worker. No Google account is required.
- **Concurrent upload:** completed playback segments must upload during normal
  capture. A finished-file-only uploader is not an equivalent implementation.
- **Recoverability:** preserve a durable local source and persistent retry state.
  A URL or spinner is not proof that a recording is ready.
- **YouTube is opt-in:** Save to YouTube affects only the selected recording or
  an explicitly enabled recording preset. A course tag does not grant consent.
  Upload privately; publication requires later manual approval.
- Export, publication, changing the Gloom link's playback backend, and deleting
  local/R2/YouTube copies are independent actions. Never infer one from another.
- Keep microphone and system audio independently controllable. System audio is
  off by default. No unexpected third-party upload or public live broadcast.

The v0.2 specification supersedes the earlier YouTube-first design in Git
history. Do not resurrect that design or treat R2 sharing as deferred scope.

## Execution and credentials

Use the provided private owner config and execution secret facility. Never
commit/log filled config, private media, bucket credentials, OAuth tokens,
application secrets, or presigned URLs. `.gitignore` is not a security boundary.
Deployment credentials must not ship in the desktop app or public frontend.

Live changes require authorization for the specific named environment in the
owner config. Do not upgrade subscriptions, change nameservers, alter unrelated
resources, delete course media, or run a real YouTube upload without its separate
consent. Keep development and production isolated. Preserve data and existing
secrets on repeated provisioning/deployment runs.

## Completion standard

Deliver working implementation, a Windows build, automated setup/deploy/tests,
and authorized live deployment rather than only another plan or scaffold.
Reuse proven media/player components. Validate the capture engine's concurrent
output capability before polishing the UI.

Continue implementation/mocked testing when optional credentials or live
hardware are unavailable, but report the exact limit. Never label a mock,
headless build, proposed command, or ungranted provider approval as live-tested.
Use the completion report in `docs/agent-handoff.md` and document meaningful
technical deviations without weakening the product invariants.
