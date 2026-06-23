# TakeMeter — Project 3 Checklist (AI201, Week 3)

**Due:** Monday, June 22nd @ 2:59 AM EDT · **Est. effort:** ~9–11 hours
**Goal:** Fine-tune a text classifier that rates discourse quality in an online community, then honestly evaluate it.

---

## 0. Setup
- [ ] Create GitHub repo (e.g. `ai201-project3-takemeter`) for everything outside the notebook: `planning.md`, dataset CSV, `README.md`, `evaluation_results.json`, `confusion_matrix.png`.
- [ ] Make a copy of the starter Colab notebook (File → Save a copy in Drive).
- [ ] Set Colab runtime to **T4 GPU** (Runtime → Change runtime type) before running any cell.
- [ ] Add `GROQ_API_KEY` via Colab Secrets (🔑 icon) and enable notebook access. **Never commit the key.**

---

## Milestone 1 — Choose Community & Define Labels (~45 min)
- [ ] Pick one active, text-heavy community (e.g. r/nba, r/LetsTalkMusic, r/TrueFilm, r/leagueoflegends). Must be able to collect 200+ public posts/comments.
- [ ] Read 30–40 real posts before designing labels (don't design from memory).
- [ ] Define **2–4 labels**, each with: 1-sentence definition, 2 clear examples, 1 uncertain example.
- [ ] Labels must be: mutually exclusive, exhaustive (≥90% labelable w/o "other"), grounded in community norms, ≥20% each in distribution.
- [ ] Identify the **hardest edge case** (a post genuinely between two labels) and write a decision rule for it.
- [ ] Write a 2–3 sentence description of community + labels + why distinctions matter.
- **Checkpoint:** can state labels w/ definitions + examples, and name + handle the hardest edge case.

## Milestone 2 — Write `planning.md` Spec (~45 min)
Create `planning.md` in repo root (design your own structure) addressing all six:
- [ ] **Community:** what you chose and why it's a good fit.
- [ ] **Labels:** 2–4 labels, full-sentence definitions, 2 examples each.
- [ ] **Hard edge cases:** the ambiguous post type + how you'll handle it.
- [ ] **Data collection plan:** source, count per label, plan if a label is underrepresented.
- [ ] **Evaluation metrics:** which metrics and why (accuracy alone is not enough).
- [ ] **Definition of success:** specific performance threshold for "good enough."
- [ ] **AI Tool Plan section** covering: label stress-testing, annotation assistance (will you pre-label?), failure analysis — make an explicit decision on each.
- **Checkpoint:** all six answered substantively; success criteria are objectively measurable.

## Milestone 3 — Collect & Annotate Dataset (~3–4 hrs)
- [ ] Collect **≥200 public posts/comments** (manual copy-paste is fine; don't turn collection into a coding project).
- [ ] Save as a **single CSV** with columns: `text`, `label`, plus a notes column. **Do NOT pre-split** (notebook does 70/15/15).
- [ ] Label each example by reading it — no bulk skimming. (Optional: LLM pre-label, but review & correct every one; disclose it.)
- [ ] Keep a running list of difficult cases → document **≥3** in `planning.md` Hard Edge Cases.
- [ ] Count per label; if any label > 70%, collect more from underrepresented labels.
- **Checkpoint:** ≥200 examples in one CSV, no label > 70%, ≥3 documented hard cases.

## Milestone 4 — Run Baseline (~1 hr)
> **Notebook:** `Copy_of_ai201_project3_takemeter_starter_clean.ipynb`. Only 5 cells need editing (all marked `# ── TODO`); everything else is pre-written infrastructure — don't change it.
- [ ] Run install + import cells (top).
- [ ] **Section 1 — edit `LABEL_MAP`** (cell with the r/nba example): replace with YOUR labels → int (start at 0). Submitting the example unchanged will not pass.
- [ ] **Section 1 — upload CSV** cell (Colab file picker). Edit the `df.rename(...)` cell only if your columns aren't named `text`/`label`. Confirm "✅ All labels match your LABEL_MAP."
- [ ] **Section 2** — run split + tokenize; review split sizes & per-label distribution.
- [ ] **Section 5 — set `GROQ_API_KEY`** (Option A = Colab Secrets, recommended).
- [ ] **Section 5 — write `SYSTEM_PROMPT`**: name community + task, define each label, 1 example per label, instruct "output ONLY the label name." Output must match a label string exactly (parser is case-insensitive substring match).
- [ ] Run the baseline loop (`llama-3.3-70b-versatile`) → record accuracy + per-class metrics. If >~10% unparseable, tighten the prompt.
- [ ] Write a hypothesis: where did the baseline struggle / which labels confused?
- **Checkpoint:** baseline numbers saved; fine-tuned model NOT yet looked at.

## Milestone 5 — Fine-Tune Model (~1.5–2 hrs)
- [ ] Run **Section 3**: fine-tune `distilbert-base-uncased` (defaults: 3 epochs, LR 2e-5, batch 16). Note any hyperparameter changes + why.
- [ ] Run **Section 4**: evaluate on test set → per-class metrics + `confusion_matrix.png`. Pick **3 wrong predictions** to analyze.
- [ ] Run **Section 6**: side-by-side baseline vs fine-tuned + write `evaluation_results.json`.
- [ ] Download & commit `evaluation_results.json` and `confusion_matrix.png`.
- **Checkpoint:** fine-tuning succeeded; results comparable to baseline. (If worse across board → check label leakage, imbalance, training bug.)

## Milestone 6 — Evaluate, Document & Record (~1–2 hrs)
- [ ] (Optional but recommended) Use an LLM to surface patterns in wrong predictions; **verify patterns yourself**.
- [ ] Write **evaluation report directly in README**, including:
  - [ ] Overall accuracy for **both** models
  - [ ] Per-class metrics (precision/recall/F1) for both models
  - [ ] Confusion matrix as a **markdown table** in README (not only the PNG)
  - [ ] **≥3 wrong predictions** with deep analysis (which labels confused, why the boundary is hard, labeling vs data problem, what would fix it)
  - [ ] **Sample Classifications** subsection: 3–5 posts through fine-tuned model w/ predicted label + confidence; ≥1 correct example explained
- [ ] Write **reflection**: what the model captured vs. what you intended (overfit / missed).
- [ ] Write **spec reflection**: one way the spec helped, one way implementation diverged + why.
- [ ] Write **AI usage section**: ≥2 specific instances (what you directed, what it produced, what you changed); disclose any annotation assistance.
- [ ] Record **3–5 min demo video**: 3–5 classifications w/ label+confidence, 1 correct narrated, 1 incorrect narrated, walkthrough of eval report.

---

## Final Submission (via Course Portal)
- [ ] Link to GitHub repo
- [ ] `planning.md` in repo root
- [ ] Labeled dataset CSV in repo (or linked from README)
- [ ] `README.md` with: community + reasoning · label taxonomy (defs + 2 examples each) · data collection/labeling/distribution + 3 hard cases · fine-tuning approach + ≥1 hyperparameter decision · baseline description (prompt + method) · full eval report (both models, confusion matrix table, 3 wrong preds, sample-classifications table) · reflection · spec reflection · AI usage section
- [ ] Demo video (3–5 min)

## Stretch (extra credit — update `planning.md` before each)
- [ ] Inter-annotator reliability (2nd person labels 30+, report Cohen's kappa / % agreement)
- [ ] Confidence calibration (are confidence scores meaningful?)
- [ ] Error pattern analysis (systematic error pattern, not just a list)
- [ ] Deployed interface (accepts a post → shows label + confidence; document how to run)
