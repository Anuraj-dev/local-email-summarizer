# Daily Gmail attention assistant

Implementation plan, 19 September 2026. Planning deliverable only. No accounts connected, software installed, or schedules enabled.

## 1. Product decision

Build a personal app that checks three Gmail accounts, extracts what changed, and shows a short list of emails requiring attention. Every item has bullets, a reason, a suggested action, and a link to its source. Run daily and whenever Raja opens the app.

Use a native QML/Quickshell Omarchy plugin, a local TypeScript/Node.js worker, SQLite, and MiniCPM-2B through llama.cpp. Open the email panel from the shell bar. Keep the model and all processing on the laptop. Gmail API requests still require internet access.

Rust is viable but optional. It will not accelerate inference already handled by llama.cpp. Choose it for a later worker replacement if reducing Node dependencies or learning Rust is a separate objective.

Confirmed requirements:

- Three accounts, all Gmail.
- Current Linux laptop is the target machine.
- Local MiniCPM-2B inference.
- Daily scheduled run plus refresh on opening.
- Formatted bullets, attention ranking, and current thread updates.
- Native GUI delivered as an enableable Omarchy shell plugin.

Proposed defaults, adjustable during implementation:

- Daily run at 08:00 Asia/Kolkata.
- Show results in the app and send a desktop notification for new urgent items. Sending a digest email is outside v1.
- Initial sync covers the last 30 days, including archived incoming mail and Sent context. Offer an explicit older-mail backfill. Clearly show that earlier mail has not been assessed.
- Exclude Spam, Trash, and Drafts from automatic assessment. Include promotions and newsletters in assessment, but collapse routine results.
- No automatic replies, deletion, archiving, or marking mail read. Done and snooze are local app states.

## 2. Daily experience

1. Click the mail widget in the Omarchy bar. It opens a native panel and requests a background refresh.
2. The saved digest appears immediately with the last successful sync time for each account.
3. A refresh starts unless one is already running. Progress reports messages fetched, analyzed, and awaiting analysis.
4. Results appear by thread, with account badges and these groups: Act now, Review soon, Needs review, and Updates.
5. Open an item to inspect the source excerpt, then open Gmail to act. Mark done or snooze locally.
6. A new relevant message can reopen a completed thread. Existing unresolved tasks remain visible even when no new mail arrives.

Example output, fictional data:

> Act now · College account · Project submission
>
> - Change: The submission deadline moved to tomorrow at 17:00 IST.
> - Action: Upload the final report before the new deadline.
> - Why shown: A direct request with a deadline inside 24 hours.
> - Source: Message received today at 09:12. Open email.

Updates use the same format with an explicit “No action requested” when supported. Generate the layout from structured data, not model-authored Markdown.

Always display coverage. “No urgent items among 85 analyzed messages; 6 still need review” is valid. “Nothing needs attention” is invalid when an account is disconnected or processing is incomplete.

## 3. Stack

| Area | Choice | Reason |
|---|---|---|
| UI | QML, Qt Quick, Quickshell, Omarchy shell components | Native bar widget and formatted email panel |
| Worker | Node.js active LTS, TypeScript CLI | Gmail integration, validation, and app logic in .ts files |
| Database | SQLite with better-sqlite3 and Drizzle migrations | Local persistence with no database service |
| Validation | Zod | Validate model output and app requests |
| Gmail | Official googleapis Node client | OAuth and Gmail API integration |
| Inference | Pinned llama.cpp build, MiniCPM-2B SFT GGUF | Local execution and constrained JSON output |
| Secrets | Linux Secret Service through a verified maintained binding | Keep refresh tokens outside the database |
| Scheduling | systemd user timer and a shared run function | Daily execution, catch-up, and CLI access |
| Testing | Vitest, CLI integration tests, QML lint and real shell checks | Logic, JSON contracts, and native GUI behavior |
| Packaging | pnpm lockfile, compiled Node CLI, Omarchy plugin manifest | Reproducible local installation |

The plugin calls the compiled CLI asynchronously using Quickshell.Io.Process with argument arrays. No web server is needed. SQLite WAL and short transactions support panel reads while the worker writes. Keep one inference job active at a time. QML files describe the native GUI; the core implementation stays in TypeScript.

Use current compatible versions at implementation time and lock them. No vector database, Redis, Kubernetes, agent framework, or fine-tuning in v1. Related context comes from Gmail thread IDs.

References: [Drizzle SQLite setup](https://orm.drizzle.team/docs/sqlite/get-started-sqlite), [llama.cpp server](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md).

## 4. Hardware and model gate

Read-only inspection of this laptop found:

- AMD Ryzen AI 7 350, 8 cores and 16 threads.
- Approximately 22 GiB usable RAM, approximately 11 GiB available during inspection.
- NVIDIA GeForce RTX 5050 Laptop GPU, 8,151 MiB VRAM, 7,682 MiB free during inspection.
- NVIDIA driver 610.57.04 responds to nvidia-smi.
- Neither llama-server nor ollama was found on PATH. No matching inference process was found by the limited process-name check. This does not prove no model files exist elsewhere.

The hardware looks suitable for a quantized small model. CUDA inference, model compatibility, memory usage, and throughput remain unverified.

Use the text model openbmb/MiniCPM-2B-sft-bf16 as the initial checkpoint candidate. The model card describes 2.4B non-embedding parameters and warns about hallucination and prompt sensitivity. SFT is the initial choice because the card associates the DPO variant with longer responses and greater hallucination risk. These are reasons to evaluate it, not evidence that it meets this product's needs. [MiniCPM model card](https://huggingface.co/openbmb/MiniCPM-2B-sft-bf16).

Before building the full UI:

1. Pin the exact checkpoint revision, tokenizer, chat template, converter, and llama.cpp revision. Verify the checkpoint's context limit from its configuration.
2. Obtain or convert a matching GGUF with provenance and a checksum. Start with Q4_K_M if supported by that conversion. Record the artifact actually tested.
3. Test plain extraction and schema-constrained JSON on 20 short synthetic emails. Confirm the correct chat template and end-of-generation behavior.
4. Benchmark CUDA on this GPU with CPU fallback. Driver detection alone is insufficient. llama.cpp documents CUDA builds, but this exact combination needs an execution test. [Build documentation](https://github.com/ggml-org/llama.cpp/blob/master/docs/build.md).
5. Measure cold startup, warm latency, tokens processed, peak RAM/VRAM, and failures on 100 realistic messages. Use one generation slot and bounded input/output lengths.
6. If GGUF compatibility fails, first test the same checkpoint with its documented Transformers path in an isolated Python environment. Keep that runtime behind the same adapter. Audit and pin any required remote model code before executing it.

Do not silently substitute another model. If MiniCPM fails the quality gate, retain source snippets and Needs review, then make an explicit choice about a different local model or narrower product claims.

## 5. Gmail access and synchronization

Create one Google Cloud project with Gmail API enabled and a Desktop OAuth client. Authenticate each of the three accounts separately through the system browser. Use authorization code flow with PKCE, state verification, and a loopback redirect. Store separate refresh tokens under separate keyring entries. Derive account identity from Gmail, not a user-entered label. [Google desktop OAuth](https://developers.google.com/identity/protocols/oauth2/native-app).

Request gmail.readonly. Reading bodies requires more than metadata access. This is a restricted scope. Resolve the applicable personal-use setup and any Workspace administrator restrictions during onboarding. A public release would need a separate review of Google's requirements. [Gmail scopes](https://developers.google.com/workspace/gmail/api/auth/scopes).

Do not mistake a successful first login for durable access. Google's external OAuth Testing mode normally issues seven-day refresh tokens when Gmail scopes are requested. Make production/personal-use configuration and reconnect behavior an early gate. [Google refresh token expiration](https://developers.google.com/identity/protocols/oauth2).

Initial sync:

- Capture a starting history cursor, page through the configured date range, and fetch full MIME content for each selected message.
- Store provider IDs as strings. Keys include account ID, even where IDs appear globally unique.
- Fetch associated thread context as needed, including sent replies. Mark incomplete context explicitly.
- Replay history from the captured cursor so mail arriving during backfill is not lost.

Later syncs:

- Read changes with history.list, exhaust pagination, and upsert changed messages idempotently.
- Handle added/deleted messages and label changes. Read state is metadata, never proof of completion.
- Persist fetched changes and analysis jobs before advancing the account cursor. A model failure must not lose synced messages.
- If the history cursor expires and Gmail returns 404, perform a bounded reconciliation plus revalidation of tracked unresolved threads. Keep old unresolved items visible until resolved or explicitly dismissed.
- Retry transient errors and rate limits with bounded exponential backoff, jitter, and Retry-After handling. Authentication failures require reconnection and must not stop healthy accounts.

Gmail documents partial history sync and the 404 recovery requirement. The bounded window and unresolved-thread reconciliation above are application policy. [Gmail synchronization](https://developers.google.com/workspace/gmail/api/guides/sync).

## 6. Analysis and priority rules

Pipeline: fetch → decode MIME → normalize text → extract evidence → validate → apply priority policy → update thread state → render digest.

Normalize HTML to text without loading remote images or following links. Preserve the source body locally for evidence inspection. Remove obvious duplicate quoted sections carefully. Decode supported charsets and preserve meaningful list structure. Never silently clip long input.

Process every new incoming message within the selected scope, not only unread mail. For long bodies, tokenize and split within the tested context budget, reserving room for instructions and output. Merge extracted facts with message IDs and evidence offsets. An unprocessed chunk means incomplete analysis, not routine mail.

For each message or changed thread, provide:

- Sender, recipients, account identity, subject, received timestamp, and timezone.
- New message text and bounded prior source context.
- Prior extracted facts, explicitly marked as previous analysis rather than source truth.
- A request for facts, proposed actions, dates, and supporting excerpts. The model gets no tools.

Suggested output contract:

```ts
type Extraction = {
  bullets: Array<{ text: string; messageId: string; evidence: string }>;
  action: { text: string; messageId: string; evidence: string } | null;
  deadline: {
    originalText: string;
    messageId: string;
    evidence: string;
  } | null;
  actionState: 'requested' | 'waiting' | 'resolved' | 'unclear' | 'none';
};
```

The application supplies IDs and validates references against the input set. Validate JSON, field lengths, enum values, and that excerpts occur in the normalized source. Evidence matching does not prove the paraphrase is correct; semantic accuracy needs human evaluation. Retry malformed output once. Persistent failures go to Needs review.

Resolve dates in application code with the source timestamp and explicit timezone. Preserve date-only precision. “Friday” with ambiguous context stays ambiguous. Do not invent a time or silently choose one of conflicting dates.

| Group | Initial policy |
|---|---|
| Act now | Evidence-backed unresolved action due within 24 hours or overdue; important security/account-access notice needing inspection |
| Review soon | Direct request, decision, or action without an immediate deadline; VIP update needing review |
| Needs review | Failed extraction, incomplete context, unsupported language, attachment-only content, ambiguous or contradictory action/deadline |
| Updates | Validated informational changes with no identified action |

Unread status and words such as “urgent” do not determine priority alone. A sender claiming to be a bank does not become trusted. Treat account warnings as messages to inspect through the official service, not as instructions to click email links.

Allow explicit per-account VIPs, muted senders, and corrected priorities. Explain each ranking in plain language. Do not rely on the model's self-reported confidence to hide mail.

Reevaluate deadlines on every run without regenerating unchanged summaries. Completion remains user-controlled in v1; a sent reply may change the suggestion to waiting but does not prove the underlying task is finished. Preserve unresolved work across daily digests. Reopen on a new action request, changed deadline, or uncertain material update.

## 7. Persistence and app interfaces

Use these tables with foreign keys and migrations:

| Table | Essential fields |
|---|---|
| accounts | id, Gmail address, display label, keyring reference, history cursor, sync time, error |
| messages | account_id + gmail_id unique, thread_id, sender/recipients, date, labels, normalized body, content hash, deletion flag |
| analyses | message/thread input hash, model/template/prompt versions, validated output, evidence, status, error |
| attention_items | account/thread identity, priority, local state, deadline precision/timezone, snoozed_until, last change hash |
| jobs | unique input/version key, state, attempts, next_attempt_at, lease owner and expiry |
| runs | trigger, timestamps, per-account coverage, counters, outcome |
| notification_events | event key unique, delivery state, delivered_at |
| preferences | schedule, VIPs, muted senders, retention, last-viewed run |

Cache extraction using content/context hash plus model, prompt, and normalization versions. A policy change can rerank cached facts without rerunning the model.

Keep normalized bodies for 30 days by default, except evidence needed by unresolved items. Keep minimal message metadata for cursor reconciliation and deduplication. Purge resolved analysis after 90 days by default. Account removal deletes its local content and keyring entry. Explain that the database contains email text; file permissions are not encryption. Use device disk encryption or add an encrypted database if required.

Local CLI contract, with schemaVersion and requestId on every JSON envelope:

- email-digest status --json returns coverage, counts, and current run status.
- email-digest digest --json returns bounded, paginated item lists, with an optional account filter.
- email-digest accounts list --json and accounts connect handle onboarding. Connect opens Chrome for OAuth.
- email-digest refresh --json starts the installed oneshot service without blocking and returns its current status. The database lease remains the final guard against concurrent runs.
- email-digest sync runs the actual foreground worker for systemd and diagnostics.
- email-digest item update --id ID --state done|open --json updates local state. Separate flags support snooze and priority feedback.
- email-digest item source --id ID --json returns bounded evidence text.
- email-digest item open --id ID opens the source through a trusted application-generated Gmail target.

Commands return either data or an error code/message. Keep stdout machine-readable and diagnostics on stderr. Validate input in the CLI. The GUI must check exit status, schemaVersion, and ok before accepting results. A foreground sync reports complete, partial, or failed with distinct exit codes.

Suggested source layout:

```text
src/core/gmail.ts       OAuth and incremental Gmail sync
src/core/normalize.ts   MIME decoding and source text
src/core/inference.ts   Local model adapter
src/core/attention.ts   Priority and lifecycle rules
src/core/run.ts         Shared orchestration
src/db/                 Schema and migrations
src/cli.ts              Headless entry point and JSON contract
plugin/manifest.json    Omarchy bar-widget registration
plugin/widget/EmailWidget.qml
plugin/widget/EmailPanel.qml
plugin/widget/EmailCard.qml
plugin/widget/EmailData.qml
tests/fixtures/         Synthetic email and history fixtures
evals/                  Label schema and scoring code
packaging/              Service and timer templates
```

Native shell integration, confirmed by inspecting the installed Pravah plugin:

- Its manifest uses schemaVersion 1, kinds containing bar-widget, and entryPoints.barWidget pointing to a QML component.
- Its widget imports QtQuick, Quickshell.Io, qs.Commons, and qs.Ui, derives from BarWidget, and exposes an IpcHandler.
- Its data component delegates work to a CLI. Use that integration pattern without copying its task-specific logic.

Create a user-owned plugin ID raja.email-digest, with a manifest and barWidget configuration following the installed schema. Install under the user's Omarchy plugin directory, then enable through the installed plugin/bar controls. Inspect their current help before writing installer commands. Preserve existing shell configuration and never edit packaged shell files. Disabling the widget hides the GUI; a separate setting controls daily background runs.

Design the panel around a compact mail icon with urgent count. Click opens a panel around 560 pixels wide, capped to the current screen's available dimensions. Provide account filters, last-check times, Refresh, and the four priority groups. Use two to four bullets per expanded card, with action and deadline first. Each card offers source, done, and snooze. Collapse routine updates initially and keep Needs review visible. Use shell theme colors, font settings, visible keyboard focus, Escape to close, and labels alongside urgency colors.

Render all email and model strings as plain text. Keep long subjects wrapping and lists virtualized. No OAuth tokens or raw mailbox database access belongs in QML. Poll the cheap status command every 60 seconds when closed and every two seconds only while an active run is visible. Read the digest when its revision changes. Closing or unloading the panel must not kill the worker. Stop plugin-owned polling processes on unload.

## 8. Scheduling and recovery

Use a systemd user calendar timer with Persistent=true. It can catch up when activated after a missed daily run. The user manager normally starts at login; a locked keyring can still block credentials. A powered-off laptop cannot fetch mail, and daily checks do not provide immediate alerts between runs. [systemd timer source documentation](https://raw.githubusercontent.com/systemd/systemd/main/man/systemd.timer.xml).

Proposed timer configuration, to be installed only during implementation:

```ini
[Unit]
Description=Daily Gmail digest

[Timer]
OnCalendar=*-*-* 08:00:00 Asia/Kolkata
Persistent=true
Unit=email-digest.service

[Install]
WantedBy=timers.target
```

The paired oneshot service uses absolute Node and compiled CLI paths, a private working directory, and a finite timeout. Retry temporary network failures with a bounded service retry policy. Record exhausted retries for the dashboard. Reconnect on the next launch when credentials require interaction.

Plugin refresh and scheduled refresh share a database lease with heartbeat and stale-lease recovery. Deduplicate extraction jobs by input/version. Repeated launches join an active run. Crash recovery resumes pending jobs instead of repeating completed inference.

Start the managed model process only when analysis is needed. Stop that exact process after an idle period. Do not stop a user-owned inference service. Continue sync and expose pending analysis when the model is unavailable.

Desktop notifications are opt-in during setup. Notify only for new urgent items, escalation into urgency, or a connection failure requiring action. Deduplicate by item and meaningful state change. Keep routine updates in the dashboard. A notification daemon is not a transactional store; a crash at delivery can still cause a rare duplicate, so document that limit.

## 9. Local security boundaries

- Bind the inference endpoint and temporary OAuth callback listener to 127.0.0.1. The plugin uses local CLI calls, with no general-purpose HTTP app API. Validate OAuth state and authenticate access to the inference endpoint.
- Keep refresh tokens in the keyring, access tokens in memory, and secrets out of browser storage and logs. If the keyring is unavailable, show a setup error rather than silently saving plaintext credentials.
- Treat emails as untrusted data. Model text cannot execute commands, change priority policy, fetch URLs, or send messages.
- Set QML textFormat to Text.PlainText for email and model text. Display plain source text instead of executable email HTML. Disable remote tracking content. Invoke CLI commands with argument arrays, never shell interpolation of email data.
- Generate Gmail navigation targets in application code from trusted account/message metadata. Verify the correct signed-in account opens in Chrome; provide an account-aware search fallback if direct links fail.
- Log IDs, timings, counts, and errors without email bodies or tokens. Keep local private evaluation mail out of Git and CI artifacts.

## 10. Build sequence and gates

Effort estimate is a planning estimate for one engineer, not a delivery commitment. Expect roughly 9–15 focused engineering days plus a seven-day real-use trial. OAuth setup and model compatibility may add time.

| Stage | Deliverable | Acceptance gate | Estimate |
|---|---|---|---|
| 1 | MiniCPM feasibility and labeled examples | Correct checkpoint runs locally; valid extraction; benchmark report | 1–2 days |
| 2 | One-account OAuth and incremental sync | Pagination, refresh, reconnect, and cursor recovery tested against Gmail | 1–2 days |
| 3 | Three-account persistence and jobs | No cross-account mixing; interrupted sync resumes without loss | 1–2 days |
| 4 | Extraction, ranking, and thread lifecycle | Quality gate passes; failures stay visible | 2–3 days |
| 5 | Native Omarchy plugin | Real shell panel works; Chrome source links open the correct account | 2–3 days |
| 6 | Timer, recovery, notifications, retention | Missed-run catch-up and crash recovery pass on this laptop | 1–3 days |
| 7 | Parallel use beside normal Gmail review | Seven days of measured misses, noise, runtime, and failures | 7 calendar days |

Build the first complete slice as one account → five real messages → local extraction → bullet digest. Expand only after this works. Do the three-account OAuth check before committing to the rest of the UI.

## 11. Verification and release criteria

Prepare a private labeled corpus of at least 150 messages across the three accounts, with at least 40 action-required examples. Include direct requests, receipts, newsletters, deadlines, security notices, long threads, sent replies, ambiguous dates, and actual mailbox languages. Split by thread and time so related messages do not leak between tuning and held-out evaluation. Tune on 100 messages and keep at least 50 untouched for the final gate. Publish counts with percentages; a small test set does not establish production reliability.

Proposed release targets, not measured results:

- At least 95% action-required recall on held-out examples. Count Needs review as surfaced, but report the review rate separately so routing everything there cannot pass unnoticed.
- At least 80% precision among Act now results and no more than 20% Needs review on representative mail.
- Zero fabricated deadlines or actions in the held-out audit. Audit semantic support, not just quoted substring matching.
- 100% source association accuracy in audited items.
- 100% invalid or incomplete outputs visibly marked for review.
- Cached native panel populated within two seconds on this laptop.
- Provisional target: process 100 ordinary new messages within five minutes on AC power after model startup. Record body/token distribution, long-message cost, and cold-start time separately. Revise only after the stage-one benchmark.

Required checks during implementation:

| Check | What it must prove |
|---|---|
| pnpm typecheck, pnpm test, pnpm build | Contracts, meaningful logic, and production build pass |
| Sync integration tests | Pages, duplicate events, expired cursor, token errors, deletion, and one failed account do not lose data |
| MIME fixtures | HTML, charset handling, long input, quoted content, and attachment-only mail remain traceable |
| Model failure fixtures | Invalid JSON, invented IDs, absent evidence, timeouts, and injected email instructions remain contained |
| pnpm eval | Frozen checkpoint/prompt passes the held-out corpus and reports misses, precision, review rate, and latency |
| CLI contract tests and QML lint | Versioned JSON, process failures, malformed responses, and QML diagnostics handled |
| Real Omarchy GUI gate | Three account badges, refresh, source viewing, snooze, done, reopening, stale coverage, theme changes, keyboard use, and scaled displays work |
| Real Gmail gate | Consent, durable token refresh, account isolation, incremental sync, and correct source navigation work |
| Real laptop gate | GPU inference, offline retry, missed timer, locked keyring, concurrent refresh, and interrupted worker recover |
| Privacy gate | Network inspection shows email analysis stays local; logs contain no tokens or bodies |

Use controlled clocks and explicit completion events in tests. Kill and restart the actual worker during a run to test persistence. A mocked test is not evidence that OAuth, CUDA, the native QML panel, desktop notifications, or scheduling works on this laptop. Enable, disable, and hot-reload the plugin without disturbing other widgets. Test a backend outage while the bar stays responsive. Record screenshots and shell diagnostics; browser automation does not validate QML.

During the seven-day trial, keep reviewing Gmail normally and record important messages the app missed or misranked. If quality misses the gate, keep the app in assisted-review mode. Do not claim that a quiet digest means the inbox is safe to ignore.

## 12. Rust decision and later scope

Recommended v1 uses TypeScript for the worker and QML for the native GUI. It concentrates effort on Gmail correctness and model quality.

A Rust worker can implement exactly the same JSON CLI contract later, leaving the plugin unchanged. Candidate libraries are Tokio, reqwest, serde, and SQLite through rusqlite or SQLx. Verify dependencies when that branch is chosen. Rust would reduce dependence on a Node runtime, but a rewrite has migration and packaging costs and does not improve the model's reasoning. Tauri and React are unnecessary for the requested Omarchy-native GUI.

After the release gates pass, consider attachment extraction, optional checks every 15–30 minutes while awake, user-approved digest email delivery, or a different local model. Each changes scope. Attachments especially need their own extraction coverage and verification.

The next implementation task is the MiniCPM benchmark plus one-account OAuth slice. That answers the two largest uncertainties before spending time on packaging or a Rust rewrite.
