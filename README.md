<h1 align="center">Local Email Summarizer</h1>

<p align="center">
  A daily attention list for three Gmail accounts, analyzed on your laptop.<br />
  It shows which threads need action, which to review soon, and which only changed.
</p>

<p align="center">
  <a href="#running-it">Running it</a>
  ·
  <a href="docs/implementation-plan.md">Implementation plan</a>
</p>

Local Email Summarizer checks three Gmail accounts once a day and again whenever you open its panel. Each thread that needs you gets a short change summary, the action it asks for, the reason it ranked there, and a link to the source message. It only reads mail. It never sends, deletes, archives, or marks anything read in Gmail.

The worker is a TypeScript and Node.js command line tool. A local MiniCPM-2B model, run through llama.cpp, extracts facts from each message, and Zod checks that output before anything is saved. Threads and evidence go into SQLite, and the panel is a QML plugin for the Omarchy shell bar. Gmail API calls still go over the internet, but the model and the analysis stay on the laptop.

The repository on `main` holds only the design. Nothing runs yet. The sections below describe the v1 plan in [`docs/implementation-plan.md`](docs/implementation-plan.md).

## What you get

- **Four groups, sorted by what you have to do.** Act now holds unresolved actions due within 24 hours or already overdue, plus security notices that need inspection. Review soon holds direct requests without an immediate deadline. Needs review holds failed, incomplete, or ambiguous extractions, including attachment-only mail, because v1 does not read attachments. Updates holds informational changes with no action.
- **Every item cites its evidence.** The model returns bullets, one action, and one deadline, each tied to a message ID and a quoted excerpt. The app checks the JSON, field lengths, enum values, and IDs, and rejects any excerpt that does not occur in the normalized message text. It retries malformed output once. A second failure sends the thread to Needs review.
- **Deadlines stay exact.** The app resolves dates in code, using the message timestamp and its time zone. A date without a time stays date-only. A phrase such as "Friday" stays ambiguous when the context does not settle it, and the item goes to Needs review instead of guessing.
- **Analysis stays on the laptop.** MiniCPM-2B runs through llama.cpp, with one inference job at a time. Message text and extracted facts sit in a local SQLite database. Analysis sends no mail text off the laptop. Network calls go to Google's Gmail and OAuth endpoints.
- **Read-only Gmail access.** The app requests the `gmail.readonly` scope. Done and snooze are local states on each thread, so Gmail itself never changes.
- **Sync that resumes after a crash.** Each account stores a Gmail history cursor. The cursor moves forward only after the fetched changes and analysis jobs are saved. If Gmail reports an expired cursor, the app reconciles the window and rechecks unresolved threads, so open items do not disappear.
- **Coverage on every view.** The panel shows how many messages were analyzed, how many still wait, and the last successful sync for each account. A disconnected account or unfinished analysis shows up there, so a short list does not look like a clear inbox.
- **Tokens in the keyring.** Each account signs in through the system browser with PKCE and a loopback redirect. Each refresh token gets its own Linux Secret Service entry. Tokens never go into the database.

## Running it

There is nothing to install yet. The first milestone is a local MiniCPM-2B benchmark, followed by a one-account Gmail OAuth and sync slice. Each stage has an acceptance gate in the plan.

You will need these once the stages reach them:

- A Linux laptop with Omarchy for the panel. The plan targets CUDA on an NVIDIA GPU, with CPU fallback.
- Node.js active LTS for the worker.
- A pinned llama.cpp build and the MiniCPM-2B SFT GGUF checkpoint.
- A Google Cloud project with the Gmail API enabled and a Desktop OAuth client.
- Three Gmail accounts.

Google's Testing mode for OAuth usually issues refresh tokens that expire after seven days. The plan treats the personal-use OAuth setup as an early gate for that reason.

## How it works

1. The first sync pages through the last 30 days and fetches the full MIME content of each message. Later syncs read changes with the Gmail history API. Spam, Trash, and Drafts are excluded from automatic assessment.
2. The worker decodes MIME and converts HTML to text. It does not load remote images or follow links.
3. MiniCPM-2B gets the new message, bounded earlier context, and earlier extracted facts labeled as previous analysis, not source truth. The model gets no tools.
4. The app validates the output, then applies the group rules in code. Unchanged summaries are not regenerated. Deadlines are recalculated on every run.
5. The app saves the thread state and renders the panel from structured data, not from Markdown written by the model.

A systemd user timer runs the sync daily at 08:00 Asia/Kolkata, with `Persistent=true`, so a missed run catches up when the timer next starts. A powered-off laptop cannot fetch mail.

## Documentation

- [Implementation plan](docs/implementation-plan.md): architecture, security boundaries, delivery stages, and verification criteria.

## License

MIT. See [LICENSE](LICENSE).
