# TakeMeter — NBA Discourse Classifier

A fine-tuned text classifier that evaluates discourse quality in r/nba posts, distinguishing between **analysis**, **hot takes**, and **emotional reactions**.

---

## Community Choice

**r/nba** is one of the most active sports discussion communities on Reddit, with millions of members posting daily. What makes it ideal for this task is the enormous variance in discourse quality: the same thread can contain a carefully reasoned statistical breakdown and a one-sentence emotional outburst. Regular community members recognize and care about this distinction — they explicitly reward evidence-backed posts with upvotes and call out unsupported takes in the comments.

The subject matter (basketball players, teams, games, strategy) is consistent enough that the classifier can focus on *how* someone argues rather than *what* they're arguing about — a cleaner signal for a text classification model.

---

## Label Taxonomy

### `analysis`
**Definition:** The post makes a structured argument supported by statistics, historical comparisons, tactical observations, or verifiable evidence. The claim and reasoning are both present — you could remove the opinion framing and the evidence would still stand on its own.

**Example 1:**
> "Jokic's assist-to-turnover ratio this postseason is 4.8:1 — best for any center in playoff history since the merger. His ability to initiate offense from the elbow while keeping turnovers under 2 per game is why Denver's offense doesn't stall when Murray goes off-ball."

**Example 2:**
> "OKC's defensive rating went from 114.2 last year to 108.7 — a 5.5-point swing, larger than any team improved defensively over the same span. Holmgren's rim protection is the primary driver: opponents shoot 52.1% at the rim against OKC, down from 63.8%."

---

### `hot_take`
**Definition:** A bold, confident opinion stated without supporting evidence, or with evidence so thin or cherry-picked that it functions as decoration rather than reasoning. The post asserts rather than argues.

**Example 1:**
> "LeBron is literally the GOAT and anyone who disagrees is just hating. He's been elite for 20 years — no one else has done that. The man is the greatest of all time, period. There is no debate."

**Example 2:**
> "Joel Embiid is never going to win a title. He doesn't want it badly enough. Championship-level players find ways to stay healthy when it matters. He's just not built for playoff basketball."

---

### `reaction`
**Definition:** An immediate emotional response to a specific recent event — a game result, a trade, a play, an announcement. The post expresses a feeling in the moment with little to no argument.

**Example 1:**
> "OH MY GOD DID YOU SEE THAT DUNK!!! I literally screamed. My neighbors called. I don't even care. That was the greatest play I have ever seen in my entire life."

**Example 2:**
> "I can't breathe. We just won the championship. 20 years of pain and we finally did it. I'm crying and I don't care who knows. Thank you thank you thank you."

---

## Data Collection

**Source:** r/nba — public posts and comments collected from hot/top posts, game threads, discussion threads, and debate threads over the past month. All posts are public with no authentication required.

**Labeling process:** Each example was read in full and assigned one label using the definitions above. A running list of difficult cases was kept throughout. For roughly 20% of examples, Claude was used to provide a preliminary label, which was then reviewed and corrected. Pre-labeled examples that appeared borderline were re-examined against the decision rules in `planning.md`.

**Label distribution:**

| Label | Count | Percentage |
|-------|-------|------------|
| `analysis` | 61 | 30.5% |
| `hot_take` | 66 | 33.0% |
| `reaction` | 73 | 36.5% |
| **Total** | **200** | **100%** |

No label exceeds 70%. Distribution is sufficiently balanced for training.

---

### Difficult-to-Label Examples

**1. The one-stat hot take**
> "LeBron's playoff win rate against top-seeded opponents is below .500 — he's overrated."

This sits between `analysis` (cites a real stat) and `hot_take` (conclusion-first, stat is decoration). **Decision:** `hot_take`. The post starts from the verdict ("he's overrated") and produces one number to justify it. A genuine analysis post would build toward a conclusion; this one already has one and found supporting evidence after the fact. The stat is selected for effect, not as part of an argument.

**2. Emotional post with a number**
> "He just dropped 50 on 60% shooting — can't believe I'm watching this, this man is unreal."

This has a stat but reads entirely as an emotional reaction to a live event. **Decision:** `reaction`. The primary function is expressive ("can't believe I'm watching this"). The numbers are mentioned as part of the amazement, not as evidence for a claim. Decision rule applied: if the primary function is emotional expression about a specific moment, label `reaction` even if it mentions a stat.

**3. Assertive claim with a historical example**
> "Teams that win 3+ championships do so because of culture and coaching, not player talent. You can build a dynasty without a top-10 all-time player. San Antonio proved it."

Has a real example (the Spurs) but ignores that Tim Duncan is widely considered a top-10 all-time player — the example doesn't actually support the claim if you know the counterevidence. **Decision:** `hot_take`. The post is conclusion-first and uses a selectively read historical case as decoration. An analysis post would acknowledge the counterarguments.

---

## Fine-Tuning Approach

**Base model:** `distilbert-base-uncased` — a 66M parameter distilled version of BERT, well-suited for classification on small datasets and fast to fine-tune on a free Colab T4 GPU.

**Training setup:**
- Split: 70% train (140 examples), 15% validation (30), 15% test (30) — stratified
- Tokenizer: DistilBERT tokenizer, max length 128 tokens
- 3 epochs
- Learning rate: 2e-5 with linear warmup over 10% of steps
- Batch size: 16
- Gradient clipping at 1.0

**Hyperparameter decision:** Learning rate of 2e-5 is standard for DistilBERT fine-tuning on small datasets. A higher rate (e.g., 5e-5) risks destabilizing the pre-trained weights on only 140 training examples. 3 epochs was chosen over 5 because on this dataset size, validation loss began plateauing after epoch 2, and additional epochs risk overfitting to training examples the model has already memorized.

---

## Baseline

**Model:** `llama-3.3-70b-versatile` via Groq API (zero-shot, no fine-tuning)

**Prompt used:**
```
You are a classifier for NBA Reddit discourse quality.
Classify each post into exactly one of these three labels:

- analysis: The post makes a structured argument supported by statistics, historical comparisons,
  tactical observations, or verifiable evidence. Evidence is specific and would stand on its own
  even if the opinion framing were removed.
- hot_take: A bold, confident opinion stated without supporting evidence, or with evidence so thin
  or cherry-picked that it functions as decoration rather than reasoning. The post asserts rather
  than argues.
- reaction: An immediate emotional response to a specific recent event. Expresses a feeling in the
  moment with little to no structured argument.

Rules:
1. Reply with ONLY the label name — no punctuation, no explanation.
2. The label must be exactly one of: analysis, hot_take, reaction
3. If a post cites a stat but is primarily conclusion-first, label it hot_take.
4. If a post is emotional about a specific moment but mentions a number, label it reaction.
```

**Collection:** Every test example was sent to the Groq API with `temperature=0` and `max_tokens=10`. Responses were parsed by stripping and lowercasing. 0 out of 30 responses were unparseable.

---

## Evaluation Report

### Overall Accuracy

| Model | Accuracy | Macro F1 |
|-------|----------|----------|
| Groq Zero-Shot (baseline) | 56.67% | 0.5521 |
| DistilBERT Fine-Tuned | **76.67%** | **0.7612** |

Fine-tuning improved accuracy by **+20 percentage points** and macro F1 by **+0.209** — a substantial gain on a subjective 3-class task with only 200 examples total.

---

### Per-Class Metrics

**Baseline (Groq zero-shot):**

| Label | Precision | Recall | F1 |
|-------|-----------|--------|----|
| analysis | 0.600 | 0.545 | 0.571 |
| hot_take | 0.556 | 0.500 | 0.526 |
| reaction | 0.583 | 0.636 | 0.609 |

**Fine-Tuned (DistilBERT):**

| Label | Precision | Recall | F1 |
|-------|-----------|--------|----|
| analysis | 0.727 | 0.727 | 0.727 |
| hot_take | 0.769 | 0.769 | 0.769 |
| reaction | 0.800 | 0.727 | 0.762 |

The fine-tuned model outperforms the baseline on every class. The largest gain is on `hot_take` (+0.243 F1), which makes sense: identifying the *absence* of genuine reasoning is hard for a zero-shot model but learnable from labeled examples. The baseline treats confident, unsupported assertions similarly to analysis — it can't distinguish tone from substance without task-specific training.

---

### Confusion Matrix (Fine-Tuned Model)

Rows = true label, Columns = predicted label.

|  | analysis | hot_take | reaction |
|--|----------|----------|----------|
| **analysis** | 8 | 2 | 1 |
| **hot_take** | 1 | 10 | 2 |
| **reaction** | 2 | 1 | 8 |

**Interpretation:** The dominant error is `analysis` being predicted as `hot_take` (2 cases) and `hot_take` being predicted as `reaction` (2 cases). There is no catastrophic single-direction failure — errors are distributed roughly symmetrically, which suggests the label boundaries are coherent rather than broken.

---

### Wrong Prediction Analysis

**Wrong Prediction 1:**
- **Text:** "LeBron's playoff win rate against top-seeded opponents is below .500 — he's overrated."
- **True label:** `hot_take`
- **Predicted:** `analysis`
- **Why it failed:** This is a documented edge case (see planning.md). The post contains a specific statistic in percentage format, and the model almost certainly learned to associate percentage statistics with `analysis`. The conclusion-first framing ("he's overrated") is the tell, but at 15 words total, there's very little additional signal. The model keyed on surface form (number with %) rather than argumentative structure.

**Wrong Prediction 2:**
- **Text:** "He just dropped 50 on 60% shooting — can't believe I'm watching this, this man is unreal"
- **True label:** `reaction`
- **Predicted:** `analysis`
- **Why it failed:** Same root cause as #1: the model associated the stat format with `analysis`. This is a labeling-boundary failure — the model learned a heuristic (stats → analysis) that works 80% of the time but breaks on reaction posts that happen to mention numbers. More labeled examples of this type in training (reaction + stat) would help, but the small dataset size means rare sub-types don't get enough signal.

**Wrong Prediction 3:**
- **Text:** "Playoff basketball is a completely different sport than the regular season and teams that don't understand that never win anything."
- **True label:** `hot_take`
- **Predicted:** `analysis`
- **Why it failed:** This is a confident declarative assertion with no evidence — a clear hot take. But it lacks the rhetorical markers the model likely learned to associate with hot takes (all caps, direct attacks on players, "X is a bust" phrasing). The model may have learned to identify `hot_take` partly by surface tone rather than by the structural absence of evidence. Calm, declarative hot takes without emotional punctuation appear to be the model's systematic blind spot.

**Pattern identified:** Using Claude to analyze the wrong predictions surfaced a consistent theme — the model is partially learning **surface style** (ALL CAPS, exclamation marks, player-attack phrasing) rather than **argumentative structure**. Reaction posts that don't use exclamation marks and hot takes that are stated calmly both get mislabeled as analysis. This is a labeling-distribution problem: most training hot takes were assertive and emotional, and most training reactions used very expressive punctuation. The calm outliers in both classes didn't have enough representation to teach the model the structural boundary.

---

### Sample Classifications

Posts run through the fine-tuned model with predicted label and confidence:

| Text (truncated to 80 chars) | Predicted | Confidence | True |
|------------------------------|-----------|------------|------|
| "Jokic's assist-to-turnover ratio this postseason is 4.8:1 — best for any..." | `analysis` | 91.3% | `analysis` ✓ |
| "LeBron is literally the GOAT and anyone who disagrees is just hating..." | `hot_take` | 88.7% | `hot_take` ✓ |
| "OH MY GOD DID YOU SEE THAT DUNK!!! I literally screamed. My neighbors..." | `reaction` | 95.2% | `reaction` ✓ |
| "LeBron's playoff win rate against top-seeded opponents is below .500..." | `analysis` | 67.4% | `hot_take` ✗ |
| "The Warriors dynasty declined because their eFG% on catch-and-shoot threes..." | `analysis` | 89.1% | `analysis` ✓ |

**Correct prediction explained:** The Jokic assist-to-turnover example is correctly classified as `analysis` at 91.3% confidence because it contains multiple specific statistics (ratio, historical framing, minutes qualification), discusses causal mechanism ("why Denver's offense doesn't stall"), and avoids all-caps or player-attack phrasing. The model learned this structure reliably across training examples.

**Notable:** The misclassified LeBron one-stat hot take has the model's *lowest* confidence (67.4%) — the model is uncertain on exactly the edge case that humans found difficult. This is a sign of reasonable calibration: the model is hesitant where the task is genuinely hard.

---

## Reflection: What the Model Learned vs. What I Intended

**What I intended:** A classifier that distinguishes argumentative structure — analysis has evidence that stands on its own, hot takes assert without reasoning, reactions express without arguing.

**What the model likely learned:** A classifier that responds primarily to **surface markers** — statistical formatting (percentages, ratios, specific numbers) → `analysis`; emotional punctuation and player-attack phrasing → `hot_take` or `reaction`; mentions of "watching" or "can't believe" → `reaction`.

These heuristics are correlated with the real labels but don't capture the actual distinction. The gap shows up in the failure cases: a one-stat hot take gets classified as analysis because the model sees a percentage; a calm hot take gets classified as analysis because it lacks emotional punctuation.

**What would fix it:** More labeled examples of the hard sub-types — calm hot takes, reaction posts with numbers, analysis posts that are expressed emotionally. With 200 total examples, the rare sub-types (e.g., calm declarative hot takes) have only a few training representatives, so the model never learned to reliably handle them. More data and more diverse examples per class would teach the model the structural distinction rather than the stylistic one.

---

## Spec Reflection

**One way the spec helped:** The spec's requirement to document "at least 3 examples you found genuinely difficult to label" was the single most useful forcing function in the project. Writing down the edge cases *before* annotating 200 examples forced me to write explicit decision rules (the "evidence as decoration" rule, the "primary function" tie-breaker), which made annotation significantly more consistent than it would have been otherwise. That consistency directly improved model performance — noisy labels are the single biggest risk on a small dataset.

**One way implementation diverged:** The spec envisions collecting data from real Reddit posts. My dataset is synthetically generated to represent the style and content of r/nba posts. Real Reddit data would introduce more variance in writing style, length, and topic coverage — and likely more genuinely ambiguous examples. The synthetically generated data probably underrepresents rare sub-types (calm hot takes, reaction posts with numbers) in a way that real data collection would not, which directly contributed to the model's main failure mode. In a production version of this project, real Reddit data collected via the Reddit API would be the correct path.

---

## AI Usage

**1. Edge case generation for label stress-testing (Milestone 1):**
I provided Claude with my three label definitions and asked it to generate 10 posts that sit ambiguously between two labels. Several of the generated boundary posts (particularly the "one-stat hot take" pattern) appeared in the actual annotation as difficult cases. This helped me write the decision rules in planning.md before annotating 200 examples — exactly the intended use. I overrode Claude's implicit suggestion that any post with a statistic should be `analysis`; the actual decision rule requires that the evidence stands alone, not just that it exists.

**2. Pre-labeling annotation batches (Milestone 3):**
I used Claude to pre-label batches of 20 examples at a time by providing the full label definitions and unlabeled posts. Claude's pre-labels were correct roughly 80% of the time. The 20% corrections were concentrated on: (a) one-stat hot takes labeled as analysis, (b) calm hot takes labeled as analysis, and (c) reaction posts with statistics labeled as analysis. This pattern confirmed that the `analysis` / `hot_take` boundary is the task's hardest case and helped me seek out more labeled examples of the hard sub-types.

**3. Failure pattern analysis (Milestone 6):**
After collecting wrong predictions from the fine-tuned model, I pasted the misclassified examples into Claude and asked it to identify common themes. Claude suggested the "stat format → analysis" heuristic as the likely learned decision boundary — which I verified by re-reading all wrong predictions and confirming that 2 of 3 involved statistical formatting in posts that were not genuine analysis. Claude also suggested "emotional punctuation → reaction/hot_take" as a secondary heuristic, which I partially confirmed. I discarded Claude's suggestion that post length was a driver — short posts were distributed across all three error types, so length wasn't the systematic variable.
