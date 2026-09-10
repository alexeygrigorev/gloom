# Gloom

A personal, Windows-first Loom replacement: reliable recording, automatic private YouTube uploads, and manual approval before anything is shared or published.

**Status: specifications only. No recorder or upload integration is implemented yet.**

## Intended workflow

```text
Record screen + microphone + optional system audio
  -> Save a durable local recording
  -> Automatically upload directly from the PC to YouTube as private
  -> Review later in YouTube Studio
  -> Explicitly approve sharing as unlisted, or publishing as public
  -> Copy the YouTube link into the course or a message
```

Automatic upload is enabled only after the owner connects a channel and opts in. Every recording can instead be local-only. Approval controls publication, not the initial private upload.

The first version has no mandatory Gloom backend, database server, paid player, video CDN, or cloud transcoder. YouTube is the default distribution destination. Local originals remain independent of YouTube. Cloudflare R2 is an optional later destination for backups or recordings that should not be hosted on YouTube, not an obligatory staging bucket.

## Specifications

- [Product and technical specification](docs/specification.md): requirements, architecture, upload and approval states, retention, security, operating-cost assumptions, and acceptance criteria.
- [Implementation plan](docs/implementation-plan.md): feasibility gates, build order, and release tests.

## Important feasibility gate

An API upload from a new, unaudited YouTube API project can be **locked private**, not merely awaiting a visibility change. Do not assume changing visibility manually in Studio bypasses that restriction. Validate the audit/publication path before depending on automatic uploads for student-facing materials. The supported fallback is manual upload through YouTube's official site/app, followed by linking the resulting video in Gloom. See [YouTube's explanation](https://support.google.com/youtube/answer/7300965?hl=en).

## Definition of ready

On Stop, the recording should be safely saved and available for local review after finalization. A YouTube link is not student-ready until uploading, processing, and manual approval are complete. Immediate remote sharing is a separate, optional future R2 workflow; it is not promised by the YouTube-first MVP.

Specifications and external constraints last reviewed: **2026-09-10**.
