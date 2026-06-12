# Council of Valkyries

A multi-LLM deliberation pipeline where K models draft independently, critique each other's anonymized outputs, and a queen model synthesizes the final answer — tuned to your preferences at every stage.

> Inspired by [Karpathy's llm-council](https://github.com/karpathy/llm-council), with enhancements: preference injection at three stages, free-form critiques alongside rankings, and self-vote exclusion.

---

## How it works

### 1. Context pack (assembled once)
User preferences (tone, style, rubric) and any uploaded documents (PDF, PPTX, DOCX) are combined into a single context pack. Documents are parsed **once** here — not once per model — to avoid token waste and inconsistent extractions.

### 2. K Valkyries — parallel drafting
K models each receive the same `[context pack + query + preferences]` and produce independent drafts in parallel. Preferences are injected here so drafts start in the right direction.

### 3. Anonymizer
Model identities are stripped from all K drafts before they reach peer review. This prevents reviewers from recognizing their own output by style.

### 4. Peer review — rank + critique
Each model reviews the **K−1 drafts that aren't its own** (self-votes are excluded). For each draft it produces:
- A **ranking** — a clean signal for which draft wins
- A **free-form critique** — the reasons, which is what the queen actually needs to merge well

Preferences are injected here as the rubric, so critics judge against the user's standards rather than generic quality.

### 5. Sigrún — queen synthesis
Sigrún receives everything: the original query, the context pack, all K drafts, and all rankings + critiques from the council. Preferences are injected a third time so that when drafts conflict, she resolves in the user's favor. She produces the single final answer.

---

## Preference injection — three times, not once

| Stage | Purpose |
|---|---|
| Generation prompt | Drafts start in the right direction |
| Peer-review rubric | Critics judge against your standards, not generic quality |
| Synthesis prompt | Conflicts resolved in your favor |

---

## Pipeline diagram

```
User preferences          Uploaded files
(tone, style, rubric)     (PDF, PPTX, DOCX)
        │                        │
        └──────────┬─────────────┘
                   ▼
            Context pack
         (prefs + parsed docs)
                   │
                   ▼
            K Valkyries
       (independent drafts, parallel)
                   │
                   ▼
            Anonymizer
          (strips identities)
                   │
                   ▼
            Peer review
      (each ranks + critiques K−1 drafts)
                   │
                   ▼
          Sigrún (queen)
        (synthesizes final draft)
                   │
                   ▼
            Final answer
         (tuned to your preferences)
```

---

## Key design decisions

- **Parse once upstream.** All K models share the same extracted text. Independent parsing burns tokens and produces inconsistent results.
- **Rank + critique together.** Rankings give a clean winner signal; critiques give Sigrún the *reasons*, which is what she needs to merge well. Karpathy's version leans on ranking; adding critiques is the core enhancement here.
- **Exclude self-votes.** Even anonymized, models can sometimes recognize their own style. Each reviewer scores only the K−1 drafts that aren't theirs.
- **Sigrún sees everything.** Query, context pack, all drafts, all rankings, all critiques — full picture for synthesis.
