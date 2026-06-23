# TakeMeter — Planning Doc

**Community:** r/ConspiracyTheory · **Task:** 3-class discourse-quality classifier
**Author:** Nish (nraju2024@fau.edu) · **Course:** AI201, Project 3

> Working notes written before annotation. README.md is the polished final report; this file holds the reasoning behind each decision and my hard annotation calls.

---

## 1. Community

I chose **r/ConspiracyTheory**. It's a strong fit for a discourse-quality classifier because the *quality* of posts varies enormously while the *topic* stays constant — the same thread can hold a carefully argued theory with sources, a one-line "wake up sheeple" assertion, and an off-topic joke. That spread is exactly what makes "what is a good take here" a real, non-trivial distinction.

Crucially, my labels measure **how a take is argued, not whether it is true.** A classifier cannot (and should not) adjudicate whether a conspiracy claim is factually correct, and the project is about discourse quality, so a post that argues a far-fetched theory *with a reasoning chain* is `reasoned`, while a true-but-unsupported one-liner is an `assertion`. This keeps the labels objective and annotatable.

## 2. Labels (3)

| Label | One-sentence definition |
|-------|-------------------------|
| `reasoned` | The post argues a position about a claim — supporting **or** debunking it — using a reasoning chain, specific evidence, sources, or documented detail; the case would still stand if you stripped the emotional framing. |
| `assertion` | The post states a theory/opinion or asks a leading question with **no** supporting evidence or reasoning — it asserts rather than argues. |
| `noise` | Off-topic remarks, jokes, banter, one-line emotional reactions, or meta-discussion that neither makes nor evaluates a substantive claim. |

**Decision boundary (state-able in two questions):**
1. Does the post make or evaluate a substantive claim? **No → `noise`.**
2. If yes, is it backed by reasoning/evidence? **Yes → `reasoned`, No → `assertion`.**

### `reasoned` — examples (real posts from the dataset)
1. _"The Fort Mac wildfire was engineered… Oil is still being produced at all the production facilities, just slower. Once the public sees the importance of Fort Mac, they will use the influx to rebuild… and will now have gathered enough public sympathy to encourage the building of the pipelines."_ (builds a causal chain)
2. _"The big problem with this is that their surveillance isn't working to prevent these things. If they wanted people to accept the surveillance, they'd need to show it actually worked, rather than repeatedly showing that it isn't helping."_ (a reasoned debunk)

### `assertion` — examples
1. _"Recently heard the theory that Avril Lavigne is dead and was replaced by Melissa Vandella. From what I've seen it looks pretty real… thoughts?"_ (claim + "thoughts?", zero evidence)
2. _"Another false flag effort to trigger a response against Iran."_ (bare assertion)

### `noise` — examples
1. _"Thats like saying oatmeal is soup because you add water but its not, its a cereal."_ (off-topic banter)
2. _"Chris Brown = Male Hannah Montana. It's the best of both worlds."_ (joke)

## 3. Hard Edge Cases

**Hardest boundary: `reasoned` vs. `assertion`** — a post that *sounds* substantive (names an event, drops a detail) but offers no actual reasoning.

- **Bare link posts:** a title + URL with no explanation. **Rule:** a link with the poster explaining *what it shows and why it supports the claim* → `reasoned`; a link dumped with no argument → `assertion` (or `noise` if no claim at all).
- **Personal testimony:** _"I'm a New Yorker who lost many friends on 9/11…"_ — first-hand but unverifiable. **Rule:** specific first-hand testimony that bears on the claim → `reasoned` (it's reasoning from evidence, even if weak); a vague "I just feel" → `assertion`.
- **Skeptical one-liners:** _"no evidence lol."_ **Rule:** gives a *reason* for the skepticism → `reasoned`; pure mockery → `noise`.

> 📌 **Living section** — log ≥3 genuinely difficult *real* cases here during annotation (the post, the labels it could fit, what I decided). Required at the Milestone 3 checkpoint.
1. _(fill during annotation)_
2. _(fill during annotation)_
3. _(fill during annotation)_

## 4. Data Collection Plan

- **Source:** a pre-collected public export of r/ConspiracyTheory posts and comments (`reddit_ct.csv`, ~1,197 rows, 974 with body text). Public data only.
- **Prep:** `prep_ct.py` cleans the export (strips URLs, drops bots/deleted/junk, dedupes) and writes ~300 candidates to `ct_unlabeled.csv`.
- **Method:** I read and hand-label each candidate `reasoned` / `assertion` / `noise`, keep the good ones, and save the result as `dataset.csv`. I delete rows that don't fit any label rather than forcing them (target ≥90% clean-labelable).
- **Target:** ≥200 labeled, aiming ~20–40% per label; no label above 70%.
- **If a label is underrepresented:** `reasoned` is the rarest type in this subreddit, so if it falls below ~20% I'll pull more from the longer-body rows in the export (effortposts tend to be longer) until it clears the floor.

## 5. Evaluation Metrics

- **Overall accuracy** for both models — headline, but insufficient alone (a model could win by always predicting the majority class).
- **Per-class precision / recall / F1** — the interesting, consequential error is `reasoned`↔`assertion`; per-class F1 shows whether the model learned the *hard* boundary or just the easy `noise` one.
- **Confusion matrix** — to read the *direction* of errors (is it calling real reasoning a bare assertion, or vice versa?).
- **Macro-F1** is the single number I trust most, since I weight all three classes equally regardless of frequency.

## 6. Definition of Success

- **Minimum bar:** fine-tuned macro-F1 meaningfully beats the zero-shot Groq baseline, and no class has F1 ≈ 0.
- **"Good enough" for a real moderation/sorting tool:** every class F1 ≥ 0.70 **and** the `reasoned`↔`assertion` confusion is the minority of errors, not the majority.
- Both criteria are objective — readable straight off the per-class report and confusion matrix.

---

## AI Tool Plan

- **Label stress-testing:** Before annotating, give Claude the label definitions + edge-case rules and ask for 5–10 posts on the `reasoned`/`assertion` boundary. If I can't classify them cleanly, tighten the definitions first.
- **Annotation assistance:** _Decision:_ ⟨choose⟩ — (a) label all manually, or (b) LLM pre-labels a batch, then I review/correct **every** row. If (b), add a `prelabeled` column to track AI-suggested rows and disclose it in the README AI-usage section.
- **Failure analysis:** After fine-tuning, paste the misclassified examples into Claude, ask for patterns (post length, sarcasm, link-only posts, the `reasoned`/`assertion` pair), then verify each pattern myself before writing it up.

> Update this doc before starting any stretch feature.
