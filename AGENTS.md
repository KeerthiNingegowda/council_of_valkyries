# Council of Valkyries — Agent Instructions

This file is the canonical context document for AI coding assistants (Claude Code, GitHub Copilot, etc.) working in this repository. It describes the system's architecture, invariants, and conventions so you can make correct changes without re-deriving them from the code each time.

---

## What this project is

A multi-LLM deliberation pipeline. A user query plus a context pack (user preferences + parsed documents) are sent to K models in parallel. Their outputs are anonymized, peer-reviewed by the other council members, and then synthesized by a single queen model (Sigrún) into one final answer tuned to the user's preferences.

---

## Pipeline stages (in order)

| Stage | What happens | Key invariant |
|---|---|---|
| **Context pack** | Parse uploaded files + merge user preferences | Parse documents **once here only** — never inside the model calls |
| **K Valkyries** | K models draft in parallel | All receive identical context pack + query |
| **Anonymizer** | Strip model identities from drafts | Must run before peer review, no model name or telltale metadata may leak |
| **Peer review** | Each model ranks + critiques the K−1 drafts it did not write | Self-reviews are **excluded** — never pass a draft back to its author |
| **Sigrún (queen)** | Synthesizes the final answer | Receives: query + context pack + all K drafts + all rankings + all critiques |
| **Final answer** | Returned to the user | Preferences have been applied three times by this point |

---

## Preference injection — three times, not once

Preferences (tone, style, rubric) are not injected once at the context-pack step and forgotten. They are explicitly re-injected at three points:

1. **Generation prompt** — so drafts start in the right direction
2. **Peer-review rubric** — so critics judge against the user's standards, not generic quality
3. **Sigrún's synthesis prompt** — so conflicts between drafts are resolved in the user's favor

If you are modifying any prompt, check which stage it belongs to and confirm preferences are present.

---

## Invariants — do not break these

- **Parse once.** Document extraction (PDF/PPTX/DOCX → text) happens in the context-pack step. No model call may trigger its own parse. Violating this wastes tokens and produces inconsistent extractions.
- **Anonymize before review.** The anonymizer stage is not optional. Peer reviewers must not know which model produced which draft.
- **Exclude self-reviews.** Each reviewer sees K−1 drafts, never its own. Enforce this in the distribution logic, not by trusting the model to skip itself.
- **Sigrún sees everything.** Her prompt must include: the original query, the context pack, all K drafts (with their anonymous labels), and all rankings + critiques. Do not summarize or trim her input.
- **Parallel drafting.** The K Valkyrie calls must be concurrent. Do not serialize them.

---

## Naming conventions

- **Valkyrie** — any of the K council member models
- **Sigrún** — the queen/synthesis model (always a single model, more capable or at least differently prompted)
- **Context pack** — the assembled input: parsed document text + user preferences
- **Draft** — a Valkyrie's raw response before anonymization
- **Anonymous draft** — a draft after the anonymizer has run; labeled by position (e.g., Draft A, Draft B), never by model name
- **Critique** — a Valkyrie's free-form written evaluation of another's draft
- **Ranking** — a Valkyrie's ordered list of the K−1 drafts it reviewed

---

## What to do when modifying prompts

1. Identify which stage the prompt belongs to (generation / peer-review / synthesis).
2. Confirm user preferences are injected at that stage.
3. For peer-review prompts: confirm the rubric section references the user's preferences, not a hardcoded quality standard.
4. For Sigrún's prompt: confirm all five inputs are present (query, context pack, all drafts, all rankings, all critiques).

---

## Out of scope

- Looping / re-running the council if quality is low (not implemented; design decision deferred)
- Per-model context packs (all Valkyries get the same pack by design)
- Self-review handling via model instruction (always enforce structurally, not via prompt)
