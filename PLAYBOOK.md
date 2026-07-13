# Playbook: building an interactive practice-quiz app like this one

This document is written for an LLM (or a human) tasked with building a similar app for
**different content** — a different course, a different exam, a different language. It
captures the architecture, the content-generation pipeline, and — most importantly — every
non-obvious bug this specific build hit, so they don't have to be rediscovered.

Treat `index.html` in this repo as the reference implementation. This playbook explains the
*why* behind its choices and gives copy-pasteable scripts for the parts that are easy to get
subtly wrong.

---

## 1. What this actually is

A single self-contained `index.html` (React + inline CSS, no build step) plus one plain
JavaScript data file per content topic (`data_<topic>.js`, each just assigning an array to a
`window.QUESTIONS_<TOPIC>` global). No bundler, no npm install, no server required to run it —
double-click `index.html` and it works, because:

- React 18 / ReactDOM are loaded from a CDN as UMD globals (`<script src="https://unpkg.com/react@18/umd/react.production.min.js">`).
- JSX is transformed **in the browser** at load time via `@babel/standalone`.
- Question data is loaded via plain `<script src="data_x.js">` tags (which work over `file://`),
  **not** `fetch()` of a `.json` file (which fails over `file://` due to CORS in some browsers).

This tradeoff (no build step, CDN dependencies) is the right one for a small study tool that
needs to be trivially portable and forkable — don't reach for Vite/webpack/npm here unless the
project genuinely grows beyond this scope.

### The one Babel gotcha that will cost you an hour if you don't know it

Do **not** write your JSX into `<script type="text/babel">` and rely on `@babel/standalone`'s
auto-scan-and-transform behavior. Two independent problems:

1. **JSX runtime mismatch.** Recent Babel defaults to the "automatic" JSX runtime, which
   compiles `<div/>` into `import { jsx as _jsx } from "react/jsx-runtime"` — an ES module
   import that cannot run in a plain browser `<script>` with no module loader. You'll get
   `Cannot use import statement outside a module` at eval time, or (worse) it just silently
   fails to render anything with no console error, because the auto-scanner swallows the error.
2. Explicitly force the **classic** runtime instead, and transform manually so failures aren't
   swallowed:

```html
<script type="text/plain" id="app-source">
  // ... all your JSX/component code goes here, completely normal React ...
  ReactDOM.createRoot(document.getElementById("root")).render(<App />);
</script>
<script>
(function () {
  var src = document.getElementById("app-source").textContent;
  var compiled = Babel.transform(src, { presets: [["react", { runtime: "classic" }]] }).code;
  (0, eval)(compiled);
})();
</script>
```

Using `type="text/plain"` (not `text/babel`) means Babel's auto-scanner ignores the tag
entirely, and you control exactly when/how it's compiled and run — no timing races, no
swallowed errors, no accidental automatic-runtime imports.

---

## 2. Question data schema

Every question, regardless of topic, has this shape:

```json
{
  "id": "abd-001",
  "topic": "בטן",
  "question": "question text, may embed technical terms in a different language/script",
  "options": ["option A", "option B", "option C", "option D"],
  "correctIndex": 2,
  "explanation": "why the correct option is correct — shown after answering",
  "reference": { "doc": "source document name", "page": 12, "section": "short label" },
  "source": "summary | past_exam | variant"
}
```

- `reference` is what lets the student jump back to the exact page/section of the source
  material to review a mistake — don't skip it, it's the single highest-value field for actual
  studying.
- `source` tracks provenance (mined from a course summary vs. a real past exam vs. a generated
  rephrasing of an earlier question) — cheap to add, useful later for auditing/filtering.
- Keep `options` to exactly 4 and always verify `0 <= correctIndex <= 3` — the whole app assumes
  this (bullet letters, keyboard layout, etc. are all hardcoded to 4).

Each topic's array is embedded as: `window.QUESTIONS_ABDOMEN = [ ...185 objects... ];` in its
own `data_abdomen.js` file. Converting a validated JSON array to this format:

```python
import json

def convert(src_json_path, window_var_name, dest_js_path):
    with open(src_json_path, encoding="utf-8") as f:
        data = json.load(f)
    ids = [q["id"] for q in data]
    assert len(ids) == len(set(ids)), "duplicate ids!"
    for q in data:
        assert len(q["options"]) == 4
        assert 0 <= q["correctIndex"] <= 3
        assert "reference" in q and "page" in q["reference"]
    with open(dest_js_path, "w", encoding="utf-8") as out:
        out.write(f"window.{window_var_name} = ")
        json.dump(data, out, ensure_ascii=False, indent=2)
        out.write(";\n")
```

---

## 3. The content-generation pipeline (this is most of the actual work)

Generating hundreds of accurate, well-sourced, appropriately-difficult multiple-choice
questions is not a single prompt — it's a pipeline with a mandatory verification stage. Skipping
verification is how bugs described in section 4 end up live in production.

### Step 0 — get source material into plain text

If the source is PDFs, extract with `pdftotext -layout` (install via `brew install poppler` if
`pdftotext`/`pdftoppm` aren't present) rather than reading them page-by-page as rendered images —
much cheaper and faster for text-heavy PDFs, and lets you `grep` across the whole corpus.

### Step 1 — mine an initial bank, one parallel agent per topic

Dispatch one agent per content topic (if the material naturally splits that way) to read its
full source text and write ~20-30 new questions as a pure JSON array to a scratch file. Give
each agent, explicitly:

- The full source text file path, with instructions to read it **in its entirety** (chunked
  `Read` calls with offset/limit if it's long) — not just skim the first page.
- **Hard rule: never invent a fact.** Every question must be traceable to something literally
  stated in the source text. If unsure, skip the question rather than guess.
- 2-4 concrete example questions from a *real* past exam (if you have any) pasted directly into
  the prompt, so the agent calibrates difficulty/style/phrasing against a real target instead of
  its own assumptions.
- The exact output JSON schema (section 2) and file path to write to.
- Instruction to double-check each question against the source text before finalizing.

Validate the output programmatically the moment it lands (schema, duplicate IDs, 4 options,
valid correctIndex) before trusting a single word of the agent's own "I verified everything"
self-report.

### Step 2 — expand volume

If you need more questions than a single mining pass yields (e.g. to support many
non-repeating mock exams drawn from a large pool), dispatch a second round per topic that:

- Mines *deeper* into the same source text for facts not used in round 1.
- If you have a large compilation of real past exam questions (e.g. a multi-year "recollections"
  document), mine those too — but see the specific warnings in section 4 about matching
  questions to the *correct* answer key when a document contains multiple exam sittings mixed
  together.
- Generates deliberate **variants** of existing questions (same fact, different phrasing/
  distractor set) to multiply volume without needing new source content — tag these
  `"source": "variant"`.
- **Instruct explicit incremental saving.** Agents can hit a session/usage limit mid-task and
  return having produced *nothing* if they only write output at the very end. Tell them: "write
  your cumulative output to the file after every batch of ~30 questions, don't wait until
  you're completely done." This alone saved multiple rounds of work in this build.
- **Use topic-prefixed filenames for intermediate/working files** (`abd_batch1.json`, not
  `batch1.json`) if multiple agents run concurrently against a shared scratch directory — one
  agent in this build nearly had its output silently overwritten by another agent's
  identically-named intermediate file.

### Step 3 — the mandatory audit/fix pass

Do not skip this. At this generation scale, agents *will* introduce the specific bug classes in
section 4, and the finished bank will look plausible on casual inspection while being
systematically gameable or subtly wrong. Budget real time for this — it took as many rounds as
the original generation in this build.

---

## 4. Bug classes you will hit, and how to detect/fix each one

These are not hypothetical — every one of these was found live in this build after an agent
reported "all done, fully verified."

### 4.1 — `correctIndex` contradicts its own `explanation`

The explanation text correctly describes the true answer, but `correctIndex` points at a
different option. Classic cause: converting a real exam question (which used its own A/B/C/D
lettering and a separate answer key) into this schema, and mis-mapping the letter to the new
array index — especially when the agent also reorders/trims options (e.g. condensing a
5-option original into 4).

**Detector** (cheap — no need to re-open giant source files):

```python
import json, re

letter_to_idx = {"א": 0, "ב": 1, "ג": 2, "ד": 3}  # adjust to your alphabet/language
pattern = re.compile(r"תשובה\s*([אבגד])")          # adjust to match how your agents cite answers

for q in data:
    for m in pattern.findall(q.get("explanation", "")):
        if letter_to_idx[m] != q["correctIndex"]:
            print("MISMATCH", q["id"])
            break
```

**Important:** this will also flag legitimate cases where an agent reordered options during
conversion and correctly updated `correctIndex` to match — the cited letter simply refers to the
*original* document's lettering, not the new array position. Every flagged item still needs a
human/agent to actually read it and confirm whether it's a real bug or a citation artifact
before "fixing" anything. In this build, roughly 15% of flags were real bugs and the rest were
false positives from legitimate reordering.

### 4.2 — systematic answer-position bias

Check the distribution of `correctIndex` across the *entire* bank:

```python
from collections import Counter
print(Counter(q["correctIndex"] for q in data))
```

Expect roughly even quarters. In this build, one topic came back **100% index 0** — every single
question's answer was position א — because the mining agent had a habit of always placing the
extracted-from-real-exam correct answer first. This makes the quiz trivially gameable
("always guess the first option") and is just as serious a defect as a factual error.

**Fix** — shuffle each question's own 4 options and recompute `correctIndex` by matching the
original correct option's *text* (not its old index) in the shuffled list:

```python
import random
random.seed(42)  # reproducible

for q in data:
    correct_text = q["options"][q["correctIndex"]]
    opts = q["options"][:]
    random.shuffle(opts)
    q["options"] = opts
    q["correctIndex"] = opts.index(correct_text)
```

This is safe **only if no question has two options with identical text** — check that first
(`len(set(q["options"])) == len(q["options"])` for every question), otherwise `.index()` can
resolve to the wrong occurrence.

### 4.3 — the correct answer is systematically the longest/most-detailed option

This is subtler than 4.2 and just as exploitable. LLM-written distractors tend to be short,
generic dismissals ("doesn't exist," "same as X," a bare noun) while the correct option gets a
justification clause ("X — because of Y, per Z"). A test-taker who has no idea what the right
answer is can still score well above chance by picking whichever option is longest or most
hedged.

**Detector:**

```python
strictly_longest = 0
ratio_sum = 0
for q in data:
    lens = [len(o) for o in q["options"]]
    ci = q["correctIndex"]
    others = [l for i, l in enumerate(lens) if i != ci]
    if lens[ci] > max(others):
        strictly_longest += 1
    ratio_sum += lens[ci] / (sum(others) / len(others))
n = len(data)
print(f"correct is strictly-longest: {100*strictly_longest/n:.1f}% (expect ~25% by chance)")
print(f"avg length ratio (correct / avg others): {ratio_sum/n:.2f} (expect ~1.0-1.3)")
```

In this build this came back at 60-70% (vs. an expected ~25%) with a length ratio of ~1.7-2.0x,
across a bank that had already passed casual review.

**Fix** requires an actual rewrite pass, not a mechanical script — you can't just truncate
strings without destroying meaning. Dispatch a fix agent with:
- The measured stats so it understands the scale of the problem.
- 2-3 concrete before/after examples pulled directly from *its own* data (the worst offenders —
  sort by length ratio descending and grab the top few) so it has a precise target to eliminate,
  not an abstract instruction.
- 2-3 examples of *good*, real exam-style options (short, parallel, same-category, genuinely
  confusable) as the style target to calibrate against.
- Explicit instruction: move any nuance/justification that made the correct option long into the
  `explanation` field instead (which is fine to be detailed, since it's only shown *after*
  answering) — keep the four options themselves terse and parallel.
- The same self-check script above, to be run *by the agent itself* before it reports done.

**Do not trust the agent's self-reported before/after numbers.** Re-run the detector yourself on
its output. In this build, one agent self-reported 30% when the real measured value was 56% —
still a real improvement over the ~61% baseline, but nowhere near what was claimed. Independent
verification caught this immediately; trusting the report would not have.

Also note: fixing 4.3 can *reintroduce* 4.2 if the fixing agent has its own positional habits
(e.g., tends to move the now-terser correct answer to a specific slot). Always re-check the
`correctIndex` distribution after *any* rewrite pass, and re-apply the shuffle-and-reindex fix
from 4.2 as a final normalization step regardless of what the length-bias numbers look like.

### 4.4 — the question stem gives away the answer

The question describes or names the very thing that is also the correct option's exact wording.
A noisy heuristic to find *candidates* for human/agent review (expect a high false-positive rate
— common connective words will trigger it):

```python
import re

def normalize(s):
    return re.sub(r"[^a-zA-Z0-9]+", " ", s).strip().lower()

for q in data:
    qtext = normalize(q["question"])
    correct = q["options"][q["correctIndex"]]
    for token in re.findall(r"[A-Za-z]{6,}", correct):  # tune length threshold to your content
        if normalize(token) in qtext:
            print("candidate leak:", q["id"], token)
            break
```

Fixing this requires actually reading the flagged question and judging whether it's a real
giveaway or coincidental term overlap (e.g. both stem and answer naturally mention the same
anatomical region) — don't auto-rewrite based on the heuristic alone.

### 4.5 — cross-referencing a multi-year "recollections" document

If your source material includes a compiled document of past exam questions spanning multiple
years/sittings mixed together (common for student-maintained "exam recollection" archives), be
aware:

- Each sitting typically has its own separate answer key, often with a header like "answers" /
  "answer key" located after (sometimes *before*, depending on how the document was assembled)
  its matching question list.
- **Never let an agent match a question to an answer key from a different sitting** just because
  question numbers happen to coincide (sitting A's "question 28" and sitting B's "question 28"
  are unrelated). Require the agent to identify sitting boundaries first (look for the section
  headers naming lecturers/dates/cohort) and only match within the same section.
- If the correct answer for a mined question can't be confidently matched within its own
  section, the question should be dropped — not guessed.
- Cohort/class labels that look like future years (e.g. a document from 2026 referencing "class
  of 2028") are normal in institutions that name student cohorts by expected graduation year —
  don't assume this is a hallucinated date without checking.

---

## 5. Product features and UX patterns worth keeping

These aren't specific to anatomy or Hebrew — they're generically good patterns for any
practice-quiz tool:

- **A one-click "simulate the real exam" mode**, separate from free-form practice, if the real
  exam has a known fixed structure (e.g. a fixed number of questions per topic in a fixed
  order). Draw a **fresh random subset** from the full question pool on every run (don't hardcode
  a fixed set of 60 questions) so repeated mock exams don't repeat and feel stale.
- **Two answering modes:**
  - *Practice*: reveal correct/incorrect + explanation + reference immediately after each
    answer; lock the question once answered (can't flip-flop after seeing the reveal).
  - *Test*: no reveal at all while answering — just a neutral "this is selected" highlight, no
    color-coding of right/wrong. Show an "answered X/Y" counter instead of a running score so
    nothing leaks. Only reveal everything on the results screen after finishing.
- **Free navigation.** Previous/Next should work regardless of whether the current question has
  been answered — don't force answering before advancing. This requires storing answers in an
  array indexed by question position (`answers[i]`), not a single "current selection" variable,
  so that navigating back to a previously-answered question correctly restores its state
  (including re-showing practice-mode's reveal if applicable).
- **A "finish now" escape hatch** visible from any question, not just the last one.
- **Results screen defaults to showing every question**, not just the missed ones — with a
  filter toggle to narrow to mistakes only. Each item gets a clear status
  (correct/wrong/unanswered), the correct answer, the explanation, and the source reference.
  Skip the redundant "here's what you answered" line when the student got it right.
- **"Retry missed only"** should always run in practice mode (immediate feedback) regardless of
  which mode the original run used — you want feedback when specifically reviewing mistakes,
  even if you took the original run in test mode.

---

## 6. Local testing gotchas

- When scripting a headless browser to click through the quiz for testing, **insert a short
  delay between "click an option" and "click Next"** — React's state update doesn't necessarily
  flush to the DOM synchronously within the same script tick, so querying the Next button's
  `disabled` attribute immediately after clicking an option can read stale state and silently
  no-op. `await new Promise(r => setTimeout(r, 30-50))` between actions is enough.
- If your content is RTL (Hebrew/Arabic) and you display something like `"{i+1} / {total}"`,
  be aware the bidi algorithm can visually reorder the numbers around the `/` in a way that looks
  wrong at a glance but is actually correct in the DOM (`textContent` will show the real order) —
  don't chase a phantom bug here, verify via `element.textContent`, not a screenshot, when in
  doubt.
- A local static server is the simplest way to preview changes before deploying. If using
  Python's built-in server and it fails with a `PermissionError` on `os.getcwd()` when pointed at
  a path outside your working sandbox, switch to a Node-based static server instead
  (`npx --yes serve -l <port> <absolute-path>`) — it doesn't hit the same sandboxing issue.

---

## 7. Deployment (GitHub + Vercel)

```bash
cd your-app-folder
git init && git branch -M main
git add index.html data_*.js README.md
git commit -m "Initial commit"
git remote add origin https://github.com/<user>/<repo>.git
git push -u origin main

vercel login          # device-flow auth; may auto-approve if already logged into vercel.com
vercel --prod --yes   # deploys and aliases to https://<project>.vercel.app
```

**Known Vercel CLI bug** (hit on CLI 52.x-54.x): running `vercel --prod` or `vercel link` in a
directory that has a git remote pointing at GitHub, but has **no existing Vercel project linked
yet**, can crash with `Error: Detected linked project does not have "id"`. Root cause: the CLI's
auto-detection queries `GET /v9/projects?repoUrl=...`, gets back a legitimately empty
`{"projects": [], ...}` (no project exists yet), and then mishandles the empty array as if it
found a match. Upgrading the CLI does not fix this.

**Workaround:** deploy once from a plain copy of the files with **no `.git` folder present**
(e.g. copy just `index.html` + `data_*.js` to a scratch directory and `cd` there) — this skips
the buggy repo-based auto-detection path entirely and creates the project cleanly. Then, to link
your real git-tracked directory to that same project for future deploys, fetch the project's
real ID from the API and write it directly:

```bash
# get the project id (created by the workaround deploy above)
curl -s "https://api.vercel.com/v9/projects/<project-name>?teamId=<team-id>" \
  -H "Authorization: Bearer $(python3 -c "import json; print(json.load(open('$HOME/Library/Application Support/com.vercel.cli/auth.json'))['token'])")"

# then in your real git-tracked directory:
mkdir -p .vercel
cat > .vercel/project.json <<EOF
{"projectId":"prj_...","orgId":"team_...","projectName":"your-project-name"}
EOF
echo ".vercel" >> .gitignore
```

After that, `vercel --prod --yes` from the real directory works normally (the CLI skips
auto-detection once a local link already exists).

**Connecting GitHub for auto-deploy-on-push** requires a one-time authorization in the browser
(the Vercel GitHub App needs explicit access granted to that specific repo) — this cannot be
completed non-interactively. Point the user to
`https://vercel.com/<team>/<project>/settings/git` → "Connect Git Repository". Once connected,
every `git push` to the default branch triggers an automatic production deploy — no more manual
`vercel --prod` needed.

---

## 8. Recipe: refitting this for entirely new content

1. Gather source material as plain text (extract PDFs first — section 3, step 0).
2. Decide the topic/block structure: what are the separate subject areas that need their own
   question pool? Does the real exam (if one exists) have a fixed per-topic question count and
   order worth simulating as a one-click mode?
3. Run the full content-generation pipeline from section 3 for each topic — mine, expand,
   **audit** (section 4 is not optional).
4. Produce `data_<topic>.js` files in the schema from section 2, one per topic.
5. In `index.html`, the only app-logic changes needed are:
   - The `TOPICS` array (topic keys, display labels, which `window.QUESTIONS_*` global each
     one maps to).
   - `EXAM_BLOCK_ORDER` (if you have a fixed-structure exam-simulation mode) — the order and
     which topics are included.
   - Header title/description text and footer copy that names the specific subject.
   - Everything else (both answering modes, navigation, results screen, filters) is
     content-agnostic and needs no changes.
6. Test locally by opening `index.html` directly, then via a local static server before
   deploying (section 6).
7. Deploy per section 7.

## 9. File manifest of a working instance

```
your-app-folder/
├── index.html          # all app logic + styles + markup, single file, no build step
├── data_topic1.js       # window.QUESTIONS_TOPIC1 = [...]
├── data_topic2.js       # window.QUESTIONS_TOPIC2 = [...]
├── data_topic3.js       # ...one per topic
├── README.md            # what the app is, for repo visitors
└── PLAYBOOK.md           # this file — how to build another one
```
