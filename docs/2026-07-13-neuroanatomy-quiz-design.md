# Neuroanatomy practice-quiz — design spec

Date: 2026-07-13. Content language: Hebrew (RTL). Built per `PLAYBOOK.md`.

## 1. Goal

A self-contained study app for a medical-school neuroanatomy exam: ~300 sourced
multiple-choice questions across 8 anatomical topics, with practice / test / exam
modes and an Insights tab that tells the student which topics to work on.

## 2. Architecture (fixed by PLAYBOOK)

- Single `index.html`: React 18 UMD from CDN, JSX compiled in-browser by
  `@babel/standalone` using the **classic** runtime via the `type="text/plain"` +
  manual `Babel.transform(...classic...)` + `(0,eval)` pattern. No build step.
- One `data_<topic>.js` per topic, each `window.QUESTIONS_<TOPIC> = [ ... ];`,
  loaded by plain `<script src>` (not fetch — must work over `file://`).
- RTL layout throughout; UI copy in Hebrew.

## 3. Question schema (per PLAYBOOK §2)

```json
{
  "id": "cbl-001",
  "topic": "cerebellum",
  "question": "…",
  "options": ["…","…","…","…"],
  "correctIndex": 2,
  "explanation": "why correct — shown after answering",
  "reference": { "doc": "source name", "page": 90, "section": "short label" },
  "source": "summary | past_exam | variant"
}
```
Invariants: exactly 4 options; `0 <= correctIndex <= 3`; ids unique; every question
has `reference.page`; no two options with identical text.

## 4. Topics (8, anatomical subject, ~37 Q each → ~300)

| key | Hebrew label | source coverage |
|---|---|---|
| `spinal_tracts` | חוט השדרה ומסילות | spinal cord + ascending/descending tracts |
| `cortex_limbic` | קורטקס, המיספרות ולימבי | cortex, hemisphere surfaces, functional localization, limbic |
| `diencephalon` | דיאנצפלון | general §diencephalon + dedicated diencephalon doc |
| `basal_ganglia` | גרעיני הבסיס | general §basal ganglia + dedicated Herut doc |
| `brainstem_cn` | גזע המוח ועצבים קרניאליים | brainstem + cranial nerves |
| `cerebellum` | צרבלום | cerebellum |
| `ventricles_meninges` | חדרים, קרומים ו-CSF | ventricular system, meninges, CSF |
| `vasc_white` | אספקת דם, חומר לבן ודימות | blood supply, white matter, imaging/mapping |

## 5. Modes & UX (per PLAYBOOK §5)

- **Practice**: reveal correct/incorrect + explanation + reference immediately;
  question locks once answered.
- **Test**: neutral "selected" highlight only, no right/wrong color; "answered X/Y"
  counter; reveal everything only on the results screen.
- **Exam**: one-click simulate-the-real-exam. Real exams are ~36 flat questions
  spanning all topics with **no fixed per-topic block order**, so exam mode draws a
  **fresh random 36 across the full pool** each run. Runs under test-mode rules.
- Free Prev/Next regardless of answered state; answers stored in `answers[i]` array.
- "Finish now" escape hatch from any question.
- Results screen shows every question by default, mistakes-only filter toggle; each
  item shows status, correct answer, explanation, reference.
- "Retry missed only" always runs in practice mode.

## 6. Insights tab (groundwork)

Persistence: `localStorage["neuroquiz_history_v1"]` = array of attempts
`{ qid, topic, correct, chosenIndex, ts, mode }`. Written immediately in practice
mode, on results-commit in test/exam mode. Helpers `recordAttempt()` /
`computeInsights()` kept isolated so the tab can be extended later.

Tab shows:
- Per-topic accuracy (attempted / correct / %).
- Weakest topics ranked, with a minimum-attempts threshold.
- Repeated mistakes: qids missed ≥2 times, with topic + jump-to-review link.
- Overall stats + a "recommended focus" summary (bottom-N topics + repeat misses).
All client-side; no schema change (questions already carry `topic`).

## 7. Content pipeline (Sonnet workers; orchestrator runs the cheap scripts)

0. Sources already extracted to `sources/*.txt` via `pdftotext -layout`.
1. **Mine** — 8 parallel Sonnet agents, one per topic. Each reads its source slice
   in full, gets 2–4 real past-exam examples for calibration, writes ~30 questions
   to a topic-prefixed scratch JSON, saving incrementally every ~10. Hard rule:
   never invent a fact; skip if unsure. Diencephalon/basal-ganglia agents also get
   their dedicated docs.
2. **Expand** — deeper mining + past-exam mining + tagged `variant`s to reach ~300.
   Past-exam mining must respect sitting boundaries (§8) — never cross answer keys.
3. **Audit** — orchestrator runs all detector/validation scripts itself: schema,
   dup ids, 4-options, correctIndex range, position-bias `Counter`, length-bias
   ratio, stem-leak, explanation-vs-index. Dispatch Sonnet fix agents only for
   rewrite-heavy issues (4.1 real bugs, 4.3 length rewrites); re-run detectors after;
   apply final shuffle-and-reindex normalization as the last step.
4. **Build `index.html`** — one Sonnet agent builds it fresh from PLAYBOOK §1/§5/§6
   plus §6 of this spec (Insights). Orchestrator reviews.
5. **Deploy** — GitHub + Vercel per PLAYBOOK §7; orchestrator runs the CLI, flags the
   one browser-auth step (GitHub App connect) that only the user can complete.

## 8. Source-specific hazards

- `pastexams_recollections.txt` mixes multiple theory sittings (מועד א'/ב' 2024, …),
  each followed by its own `תשובות` answer key. Match questions to the answer key
  **within the same sitting only**; never by coinciding question numbers across
  sittings.
- The recollections also contains **practical/image exams** (מבחן מעשי) that reference
  physical specimens — **exclude these from mining** (not convertible to text MCQ).
- `pastexam_2025_moedA.txt` is a clean single sitting (~36 Q) with an answer-key
  table at the end (`מסיח נכון`) — good calibration + minable source.

## 9. Decisions taken (not open questions)

- Exam mode = 36 random cross-topic (only fixed structure real exams have).
- Practical/image exams excluded from mining.
- Insights is a working tab, not a stub.

## 10. Deliverables

`index.html`, `data_<topic>.js` ×8, `README.md`, plus this spec and PLAYBOOK in repo.
Deployed to Vercel.
