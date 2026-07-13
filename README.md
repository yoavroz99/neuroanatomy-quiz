# כלי תרגול בנוירואנטומיה — Neuroanatomy Practice Quiz

A self-contained, single-file study app for a medical-school neuroanatomy exam.
**441 sourced multiple-choice questions** across 8 anatomical topics, in Hebrew (RTL),
with practice / test / exam-simulation modes and an Insights tab that tells you which
topics to work on.

## Running it

No build step, no server needed — just open `index.html` in a browser. (For local
development a static server is convenient: `npx --yes serve -l 5599 .`)

- React 18 + ReactDOM are loaded from a CDN as UMD globals.
- JSX is compiled in-browser by `@babel/standalone` (classic runtime).
- Question data is loaded via plain `<script src="data_<topic>.js">` tags.

## Modes

- **תרגול (Practice)** — immediate feedback after each answer: correct/incorrect,
  explanation, and a source reference; the question locks once answered.
- **מבחן תרגול (Test)** — no feedback while answering, an "answered X/Y" counter, and
  full results only at the end.
- **סימולציית מבחן (Exam)** — a fresh random 36 questions across all topics each run,
  mirroring the real exam.
- **תובנות (Insights)** — per-topic accuracy, weakest-topic ranking, and repeated
  mistakes, computed from your local answer history (stored in the browser only).

## Topics

חוט השדרה ומסילות · קורטקס, המיספרות ולימבי · דיאנצפלון · גרעיני הבסיס ·
גזע המוח ועצבים קרניאליים · צרבלום · חדרים, קרומים ו-CSF · אספקת דם, חומר לבן ודימות

## Content provenance

Every question is traceable to the course summaries it was mined from (Noga Lampel's
general neuroanatomy summary, plus dedicated diencephalon and basal-ganglia summaries),
calibrated in style/difficulty against real past exams. Each question carries a
`reference` (source doc, page, section) so a wrong answer links straight back to the
material to review.

## Files

```
index.html                     # all app logic + styles + markup (single file)
data_<topic>.js  (×8)          # window.QUESTIONS_<TOPIC> = [...]
docs/…-design.md               # design spec
PLAYBOOK.md                    # how this class of app is built
```

Data is private and stays in your browser (localStorage). Nothing is uploaded.
