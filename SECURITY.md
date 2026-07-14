# Security Policy

Kanban Pro is a **local-first** desktop application by Good Guy Apps. Your boards live as plain
Markdown files on your own disk — there is no Kanban Pro server, account, or cloud database. This
document explains how we think about security, what we protect against, how to report a
vulnerability, and how to operate the app safely — especially its embedded terminal and its
readiness for AI coding agents.

We are a small independent team. We keep our processes lightweight, but our disclosures are
deliberate and honest: **this policy describes only controls the product actually implements.** We
would rather under-promise and be trusted than over-claim and be wrong.

---

## Security philosophy

Kanban Pro is built to give you **sovereignty over your data and your execution environment**:

- **Local-first and offline by default.** All application logic runs on your machine. There are no
  accounts, no telemetry, and no project data sent to us or any third party.
- **The embedded terminal and AI-agent readiness are power tools.** Kanban Pro can host a real
  system terminal inside a task, and it generates context files (`CLAUDE.md`, `AGENTS.md`,
  `MAPPING.md`) that external AI coding CLIs can read. These are deliberate capabilities for
  advanced users.
- **Shared responsibility.** The application provides the technical guardrails described below. You
  remain the final authority over which folders you open and which commands you (or an agent you
  launch) choose to run. A tool that can run your shell can, by definition, run commands you tell
  it to.

---

## How Kanban Pro is built (what we can state as fact)

These are architectural properties enforced in the shipping product:

- **No servers, accounts, or cloud.** Boards are Markdown + YAML on disk; board topology lives in
  `.kanban/board.json`; history is a local append-only `.kanban/audit.ndjson`. Optional
  collaboration is achieved by *you* syncing a folder with your own file host (iCloud/Dropbox/
  OneDrive/Git) — Kanban Pro is not in that loop.
- **No telemetry and no phone-home.** No analytics, crash-reporting, ads, or usage tracking. The
  only outbound network activity in a production build is the update check against our public
  GitHub Releases feed and links you explicitly click. Renderer network access is structurally
  constrained by a strict Content Security Policy (`connect-src 'self'`).
- **Hardened Electron shell.** The UI runs with `contextIsolation: true`, `nodeIntegration: false`,
  and `sandbox: true`, behind a strict CSP. The renderer has **no direct filesystem or process
  access**; every privileged operation is brokered through an explicit, bounded `contextBridge`
  preload API over Electron IPC.
- **Main-process-owned terminal.** The pseudo-terminal (PTY) is spawned and owned by the trusted
  main process, not the sandboxed renderer. Installed AI CLIs are detected by launching a literal
  binary with an argument array (no shell interpolation).
- **We never call an LLM, and we never store your API keys.** Kanban Pro bundles no LLM/AI API
  client. Any AI agent is a separate program *you* run inside the terminal; its credentials live in
  your OS environment and belong to that tool, not to Kanban Pro. As a defense-in-depth measure,
  secret-looking environment variables (names matching key/token/secret/password/credential
  patterns) are redacted before a terminal session's resume state is written to disk.
- **Signed, notarized, and auto-updated safely.** macOS builds are signed and notarized with an
  Apple Developer ID and run under the Apple App Sandbox. Updates are delivered by the platform
  updater and verified through the OS code-signing chain. *(The Windows installer is currently
  unsigned pending an EV certificate; SmartScreen may warn on first launch — see the README.)*

---

## Threat model and trust boundaries

Being explicit about our boundaries lets us respond quickly and lets you reason about your own risk.

**Trusted:**

- **You, the local user, and your operating system.** If an attacker already has code execution or
  physical access to your machine, they can read your files and run commands directly; defending
  that boundary is the OS's job, not ours. Issues that require pre-existing local access are out of
  scope.

**Untrusted data:**

- **Project folders you did not create** — synced, shared, cloned, or downloaded boards, and
  everything inside them (tickets, `CLAUDE.md`, `.kanban/board.json`, attachments). Treat a board
  from someone else like any other untrusted document.

**By design (not defects):**

- **The terminal and any AI CLI you launch run with your full user privileges.** That is the point
  of a terminal. Kanban Pro does not sandbox, allowlist, or otherwise restrict the commands you
  choose to run.
- **Opening or syncing a folder does not auto-execute shell commands.** Kanban Pro reads and
  renders files and may generate agent-context files, but it does not run programs from a folder on
  your behalf. Execution happens only when you launch the terminal or an agent and a command is run.
- **Local files are human-readable by intention.** Plaintext Markdown/JSON/YAML in your project and
  config directories is a feature (portability, no lock-in), protected by normal OS file
  permissions. Their existence is not a vulnerability.
- **Network posture.** Offline by default; no listening localhost HTTP/WebSocket server ships in
  production, and there is no network-exposed IPC surface.

> **Honest note on controls we do _not_ yet have.** Kanban Pro does **not** implement a VS Code-style
> "Workspace Trust" gate, an in-app agent approval/permission prompt, or OS-keychain credential
> storage. We would rather tell you that plainly than imply protections that aren't there. The
> "Secure usage" section below explains how to cover these gaps on your side today.

---

## Vulnerabilities: in scope vs. out of scope

**In scope** (please report):

- Escaping the renderer sandbox / `contextIsolation` to reach Node.js, the PTY, or the filesystem
  **outside** the bounded preload IPC API.
- Bypassing the Content Security Policy in a production build in a way that enables data
  exfiltration or remote code loading.
- Path traversal, arbitrary file read/write, or command injection through an IPC handler.
- Any code path that causes commands to execute, or files to be modified, **without** the user
  launching the terminal/agent or invoking the corresponding action.
- Defeating update authenticity (installing an update whose signature does not validate).

**Out of scope** (not vulnerabilities in this product):

- A user manually typing, pasting, or authorizing destructive commands in the embedded terminal —
  that is the terminal working as intended.
- Social-engineering scenarios that require the user to ignore explicit warnings or knowingly run
  malicious commands / open a malicious board.
- Behaviour, output, or "prompt injection" of an **external** AI CLI that does not result in
  unauthorized execution by Kanban Pro itself — the third-party agent's guardrails and your chosen
  approval mode govern that.
- The presence of plaintext local configuration/board files, or secrets you chose to place in them.
- Any issue that requires prior local code execution or physical access to the machine.
- The absence of Workspace Trust, agent approval prompts, or keychain storage. These are design
  decisions documented above, not defects; feedback and feature requests are welcome through normal
  channels.

---

## Supported versions

| Version | Supported |
| ------- | --------- |
| Latest early-access release | ✅ Yes |
| Older early-access releases | ❌ No — please update |

Only the latest early-access release (on this repository's Releases page) receives security fixes.
Older builds are not patched; please update to the latest release.

---

## Reporting a vulnerability

Please report security issues **privately**. Do **not** open a public GitHub issue, pull request, or
discussion for a suspected vulnerability.

- **Preferred — GitHub Private Vulnerability Reporting:** open this repository's **Security** tab and
  choose **Report a vulnerability**. This gives us a private, encrypted channel to collaborate on a
  fix before any public advisory.
- **Email fallback:** **contact.goodguyapps@gmail.com**

Please include:

- A clear description of the issue and its impact.
- Steps to reproduce, or a proof of concept — ideally a minimal board/workflow file that triggers it.
- **How it bypasses the threat model above** (this helps us triage fast).
- Affected version(s) and platform (macOS / Windows), if known.

**Our commitment:** we aim to **acknowledge your report within 5 business days** and to share an
initial assessment or remediation plan when we have one. We appreciate coordinated disclosure and
will work with you on timing before any public discussion of the issue. As a small team we cannot
currently offer a paid bug-bounty, but we will credit reporters who wish to be acknowledged.

---

## Secure usage — recommended practices

You are the final guardrail. A few habits keep the powerful features safe:

- **Audit unfamiliar boards before you act on them.** Before launching the terminal or an AI agent
  against a board you received or synced from someone else, review its instruction and config files —
  especially `CLAUDE.md`, `AGENTS.md`, and `.kanban/board.json` — and skim ticket contents for
  anything that reads like an instruction to run commands.
- **Keep external AI agents in their default (human-approval) modes.** Most coding CLIs ask before
  running a command or editing files. Leave that on. Reserve any "auto-approve / bypass / YOLO" mode
  for disposable, isolated environments you don't mind losing.
- **Never commit raw API keys into tickets or Markdown.** Provide credentials to your agent through
  OS environment variables instead, so they don't live in portable, syncable files.
- **Keep Kanban Pro updated** so you receive security fixes, and only install builds from this
  repository's Releases page or the Mac App Store.

---

Thank you for helping keep Kanban Pro users safe.
