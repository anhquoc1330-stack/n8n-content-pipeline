# Daily Content Pipeline

An n8n workflow that reads the morning's AI news, decides what is worth writing about, drafts the posts in a specific editorial voice, and hands three finished drafts to a human for approval — every day at 07:00, unattended.

It runs the daily publishing loop of **Tạp Hoá AI**, a Vietnamese-language channel that explains practical AI to readers who are not engineers.

**Stack:** n8n · Claude (Anthropic API) · RSS · Gmail API · JavaScript
**Type:** Scheduled ETL + LLM editorial judgment + human-in-the-loop approval
**Scale:** 28 nodes, 4 sources, runs unattended at 07:00 daily

---

## The problem

Tạp Hoá AI publishes practical AI explainers for a Vietnamese audience. The daily routine before automation:

1. Read several English-language AI newsletters and feeds (~60 items/day)
2. Judge which stories actually matter to a Vietnamese reader who is not an engineer
3. Write each post in a specific voice — direct, grounded, first-person, explicitly anti-hype
4. Find a usable image
5. Publish

Steps 1–2 are judgment, step 3 is craft, steps 4–5 are mechanical. The bottleneck is that all five have to happen *every day* before the audience's morning scroll, and the quality of step 2 silently determines whether the whole day's work was worth doing.

## What I built

![n8n workflow canvas](./assets/workflow-canvas.png)

*The pipeline in n8n. The item counts on each connection are from one real morning run: 57 articles collected across sources → 34 inside the 7-day window → 9 scored at or above 3.5 → top 3 written, illustrated and emailed. Note the shape of that funnel — the expensive LLM writing step only ever sees 3 items, because two cheap filters run before it.*

```
Schedule Trigger 07:00
        │
        ├── RSS: TLDR AI · TechCrunch · VentureBeat · The Verge
        │       (each tagged with its source via a Set node)
        │
        ▼
    Merge  →  Filter: published within last 7 days
        │
        ▼
    LLM Chain (Claude) + Structured Output Parser
        → { score: 1-5, category, reasoning }
        │
        ▼
    IF score >= 3.5  ──(below)──▶ dropped
        │
        ▼
    Aggregate → Pick Top 3 (JS)
        │
        ▼
    Loop over each article
        ├── Classify: breaking_news | deep_analysis
        ├── LLM Chain (Claude): write the post in brand voice
        ├── HTTP fetch source page → extract og:image
        └── Merge content + image
        │
        ▼
    Compose approval email (JS) → Gmail
        │
        ▼
    Human reads, edits, publishes
```

## The interesting part: making an LLM hold *my* editorial standard

"Pick the best articles" produces generic tech-blog taste. I replaced it with an explicit weighted rubric in the scoring prompt:

| Criterion | Weight | What it actually asks |
|---|---|---|
| Novelty | 30% | Is this new, and has nobody in the Vietnamese AI space covered it yet? |
| Impact | 35% | How many people does it affect, and does it change how they work? |
| Virality | 20% | Would this get shared and argued about on Vietnamese social media? |
| Actionability | 15% | Can a reader apply this to their job today? |

Three design choices around it:

- **A structured output parser** enforces `{score, category, ly_do}` as a JSON schema, so a malformed LLM response fails loudly instead of poisoning downstream nodes.
- **The model must explain its score** in 1–2 sentences (`ly_do`). This is not decoration — it is what makes the rubric tunable. When the pipeline picks a bad article, the explanation tells me whether the rubric or the model was wrong.
- **The 3.5 threshold runs before top-N selection.** Taking the top 3 unconditionally guarantees three posts a day, including on days when nothing happened. Filtering first means a quiet day yields one post or none. Volume is the variable; the quality bar is fixed.

## Encoding voice, not just format

The writing node branches on content type and carries the full brand voice spec into the prompt — tone ratio, person and address forms, a required two-sided assessment on every claim, banned hype vocabulary, and a hard length range per format.

The constraint that mattered most: **every point must state both what is good and what is not**. A single-sided post reads like marketing copy, which is precisely the thing the channel positions itself against. Making that a mandatory structural element of the prompt, rather than a stylistic suggestion, is what kept generated drafts on-brand.

## What the human actually receives

![approval email](./assets/approval-email.png)

*The end of every run: one email, three ready-to-post drafts, each with its image and source link. Approving is a copy-paste; rejecting is deleting. The system never decides to publish.*

[**Full email — all three drafts (PDF)**](./assets/approval-email-full.pdf)

Each draft arrives in final form — headline, body in brand voice, hashtags, source attribution, and a scraped image — so the human decision is binary rather than editorial. That framing is the whole point of stopping here: reviewing three finished drafts takes a few minutes, while *writing* three drafts takes hours, and the reviewer still catches anything the model got wrong before it reaches an audience.

## Trade-offs

| Decision | Why | Cost |
|---|---|---|
| Stops at an approval email instead of auto-posting | A wrong post published under the brand name is expensive and hard to walk back; a delayed post is not. | Requires ~15 min of human attention daily. |
| `og:image` scraped from the source page | Zero marginal cost, always topically relevant. | Some sites return no usable image; those drafts need a manual image. |
| 7-day recency window, not 24 hours | Weekly newsletters like TLDR batch their coverage; a 24h window silently drops them. | Occasional re-surfacing of an already-covered story. |
| Weighted rubric inside the prompt | Editable without touching workflow logic. | Rubric changes are untested until the next run. |

## What I would do next

- Log every score with its explanation to a sheet, then check the rubric's weights against what actually performed — currently the weights are my prior, not evidence.
- Deduplicate across sources; the same story appearing in TLDR and TechCrunch currently competes with itself for a top-3 slot.
- Track approval rate as the pipeline's real success metric: if I approve 3 of 3 drafts unedited, the rubric is working.

## Running it

1. Import `workflow/content-pipeline.json` into n8n.
2. Configure two credentials: **Anthropic API** (both LLM Chain nodes) and **Gmail OAuth2** (send node).
3. Set the recipient address on the Gmail node.
4. Adjust the schedule trigger to your timezone.

Full setup guide — prerequisites, credentials, staged testing, a node-by-node map, customisation and troubleshooting:

- [`docs/setup-guide-en.docx`](./docs/setup-guide-en.docx) — English
- [`docs/setup-guide-vi.docx`](./docs/setup-guide-vi.docx) — Tiếng Việt

---

Built and operated by **Canh Vu** — International Business, Foreign Trade University.

No API keys, tokens or credentials are committed. The Gmail node's recipient is a placeholder; replace it before running.
