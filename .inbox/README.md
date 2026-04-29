# Inbox

Drop free-form markdown notes here that you want triaged into the wiki later. **No schema required.** A vault-aware Claude session will read each note and decide where it belongs.

## When to use this folder

You're working in **another context** (a code-agent session in `../order-of-relations`, a chat without vault skills loaded, a quick capture from your terminal) and an idea surfaces that:

- is **out of scope** for what you're currently doing, but
- is worth **not losing**, and
- needs **vault conventions** (frontmatter, cross-links, impact/effort, indexes) that the upstream agent doesn't know how to apply.

Drop it here, finish what you were doing, and process the inbox later in a vault-aware session.

## Format

- Plain markdown. No required frontmatter.
- One file per idea. Filename is whatever you want — lowercase-kebab is preferred but not enforced.
- Free-form content: bullet list, paragraph, code snippet, fragment of a chat — whatever captures the thought. The vault-aware agent will reshape it.
- Do **not** edit existing files in this folder. If you have more to say about an existing inbox item, append to the same file (it's still in flight) or file a new one.

## Lifecycle

1. **Drop** — anyone (you, a code agent, anything that can write a file) creates `<slug>.md` in `.inbox/`.
2. **Triage** — in a vault-aware Claude session, say `triage my inbox` (or `process .inbox/<file>`). Claude reads each pending note, proposes a destination, and on your confirmation:
   - Files an **open question** at `wiki/questions/<slug>.md` with proper frontmatter (`impact`, `effort`, `decided_by`).
   - Or files a **drift correction** at `.raw/drift-*.md` and ingests it.
   - Or seeds a **draft ADR** under `wiki/decisions/`.
   - Or edits an existing concept / component / flow page directly.
   - Or discards it (with your approval).
3. **Resolve** — once filed, Claude moves the original from `.inbox/<slug>.md` to `.inbox/.processed/<slug>.md`. The processed archive is kept (not deleted) as a low-cost audit trail of the moment of insight.

## What this is *not*

- **Not `.raw/`.** `.raw/` is for shaped, immutable source documents — design memos, architecture overviews, drift audits — that ingest extracts entities/concepts/flows from. Inbox items are unshaped drafts. Triage decides whether any of them deserve to *become* a `.raw/` file.
- **Not a wiki page.** Browseable wiki content lives under `wiki/`. The inbox is staging only — items leave it during triage.
- **Not auto-processed.** Nothing reads the inbox automatically. You always trigger triage explicitly in a vault-aware session.

## Empty state

If `.inbox/` shows only `.gitkeep` and `README.md`, the inbox is empty. That's the steady state — items should pass through, not accumulate.
