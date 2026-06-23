# TakeMeter — r/ConspiracyTheory Discourse-Quality Classifier

A fine-tuned DistilBERT text classifier that sorts r/ConspiracyTheory posts by **how a take is argued** (not whether it's true), compared against a zero-shot Groq (`llama-3.3-70b-versatile`) baseline.

**Headline result:** on this subjective 3-class task at 300 examples, fine-tuning **underperformed** the zero-shot baseline and **collapsed to a single class** in two separate training runs. This README documents that failure honestly and diagnoses why — which, per the project's own guidance, is the most valuable outcome here.

> Design reasoning and hard annotation calls live in [planning.md](planning.md); this README is the standalone final report.

---

## 1. Community & Reasoning

r/ConspiracyTheory holds wildly varying discourse quality on a constant topic — a single thread can mix a sourced, argued theory, a bare "wake up" assertion, and off-topic banter. That makes "what is a good take here" a real distinction worth classifying. The labels measure argument *structure*, not factual correctness — a model can't adjudicate conspiracy facts, and discourse quality is what the project targets. _(Full rationale in [planning.md §1](planning.md).)_

## 2. Label Taxonomy

| Label | Definition | Example 1 | Example 2 |
|-------|------------|-----------|-----------|
| `reasoned` | Argues a position (pro or debunking) with a reasoning chain, evidence, sources, or documented detail. | "A tiger was found to have the virus in a zoo in Kerala, so your theory is not correct." | "First sightings were 1904, but Bigfoot legends predate that by centuries, and genetic engineering wasn't possible until 1972 — the timeline doesn't hold up." |
| `assertion` | States a theory/opinion or asks a leading question with no supporting evidence. | "Another false flag effort to trigger a response against Iran." | "Trump is behind the recent Hollywood sex abuse scandals." |
| `noise` | Off-topic, jokes, banter, one-line reactions, meta — neither makes nor evaluates a claim. | "That's like saying oatmeal is soup because you add water." | "Okay, that's enough internet for me today." |

**Decision boundary:** (1) Does it make/evaluate a substantive claim? No → `noise`. (2) Backed by reasoning/evidence? Yes → `reasoned`, No → `assertion`. Hardest boundary + edge-case rules in [planning.md §3](planning.md).

## 3. Dataset

- **Source:** public pre-collected export of r/ConspiracyTheory posts + comments (`reddit_ct.csv`, ~1,197 rows). Cleaned via `prep_ct.py` (strip URLs, drop bots/deleted/junk, dedupe) → 300 candidates.
- **Labeling process:** candidates were **pre-labeled with an LLM** (Claude) from the label definitions, then reviewed and corrected by hand; rows that fit no label cleanly were dropped. Each row carries a `notes` field recording the signal behind its label. Final file: [dataset.csv](dataset.csv). _(AI assistance disclosed in §9.)_
- **Size:** 300 examples (notebook split 70/15/15 → 210 train / 45 val / 45 test).
- **Label distribution:**

  | Label | Count | % | Train n |
  |-------|-------|---|---------|
  | `reasoned` | 42 | 14% | 29 |
  | `assertion` | 126 | 42% | 88 |
  | `noise` | 132 | 44% | 93 |

  No label exceeds 70%, but **`reasoned` is heavily underrepresented at 14%** — only ~29 training examples. This imbalance is the central cause of the results below. It reflects reality: sourced, argued effortposts are genuinely rare in this subreddit relative to bare assertions and banter.

- **3 difficult-to-label examples + decisions:**
  1. *"...it takes 3000 degrees to melt glass that you can not get from a forest fire..."* — names a specific (pseudo-)fact as evidence. Sits between `reasoned` (cites a verifiable figure) and `assertion` (decorative evidence). **Decided `reasoned`** — it builds an argument from a stated fact, even if the fact is wrong (we label structure, not truth).
  2. *"Matt Damon killed Mark Wahlberg because he was paranoid about the ending to The Departed."* — an absurd joke, but phrased as a flat theory. Between `noise` (joke) and `assertion` (stated claim). **Decided `assertion`** — it asserts a propositional claim with no "just kidding" marker.
  3. *"According to Chinese sources... but a Washington Times report cites an Israeli microbiologist... Idk man. Sounds pretty fishy."* — casual, hedged tone but cites two named sources. Between `assertion` (casual register) and `reasoned` (sourced). **Decided `reasoned`** — sources + a mechanism survive the casual framing. (Posts like this — argued but casual in tone — are exactly the ones the model misclassifies; see §6.)

## 4. Fine-Tuning Approach

- **Base model:** `distilbert-base-uncased` (HuggingFace).
- **Two configurations tried:**
  - **Default:** 3 epochs, LR 2e-5, batch 16, best checkpoint by accuracy — collapsed onto the majority classes (never predicted `reasoned`).
  - **Class-weighted (rebalancing attempt):** inverse-frequency **class weights** in a custom weighted-loss `Trainer`, 5 epochs, best checkpoint by **macro-F1**. Code in `section3_weighted_retrain.py`.
  - **Outcome:** both stayed at ~0.49–0.51 accuracy with `reasoned` F1 = 0.00. The final reported model scores **0.511 / macro-F1 0.35**. Rebalancing shifted *which* class the model over-predicted, but did not teach it `reasoned`.
- **Key hyperparameter decision:** after the default run collapsed (never predicted the rare `reasoned` class), I rebalanced via **cost-sensitive class weights** rather than down-sampling — down-sampling to equal classes would drop the dataset below the required 200 examples (42×3 = 126). I also switched checkpoint selection from accuracy to **macro-F1**, since accuracy lets a model that ignores a whole class still "win."

## 5. Baseline

- **Model:** Groq `llama-3.3-70b-versatile`, zero-shot.
- **Prompt:** instructs the model to judge *how a take is argued, not whether it's true*, gives the 3 definitions + the decision rule + one real example per label, and constrains output to a single label word. (Full prompt below.)
- **How results were collected:** classified each of the 45 test posts; output matched against the label set; unparseable rate < 10%.

<details><summary>SYSTEM_PROMPT used</summary>

```
You are classifying posts and comments from r/ConspiracyTheory by the QUALITY OF THE
DISCOURSE — how a take is argued, NOT whether the claim is true.
reasoned: argues a position (supporting or debunking) with a reasoning chain, evidence,
  sources, dates, or documented detail.
assertion: states a theory/opinion or asks a leading question with NO evidence.
noise: off-topic, jokes, banter, one-line reactions, meta — no substantive claim.
Decision rule: (1) makes/evaluates a claim? No -> noise. (2) backed by reasoning? Yes ->
reasoned, No -> assertion.
Respond with ONLY one word: reasoned, assertion, or noise.
```
</details>

## 6. Evaluation Report

### Overall accuracy

| Model | Accuracy | Macro-F1 |
|-------|----------|----------|
| Zero-shot baseline (Groq) | **0.622** | **0.56** |
| Fine-tuned DistilBERT (final) | **0.511** | **0.35** |

Fine-tuning **lost to the baseline by 0.11 on accuracy and 0.21 on macro-F1.** Across both a default run and a class-weighted rebalancing attempt, accuracy stayed in the 0.49–0.51 range and `reasoned` F1 stayed at 0.00 — the collapse was robust to the fix (see §4).

### Per-class metrics — fine-tuned DistilBERT (final)

| Label | Precision | Recall | F1 | Support |
|-------|-----------|--------|----|---------|
| `reasoned` | 0.00 | 0.00 | **0.00** | 7 |
| `assertion` | 0.55 | 0.32 | 0.40 | 19 |
| `noise` | 0.50 | 0.89 | 0.64 | 19 |
| **macro avg** | 0.35 | 0.40 | **0.35** | 45 |

### Per-class metrics — zero-shot baseline (Groq)

| Label | Precision | Recall | F1 | Support |
|-------|-----------|--------|----|---------|
| `reasoned` | 0.38 | 0.43 | **0.40** | 7 |
| `assertion` | 0.61 | 0.89 | 0.72 | 19 |
| `noise` | 0.89 | 0.42 | 0.57 | 19 |
| **macro avg** | 0.62 | 0.58 | **0.56** | 45 |

**The decisive comparison:** the zero-shot baseline scores `reasoned` **F1 = 0.40** — imperfect, but it genuinely identifies argued posts with no training at all. The fine-tuned model scores `reasoned` **F1 = 0.00**. Fine-tuning didn't just fail to improve the rare class; it *destroyed* a capability the base LLM already had, because ~29 training examples taught the model to abandon the class entirely. The fine-tuned model only ties the baseline on `noise` (0.64 vs 0.57, where over-predicting `noise` happens to help); on `assertion` it's far worse (0.40 vs 0.72) and on `reasoned` it's total failure (0.00 vs 0.40). Macro-F1 drops by more than a third (0.35 vs 0.56).

### Confusion matrix — fine-tuned (final, test set)

Rows = true, columns = predicted. (Supplementary image: `confusion_matrix.png`.)

| true ↓ \ pred → | reasoned | assertion | noise |
|-----------------|----------|-----------|-------|
| **reasoned** | 0 | 3 | 4 |
| **assertion** | 0 | 6 | 13 |
| **noise** | 0 | 2 | 17 |

Two things stand out: the `reasoned` **column is entirely zero — the model never once predicted it** — and the model **over-predicts `noise` (34 of 45 posts)**. The single largest error cell is `assertion → noise` (13): bare claims get dumped into the banter bucket. Earlier training configurations collapsed similarly but in different directions — the default run leaned toward over-predicting `assertion`, the class-weighted run toward `noise` — which is itself evidence the model learned class *frequency*, not the distinctions (see §7).

### 3 wrong predictions analyzed

All three are true `reasoned` posts the model labeled `noise` — none of the 7 `reasoned` test posts were caught:

1. **"A tiger was found to have the virus in a zoo in Kerala, so your theory is not correct. Please do your research before posting BS."** (confidence 0.44)
   True `reasoned` → `noise`. **Why:** a clean debunk with a specific counterexample (the Kerala tiger), but written in a short, casual, combative register. The model reads the brevity and tone as low-effort `noise` and misses that it refutes a claim with evidence.

2. **"A lot of ISPs have started traffic shaping to handle peaks... YouTube is one of the first things they'll traffic shape. You can use this site to test if your ISP is traffic shaping..."** (confidence 0.41)
   True `reasoned` → `noise`. **Why:** a factual technical explanation plus a verification resource — textbook `reasoned`. But it has no conspiracy keywords and a flat, helpful tone, so it doesn't resemble the model's idea of an on-topic post and lands in `noise`.

3. **"Supposedly the Manhattan Project was conceived there... the photos suggest they're meticulous... the negative tabloid stories could have been planted... I'd call Bohemian Grove a conspiracy theorists' hot potato."** (confidence 0.41)
   True `reasoned` → `noise`. **Why:** a weighed, multi-consideration analysis, but rambling and hedged. The model has no representation of "chained reasoning"; it sees a long, meandering post and defaults to `noise`.

**The pattern:** the confused boundary is `reasoned → {noise, assertion}`, *never* the reverse — the model has **no concept of `reasoned` at all**. This isn't annotation inconsistency (these were labeled consistently per the §3 rules); it's a **training-data problem**: ~29 reasoned examples is too few, and they're stylistically diverse (pro-theory arguments, debunks, sourced posts, casual reasoning), so no surface pattern unifies them. Note the confidences (0.41–0.44) sit just above the 0.33 three-class chance floor — the model isn't confidently wrong, it's barely deciding at all.

### Sample classifications (fine-tuned, final)

Predicted labels and confidences are from the model's run on the test set:

| Post (truncated) | True | Predicted | Confidence | Correct? |
|------------------|------|-----------|-----------|----------|
| "A tiger was found to have the virus in a zoo in Kerala, so your theory is not correct." | reasoned | noise | 0.44 | ❌ |
| "A lot of ISPs have started traffic shaping... you can test if your ISP is traffic shaping." | reasoned | noise | 0.41 | ❌ |
| "Snowden was responsible for hacking the DNC." | assertion | noise | 0.43 | ❌ |
| "I never understood why people voted for their political platforms, but now I'm seeing the bigger picture." | noise | assertion | 0.43 | ❌ |
| _(one correctly-predicted `noise` post — see snippet)_ | noise | noise | _(run snippet)_ | ✅ |

Every confidence the model produced clusters in **0.41–0.51**, barely above the 0.33 chance floor for three classes — strong evidence it is essentially guessing rather than discriminating. For the required **correctly-predicted** example, the model gets 17 of 19 `noise` posts right (only because it over-predicts `noise`). To show one concretely, run this in the notebook and paste a row above:

```python
ci = np.where(ft_pred_ids == ft_true_ids)[0]
for i in ci[:5]:
    print(ID_TO_LABEL[ft_true_ids[i]], round(float(ft_probs[i][ft_pred_ids[i]]), 2),
          "|", test_df.iloc[i]["text"][:90])
```

A correct `noise → noise` call is reasonable for the right reason: those posts (jokes, one-line reactions, off-topic banter) genuinely make no claim, which is exactly what `noise` encodes.

> _To complete this table, paste the softmax confidence values from notebook Section 4's "wrong predictions" / sample output. Predicted labels above are exact (derived from the v2 confusion matrix)._

## 7. Reflection — Learned vs. Intended

**Intended:** a model that captures *argument structure* — whether a post reasons toward a claim with evidence.

**Actually learned:** essentially the **label prior**, not the semantics. With too little signal for the hard `reasoned`/`assertion` boundary, DistilBERT minimized loss by collapsing onto a single dominant class — `assertion` under plain training (v1), `noise` under class-weighting (v2). The fact that *which* class it collapsed onto flipped when I changed the loss weighting is the clearest evidence that it learned **frequency, not meaning**: it has no internal representation of "reasoned" (F1 = 0.00 in both runs).

**What it overfit to:** surface features — post length, casual/banter tokens, fragment structure — which is why genuinely argued posts written in a casual register got dumped into `noise`.

**What it missed:** the entire `reasoned` class, and more deeply, the concept that *structure can override tone*. The zero-shot LLM beats it precisely because it can reason about argument structure from world knowledge; a small model can't induce that from 29 examples.

## 8. Spec Reflection

- **One way the spec helped:** my [planning.md §5](planning.md) decision to evaluate on **macro-F1 and per-class metrics, not accuracy alone**, is the only reason the failure was visible. Accuracy (0.49–0.51) looks like a mediocre-but-working model; the per-class breakdown revealed an entire class was never predicted. Without that plan I'd have reported "51% accuracy" and missed the collapse entirely.
- **One way the implementation diverged + why:** the plan assumed a standard fine-tune would beat the baseline and defined success as "every class F1 ≥ 0.70." Neither held. I diverged by adding cost-sensitive class weighting and macro-F1 checkpoint selection (v2) — an unplanned step taken to fight the v1 collapse. It didn't rescue performance, but it converted a one-off result into a diagnosable pattern (two collapse directions), which is itself the finding.

## 9. AI Usage

1. **Annotation pre-labeling (disclosed).** I gave Claude the label definitions and asked it to draft a label for all 300 candidates; I then reviewed and corrected rows and dropped non-fitting ones. The `notes` column records the rationale per row. I overrode the AI on borderline `reasoned`/`assertion` and joke-vs-assertion cases (see §3).
2. **Failure analysis.** I gave Claude the two confusion matrices and `evaluation_results.json` and asked it to diagnose the failure. It identified the minority-class collapse and that class-weighting had *shifted* the collapse (assertion → noise) rather than fixing it; I verified this against the matrices and the reconstructed test split.
3. **Tooling/setup.** Claude wrote the data-prep scripts (`prep_ct.py`, the class-weighted trainer cell) and drafted the Groq `SYSTEM_PROMPT`, which I reviewed before running.

