# Practice-exam trainer

A single-file, zero-dependency multiple-choice practice tool that mimics the
PSI online exam environment used for the CNCF associate certifications on the
Golden Kubestronaut path (CAPA, PCA, CGOA, OTCA, KCNA, KCSA, …).

You load one **question bank** (a `.json` file) per certificate and take a
timed, exam-style practice run.

## Running it

**Easiest — no server:**
1. Double-click `quiz.html` (it opens in your browser via `file://`).
2. Click **Choose a .json file** (or drag one onto the page) and pick a bank,
   e.g. `banks/capa.json`.
3. Pick a mode + options and hit **Start exam**.

**Optional — with a local server** (enables the `?bank=` shortcut):
```sh
cd quiz
python3 -m http.server 8000
# then open:  http://localhost:8000/quiz.html?bank=banks/capa.json
```
The `?bank=` URL parameter only works when served over HTTP (browsers block
`fetch()` of local files under `file://`); without it the tool falls back to
the file picker.

Everything runs locally in the browser — no data leaves your machine.

## Modes & options

- **Exam mode** (default): countdown timer, no feedback until you submit, a
  review screen, and auto-submit when time runs out — faithful to the real exam.
- **Study mode**: untimed; a **Check answer** button reveals the correct
  option(s) and the explanation immediately, one question at a time.
- **Shuffle questions / shuffle answer options**: independent toggles. Scoring
  is index-safe, so shuffling never affects correctness.

Multi-answer questions are scored **all-or-nothing** (you must select exactly
the correct set), matching the real PSI exams.

## Bank file format (JSON)

```jsonc
{
  "title": "Certified Argo Project Associate (CAPA)",  // shown in the header
  "cert": "CAPA",                                       // short code (optional)
  "timeLimitMinutes": 90,                               // countdown length
  "passingScore": 75,                                   // pass threshold (%)
  "version": "2026.06",                                 // optional, for tracking
  "questions": [
    {
      "id": "cd-001",                       // optional; auto-assigned if omitted
      "type": "single",                     // "single" or "multi"
      "category": "Argo CD Fundamentals",   // groups the score breakdown by topic
      "stem": "What is Argo CD's source of truth for desired state?",
      "options": [
        "The live cluster",
        "A Git repository",
        "The controller's Redis cache",
        "A Helm chart registry"
      ],
      "answer": 1,                          // zero-based index into "options"
      "explanation": "Git holds the desired state; the controller reconciles toward it."
    },
    {
      "id": "rollouts-014",
      "type": "multi",
      "category": "Progressive Rollout Strategies",
      "stem": "Which strategies does an Argo Rollouts Rollout support? (Select all that apply.)",
      "options": ["Canary", "Blue-Green", "Recreate-only", "Mirror/shadow traffic"],
      "answer": [0, 1, 3],                  // array of zero-based indices
      "explanation": "Canary and blue-green are the progressive strategies; mirror needs a supported provider."
    }
  ]
}
```

### Field reference

| Field | Level | Required | Notes |
|---|---|---|---|
| `title` | bank | recommended | Header text; falls back to the file name. |
| `cert` | bank | optional | Short code shown as a badge. |
| `timeLimitMinutes` | bank | optional | Defaults to `90`. |
| `passingScore` | bank | optional | Percent; defaults to `75`. |
| `version` | bank | optional | Free-text revision tag. |
| `questions` | bank | **yes** | Non-empty array. |
| `id` | question | optional | Stable id; auto-assigned (`q1`, `q2`, …) if missing. |
| `type` | question | optional | `"single"` or `"multi"`; inferred from `answer` if omitted. |
| `category` | question | optional | Subtopic label for the per-topic breakdown; defaults to `"General"`. |
| `stem` | question | **yes** | Question text. Supports `` `code` ``, `**bold**`, and ```` ```fenced blocks``` ````. |
| `options` | question | **yes** | ≥ 2 strings (4 recommended). |
| `answer` | question | **yes** | `single`: one integer index. `multi`: array of integer indices. All zero-based into `options`. |
| `explanation` | question | optional | Shown in study mode and on the results review. |

The loader validates the file on load and shows a specific error (e.g.
*"Question 7: answer index 4 is out of range for 4 options"*) and refuses to
start if anything is wrong.

## Layout

```
quiz/
├── quiz.html        # the whole tool (open this)
├── README.md        # this file
└── banks/
    └── capa.json    # Certified Argo Project Associate question bank
```

Add more banks by dropping additional `.json` files into `banks/`.
