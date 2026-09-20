# Local Email Summarizer

A privacy-focused Gmail attention assistant for three accounts. It will sync email through the Gmail API, analyze messages locally with MiniCPM-2B, and show action items in a native Omarchy shell panel.

The repository currently contains the implementation-ready product and engineering plan. Application code has not been started.

## Planned stack

- QML, Qt Quick, Quickshell, and Omarchy shell components for the GUI
- TypeScript and Node.js for Gmail sync and orchestration
- SQLite and Drizzle for local persistence
- MiniCPM-2B through llama.cpp for local inference
- A systemd user timer for daily runs and missed-run recovery

## Product behavior

The app will group mail into Act now, Review soon, Needs review, and Updates. Each result will include short bullets, an evidence-backed reason, a suggested action, and a link to the source message.

Read the [implementation plan](docs/implementation-plan.md) for architecture, security boundaries, delivery stages, and verification criteria.

## Status

Planning complete. The first implementation milestone is a local MiniCPM benchmark followed by a one-account Gmail OAuth and sync slice.
