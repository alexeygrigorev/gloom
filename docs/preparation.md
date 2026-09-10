# Owner preparation before handing Gloom to an implementation agent

**Updated:** 2026-09-10  
**Purpose:** Complete the account/access decisions once, then let the agent implement, provision, build, and test. This is not a requirement to build the infrastructure yourself.

**Primary product:** Windows recording -> R2 -> Gloom player/share link. **Optional extension:** Save to YouTube -> private upload -> manual approval. Google setup is never required to begin or finish the primary product.

No scripts mentioned as future deliverables in this guide exist yet. Current files are specifications and example configuration only.

## 1. Minimum handoff package

For an agent that can implement AND deploy, prepare these five things:

| Item | You provide | Agent handles |
| --- | --- | --- |
| Repository and execution environment | Access to this repository; trusted execution with a Windows test route | Code, dependencies, builds, tests, commits/PR |
| Cloudflare account and budget | R2 enabled, chosen Workers plan, approved spending boundary | Worker, D1, bindings, migrations, usage reporting |
| Production hostname | An unused subdomain in an active Cloudflare zone | Worker domain attachment, TLS verification, playback/cache tests |
| Private buckets and scoped credentials | Two empty R2 buckets and their limited S3 credentials; a scoped deployment token | Upload protocol, secrets installation, lifecycle/CORS configuration as needed |
| Owner settings | Filled private copy of the example config; local storage and retention preferences | Validation, sensible UI defaults, setup/doctor commands |

For source-code-only implementation, cloud credentials can be deferred. That produces a buildable/testable project, not an already deployed service. Do not describe a credential-free mock deployment as complete.

Recommended execution mode: a trusted implementation agent working in a checkout on your Windows PC. A remote coding agent is also possible, but Windows compilation and an actual interactive capture/audio test must be explicitly arranged; a Linux container or a headless CI build cannot prove that your microphone and desktop were captured correctly.

## 2. Repository access and safe execution

- [ ] Give the agent read/write access to `alexeygrigorev/gloom`, with permission to create an implementation branch and pull request.
- [ ] Allow dependency downloads and outbound access needed for Cloudflare APIs, package registries, and approved test hosts. Google access is needed only for the optional integration.
- [ ] Decide where Windows builds and interactive tests will run. Record this in the owner configuration.
- [ ] Permit GitHub Actions to run build/test workflows when you plan to use CI. No deployment secrets should be made available to untrusted forks or arbitrary pull-request code.
- [ ] Use the agent environment's secret facility or a protected local secret file. Do not paste secrets into an issue, pull request, repository file, chat prompt, terminal transcript, or screenshot.

A GitHub connection is not automatically Cloudflare access. GitHub Actions secrets are available to authorized workflows, not generally readable from an agent's interactive terminal. Ensure the chosen deployment process actually receives the credentials, rather than merely storing them somewhere with the same name. Prefer deployment from the trusted local agent session for the first release; CI deployment can be added later.

The agent should verify secret presence/permission with redacted checks, never by printing values. Use synthetic recordings and a dedicated development environment until you approve real-media tests.

## 3. Cloudflare account and billing: owner action

### 3.1 Enable the required products

1. Create or use a Cloudflare account you control. Enable account security and keep recovery access yourself.
2. In the dashboard, open **Storage & databases -> R2 -> Overview** and complete the R2 subscription/checkout flow. There is included free usage, but account activation is still required. [P1]
3. Use **Workers Free for development**, or enable **Workers Paid** for the intended production setup. The current paid baseline is $5/month, with request/CPU usage allowances; this is not the same as buying a Pro plan for your website's zone. [P2]
4. Keep the domain's zone plan at Free unless an actual requirement justifies otherwise. Do not subscribe to Cloudflare Stream, Images, a paid player, or a cloud encoder for this project.
5. Choose an initial budget and record it. Suggested planning value: $10/month before tax for a small installation, excluding the domain and independent backups. This is a target, not an enforced provider cap.

The agent must not accept paid subscriptions or increase the approved scope on your behalf. Usage alerts, CPU limits, and application upload limits do not guarantee a hard maximum bill. You remain responsible for the account's metered usage. The implementation should show estimated stored bytes/requests and document where to inspect actual usage. [P2]

### 3.2 Record these non-secret identifiers privately

- Cloudflare account ID.
- Zone/domain name and zone ID.
- Approved production hostname, for example `gloom.example.com`.
- Optional development hostname, for example `gloom-dev.example.com`.
- Actual Workers plan, budget target, and whether live deployment is authorized.

These identifiers are not passwords, but there is no reason to put your account inventory in a public example file. The agent should read your private config rather than invent values.

## 4. Domain preparation: owner action

Use an unused subdomain of a domain already active in your Cloudflare account. You do not need a new domain when you already control a suitable one.

A Worker custom domain requires an active Cloudflare zone and a Worker. The agent can attach the Worker later; Cloudflare handles the associated DNS record and certificate. Reserve an unused hostname and do not pre-create a conflicting CNAME. [P3]

When your domain is not on Cloudflare DNS, activating a normal Cloudflare zone may require a nameserver change at the registrar. Handle that yourself, preserve existing website/email DNS records, and verify the zone is active. Do not delegate a broad DNS migration or changes to unrelated records as part of implementing Gloom.

**Important distinction:** the production hostname belongs to the Worker, not directly to the private R2 bucket. The Worker authorizes playback before using its media cache. Keep R2's public URL disabled.

Without a ready domain, the agent can develop on `workers.dev`. However, the documented Worker Cache API path used here requires a custom domain/route, so domain readiness is a prerequisite for validating production CDN behavior, not for writing code. [P4]

A development hostname is useful for a fully realistic cache test before production. Leaving it blank permits functional development without that guarantee; the final custom-domain test must then run on the approved production hostname with a disposable recording.

## 5. Create private buckets and limited R2 credentials: owner action

This small manual step avoids granting the agent permission to create arbitrary account tokens. Everything else about application provisioning belongs to the agent.

### 5.1 Buckets

Create two empty buckets through R2's dashboard:

| Purpose | Suggested name |
| --- | --- |
| Development and disposable tests | `gloom-media-dev` |
| Real recordings | `gloom-media-prod` |

Choose **Standard** storage. Keep public access, `r2.dev`, and direct bucket custom domains disabled. Do not manually add deletion policies or upload course originals during preparation.

For origin storage in the EU, select **Specify jurisdiction -> European Union** when creating the buckets. This is a proposed default because the existing preference was EU hosting, not a claim of a compliance requirement. Jurisdiction cannot be changed on an existing bucket. A Western Europe location hint is not the same as an EU jurisdiction restriction. Worldwide cache delivery and the rest of the application need separate consideration; an EU bucket does not prove that all processing/caches/metadata stay in the EU. [P5]

Record the actual jurisdiction. For EU buckets, the S3 endpoint uses the account's `.eu.r2.cloudflarestorage.com` hostname; the S3 region is `auto`. The agent must set the Worker binding jurisdiction consistently. Do not copy an AWS `eu-west-1` region into an R2 client. [P5][P6]

### 5.2 S3 credentials

From R2's API-token management, create **Object Read & Write** credentials restricted to the development bucket, then a separate set restricted to the production bucket. Keep the **Access Key ID** and **Secret Access Key** securely; the secret is shown once. These R2 credentials differ from the general Cloudflare API token used for deployment. [P6]

Provide them to the trusted deployment environment as:

```text
R2_DEV_ACCESS_KEY_ID
R2_DEV_SECRET_ACCESS_KEY
R2_PROD_ACCESS_KEY_ID
R2_PROD_SECRET_ACCESS_KEY
```

The agent maps the relevant pair to `R2_ACCESS_KEY_ID` and `R2_SECRET_ACCESS_KEY` Worker secrets in each environment. They are used server-side to issue narrowly scoped upload authorization. Do not embed these bucket credentials in the desktop app, website, checked-in Wrangler configuration, or downloadable installer.

Production credentials can be withheld until you authorize production deployment. Their absence must block only that deployment, not implementation or development tests. Bucket-scoped credentials still have meaningful access within that bucket; use only Gloom buckets, and rotate/revoke compromised credentials.

## 6. Cloudflare deployment credentials: owner action

Create a purpose-specific API token from **My Profile -> API Tokens** (or the supported account-token equivalent). Name it for the Gloom implementation, select only the intended account/zone, and set an expiry appropriate to your handoff. Do not use a Global API key. [P7]

### 6.1 Permissions to prepare

The following is the intended permission set for the provisioning/deployment script. Dashboard wording can use Edit or Write; the agent must verify the actual endpoints against the current permission reference rather than requesting blanket access. [P8]

| Resource | Permission | Purpose |
| --- | --- | --- |
| Chosen account | Workers Scripts: Edit | Deploy Worker code, bindings, configuration and secrets |
| Chosen account | D1: Edit | Create the small databases and apply migrations |
| Chosen account | Workers R2 Storage: Edit | Inspect/configure the Gloom buckets and their required settings |
| Chosen zone | Zone: Read | Resolve/verify the approved zone |
| Chosen zone | Workers Routes: Edit | Attach/manage the approved Worker hostnames/routes |
| Chosen account, optional | Workers Tail: Read | Read sanitized live diagnostics when needed |
| Chosen zone, only if needed | DNS: Edit | Only for explicitly required Gloom DNS operations; not needed merely to delegate all DNS administration |

Do not add account billing, membership administration, API Tokens Edit, registrar, or unrelated-product permissions. Resource scopes for some operations are account-wide rather than per Worker/database/bucket: this token may technically reach other resources. Use a dedicated account when stronger isolation is important, keep the agent trusted, restrict its written task to Gloom resources, and revoke the implementation token afterward. A task instruction is not an IAM boundary.

Store the deployment token as `CLOUDFLARE_API_TOKEN`; provide the account ID as `CLOUDFLARE_ACCOUNT_ID` or in the private owner config. Do not confuse either with a runtime Gloom owner credential.

The agent must use read-only preflight checks first, then print a redacted resource plan. Live changes are permitted only when `live_deployment_authorized = true` in your handoff configuration. It must not repair a permission error by switching to a more privileged credential or deleting/recreating an unrelated resource.

### 6.2 Secrets the agent generates, not you

The bootstrap process should securely generate separate development/production values for:

- A random **Gloom owner application token**, placed in Windows-protected storage; only its verifier is installed as `OWNER_TOKEN_SHA256` in the Worker.
- A **PLAYBACK_SIGNING_KEY**, installed as a Worker secret for expiring playback authorization.
- Necessary local recovery/configuration material, saved outside Git with an explicit recovery/export procedure.

The deploy script must preserve existing secrets on rerun, support deliberate rotation, and never print them into build logs. Worker secrets are installed using supported secret-management commands/API, not plaintext `vars`. [P9]

No SMTP account, Google login, Cloudflare Access subscription, or multi-user identity provider is required for the initial single-owner administration design.

## 7. Windows preparation

### 7.1 Required for a real recording test

- [ ] Provide a Windows machine and an interactive session with the intended monitor/window and microphone available.
- [ ] Choose a local recordings directory and an independent backup destination for permanent lessons. An external drive is acceptable; do not buy cloud backup just to start coding.
- [ ] Record available disk space, Windows version, CPU/GPU, and the microphone/output devices. The agent can gather diagnostics with permission; exact hardware tuning is its job.
- [ ] Permit microphone/screen access when Windows or the capture tool prompts. These permissions may not be grantable until the app exists.
- [ ] Decide whether the app may keep running in the tray after the window closes. It must not silently install an always-on service.

Do not record confidential material for first-run tests. A short screen recording with code-like small text, scrolling, cursor movement, microphone speech, and a separate system-audio signal is sufficient. Synthetic media can cover most automated tests.

### 7.2 For implementation/building on your PC

Install, or authorize installation of, Git, Rust with the MSVC toolchain, a current supported Node.js LTS version, Microsoft C++ Build Tools with **Desktop development with C++**, and the WebView2 runtime used by Tauri. The agent should pin the versions it actually validates in the repository. [P10]

OBS can be installed as the first reusable capture-engine candidate. Enable its authenticated local WebSocket interface only when the agent needs it. Do not hand-configure a complex scene, encoder chain, or concurrent-upload workflow: proving and automating those is implementation work. A dedicated Gloom profile should not overwrite existing OBS work. [P11]

The agent should supply or document a verified FFmpeg/ffprobe build and licensing where needed. You do not need to invent recording commands. A custom pipeline must be tested to emit usable segments during capture; merely uploading after Stop is not an acceptable substitute.

For a remote implementation agent, it can arrange a Windows CI build instead of installing developer tools on your daily PC. Still plan to install the resulting application and run the real capture/audio test. Do not grant a public workflow or untrusted contributor access to a self-hosted runner on your personal machine.

A code-signing certificate is not mandatory for a personal first build. The agent must accurately label an unsigned build and provide checksums; it must not instruct you to disable Windows security globally. Packaging/install permissions may require your interaction.

## 8. Optional YouTube preparation — skip this to ship Gloom first

Enable this section only when you want the optional exporter connected in the first handoff. The agent still implements the disabled/unconfigured UI and mocked tests without it.

### 8.1 Account and client setup

1. Confirm the intended Google account and YouTube channel. Keep a record of the channel identity so the app can show it before export.
2. For course recordings, verify that the channel can accept videos longer than 15 minutes. This channel capability is separate from application verification. [P12]
3. Create/select a Google Cloud project for Gloom and enable **YouTube Data API v3**.
4. Configure the Google Auth Platform/OAuth consent settings with your application name and support/developer contact. For an External app in Testing, add your Google account as a test user.
5. Create an OAuth client of type **Desktop app**, not a web application API key or ordinary service account. Download its client configuration to a protected location and provide only its file path through `YOUTUBE_OAUTH_CLIENT_FILE`.
6. Record the project/client/channel identifiers and setup status in the private owner configuration. Do not obtain and paste a refresh token manually; the built app should run consent and store tokens in Windows-protected storage. [P13]

The desktop should request the minimal upload/read scopes appropriate to the implementation. Studio-based approval avoids requiring automated public publication or deletion permissions initially. You retain the Google password, MFA, and account recovery details; they are not implementation credentials.

### 8.2 Separate the three verification concerns

| Concern | What to know |
| --- | --- |
| OAuth Testing/production and consent verification | External Testing projects issue short-lived refresh tokens for these scopes (normally seven days); the agent must handle reauthorization and explain the selected state. Moving to production is not the same as passing an API audit. [P14] |
| Channel longer-upload eligibility | Needed for long lessons, independent of OAuth and API audit. [P12] |
| YouTube API audit/publication restriction | New unaudited API-project uploads can be locked private. Later manual approval in Studio cannot bypass that restriction. Validate actual capability before relying on API-exported course playback. [P15][P16] |

You may not be able to finish an audit before there is a working application/demo. That must not delay Gloom's R2 path. The owner handles account submissions/identity checks; the agent supplies technical details and a truthful setup report. No audit outcome or timetable is guaranteed.

Supported fallback: use Gloom's Export file/Open YouTube upload option, upload through YouTube's official site/app, then attach the resulting video link. This is a manual-upload fallback, not automatic export. No browser automation, cookie extraction, or undocumented upload bypasses.

### 8.3 What still happens after implementation

You connect the actual channel through the app's browser consent flow. A disposable private-upload test runs only with your permission. You review and approve actual videos later; that is intentional product behavior, not unfinished implementation. Neither a successful private upload nor a Google login is permission to publish.

Set `enabled_for_live_setup = false` when skipping Google setup. Set `allow_live_test_upload = true` only when you explicitly permit a non-sensitive private test upload. R2 sharing must work in either case.

## 9. Fill the handoff configuration

Copy [config/owner.example.toml](../config/owner.example.toml) to a private file such as `config/owner.local.toml`, or keep it outside the repository. Fill account/zone/hostnames, bucket names/jurisdiction, local paths, approved environments, and YouTube setup choice. Refer to [.env.example](../.env.example) for **secret names**, not a place to commit real values.

The supplied `.gitignore` is a guardrail, not encryption. Check `git status` before every commit, use the secret facility of your trusted execution environment, and do not upload a filled local file as a public issue attachment.

Do not pre-generate Worker IDs, D1 IDs, application tokens, schema migrations, or deployment files. The agent must discover/create the authorized resources and write a local deployment inventory. Unknown IDs should remain empty, never fake placeholders passed to a live API.

### Owner-only actions versus agent work

| Owner does | Agent does |
| --- | --- |
| Accept product terms/billing and choose spending scope | Estimate usage and enforce stated application limits |
| Own/activate domain and approve hostnames | Attach Worker domains and validate TLS/cache behavior |
| Supply scoped account/bucket credentials | Install secrets, bindings, D1 schema and deploy repeatably |
| Provide Windows access and approve permission prompts | Implement capture, packaging, uploader, player, recovery and tests |
| Optionally create Google client and complete account consent/audit steps | Implement optional private export, review state and manual fallback |
| Approve actual video publication | Never automatically publish on upload completion |
| Choose deletion/backup policy | Implement protected cleanup and restore/export procedures |

## 10. Final preparation checklist

- [ ] Repository access and the actual Windows execution/test route are available.
- [ ] Cloudflare account has R2 enabled and the intended Workers plan is understood.
- [ ] An unused production hostname is reserved in an active zone, or production is explicitly deferred.
- [ ] Development/production private buckets and jurisdiction are recorded.
- [ ] Required bucket/deployment credentials are available to the correct trusted execution phase, not merely stored in inaccessible CI secrets.
- [ ] Private owner configuration is filled; live deployment authorization and budget are explicit.
- [ ] Local recording and independent-backup paths are chosen; no automatic deletion is enabled by accident.
- [ ] YouTube is either deliberately skipped or its client/configuration and live-test consent are supplied.
- [ ] The implementation task in [agent-handoff.md](agent-handoff.md) is ready to paste.

**Honest ready-to-use boundary:** the agent can deliver the application and deployed service once it has the necessary execution access. It cannot pre-approve Windows prompts, sign into your personal channel without your consent, guarantee Google's audit decision, or perform your later editorial review. Its completion report must distinguish implemented, deployed, live-tested, and owner-action-required items.

## Sources

Provider setup details checked 2026-09-10. Recheck labels/permissions during implementation; Gloom-specific names and the owner/agent work split are design decisions.

- **P1:** [R2 activation](https://developers.cloudflare.com/r2/get-started/)
- **P2:** [Workers plans and usage pricing](https://developers.cloudflare.com/workers/platform/pricing/)
- **P3:** [Worker custom-domain prerequisites](https://developers.cloudflare.com/workers/configuration/routing/custom-domains/)
- **P4:** [R2 Cache API deployment requirements](https://developers.cloudflare.com/r2/examples/cache-api/)
- **P5:** [R2 location and jurisdiction](https://developers.cloudflare.com/r2/reference/data-location/)
- **P6:** [R2 token permissions and S3 credentials](https://developers.cloudflare.com/r2/api/tokens/)
- **P7:** [Creating a scoped Cloudflare API token](https://developers.cloudflare.com/fundamentals/api/get-started/create-token/)
- **P8:** [Cloudflare permission reference](https://developers.cloudflare.com/fundamentals/api/reference/permissions/)
- **P9:** [Worker secrets](https://developers.cloudflare.com/workers/configuration/secrets/)
- **P10:** [Windows/Tauri build prerequisites](https://v2.tauri.app/start/prerequisites/)
- **P11:** [OBS remote control](https://obsproject.com/kb/remote-control-guide)
- **P12:** [YouTube longer-upload eligibility](https://support.google.com/youtube/answer/71673?hl=en)
- **P13:** [Google desktop OAuth setup](https://developers.google.com/identity/protocols/oauth2/native-app)
- **P14:** [Google OAuth refresh-token expiration](https://developers.google.com/identity/protocols/oauth2)
- **P15:** [YouTube upload restrictions](https://developers.google.com/youtube/v3/docs/videos/insert)
- **P16:** [Locked-private recovery](https://support.google.com/youtube/answer/7300965?hl=en)
