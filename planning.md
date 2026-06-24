# TakeMeter: Planning Document
**Community:** r/nba  
**Task:** Classify discourse quality in NBA Reddit posts/comments

---

## 1. Community Choice

I chose r/nba because it is one of the most active sports discussion communities on Reddit, with millions of members posting daily reactions, arguments, and breakdowns about basketball. Discourse quality varies enormously: the same thread might contain a carefully reasoned statistical argument and a one-sentence emotional outburst. That variance makes it an ideal classification target — there are real, recognizable distinctions that regular community members care about and enforce through upvotes, flairs, and explicit community norms ("we don't want hot takes without receipts here").

The NBA context is also helpful because the subject matter is consistent enough (basketball players, teams, games, strategy) that the classifier can focus on *how* someone argues rather than *what* they're arguing about.

---

## 2. Label Taxonomy

### Label 1: `analysis`
**Definition:** The post makes a structured argument supported by statistics, historical comparisons, tactical observations, or verifiable evidence. The claim and the reasoning are both present; you could remove the opinion framing and the evidence would still stand on its own.

**Example 1:**
> "Jokic's assist-to-turnover ratio this postseason is 4.8:1, which is the best for any center in playoff history since the merger. His ability to initiate offense from the elbow while keeping his turnover rate under 2 per game is why Denver's offense doesn't stall when Murray goes off-ball."

**Example 2:**
> "The Warriors dynasty declined because their eFG% on catch-and-shoot threes dropped from 58% in 2017 to 51% in 2022 — largely because teams started switching onto Curry earlier, forcing him to create off the dribble more. The three-peat model required specific defensive laziness that stopped existing."

---

### Label 2: `hot_take`
**Definition:** A bold, confident opinion stated without supporting evidence, or with evidence so thin or cherry-picked that it functions more as decoration than reasoning. The post asserts rather than argues. The claim might be true, but the post doesn't do the work to show it.

**Example 1:**
> "LeBron is overrated. He's never been the best player on his championship teams. Kyrie won the 2016 Finals and Dwyane Wade won 2012. LeBron is a stat-padder."

**Example 2:**
> "Tatum will never be a true #1 option. He disappears in big moments. You can just tell he doesn't have that killer instinct. Some guys have it, some don't, and Tatum doesn't."

---

### Label 3: `reaction`
**Definition:** An immediate emotional response to a specific recent event — a game result, a trade, a play, an announcement. The post expresses a feeling in the moment with little to no argument. Reactions can be positive or negative but are primarily expressive rather than argumentative.

**Example 1:**
> "WHAT WAS THAT DUNK??? Wembanyama just broke the internet. I can't believe what I just watched. This kid is going to be the best player ever."

**Example 2:**
> "I'm done. I actually cannot believe we just blew a 20 point lead. Fire everyone. This franchise is cursed and always will be."

---

## 3. Hard Edge Cases

### Anticipated ambiguous case: "one-stat hot take"
A post like: *"LeBron's playoff win rate against top-seeded opponents is below .500 — he's overrated"* sits between `analysis` (cites a specific stat) and `hot_take` (cherry-picked, conclusion-first reasoning).

**Decision rule:** If the evidence is specific and verifiable AND the post would still be informative if you removed the opinion framing, label it `analysis`. If the stat exists only to *support* an already-held conclusion (the post starts from the take and works backward to find a number), label it `hot_take`. The LeBron example above is `hot_take` — the stat is selected to prove a predetermined verdict, not to build toward one.

### Second anticipated case: "reaction with a stat"
A post like: *"He just dropped 50 on 60% shooting — can't believe I'm watching this, he's unreal"* mixes reaction energy with a real number. **Decision rule:** If the primary function of the post is emotional expression about a specific moment, label it `reaction` even if it mentions a stat. If the post transitions from the immediate event into structured reasoning about what it means historically or tactically, label it `analysis`.

### Difficult cases encountered during annotation:
*(to be filled in during Milestone 3 — at least 3 documented)*

---

## 4. Data Collection Plan

**Source:** Reddit r/nba — public posts and top-level comments from:
- Hot/top posts in the last month
- Game threads (good source of reactions)
- Discussion threads and "film room" posts (good source of analysis)
- Debate threads (good source of hot takes)

**Target distribution (per label):**
- `analysis`: ~70 examples (35%)
- `hot_take`: ~80 examples (40%)
- `reaction`: ~50 examples (25%)

This is roughly balanced while acknowledging that reaction posts are slightly less common in dedicated discussion threads. No single label exceeds 70%.

**If a label is underrepresented:** If after 150 examples any label is below 20%, I'll specifically seek out posts from thread types that tend to produce that label (game threads for reactions, debate/discussion threads for hot takes, film room or stat posts for analysis).

**Storage format:** Single CSV file with columns `text`, `label`, `notes`. The notebook handles the 70/15/15 train/val/test split automatically.

---

## 5. Evaluation Metrics

**Primary metric: per-class F1 score**  
Accuracy alone is misleading here because even a mild class imbalance (e.g., 40% hot_take) means a naive majority-class predictor scores 40% — which sounds decent. F1 per class captures both precision and recall and reveals whether the model is learning all three distinctions or just the dominant one.

**Secondary metric: confusion matrix**  
The confusion matrix tells me *which* label pairs the model is confusing, not just that it's making errors. This is the most diagnostically useful output — a model that confuses `analysis` and `hot_take` has a different failure mode than one that confuses `reaction` and `hot_take`.

**Also reported:** Overall accuracy for direct comparison with the Groq zero-shot baseline.

**Why not just accuracy?** On a 3-class task with 40/35/25 distribution, a model that only predicts `hot_take` gets ~40% accuracy. F1 forces the model to actually learn each class. Macro-averaged F1 penalizes models that ignore minority classes.

---

## 6. Definition of Success

**Minimum acceptable:** Fine-tuned model achieves macro-averaged F1 ≥ 0.65 on the test set, and meaningfully outperforms the Groq zero-shot baseline on at least 2 of 3 per-class F1 scores.

**Good enough for deployment:** Macro F1 ≥ 0.72, with no single class F1 below 0.60. At this level, the classifier is useful as a content moderation aid or community quality signal — it's right often enough that false positives/negatives don't undermine trust.

**What would make me suspicious:** Accuracy above 95% on a subjective 3-class task with 200 examples would indicate test leakage or labels that are trivially easy (e.g., reaction posts always contain "!!!" and analysis posts always contain numbers — the model is keying on surface features, not discourse quality).

---

## 7. AI Tool Plan

### Label stress-testing
Before finalizing labels, I prompted Claude with my three label definitions and asked it to generate 10 boundary posts — posts that sit ambiguously between two labels. Several generated posts between `analysis` and `hot_take` confirmed that the "evidence as decoration" rule was necessary (described in Section 3). Two posts between `reaction` and `hot_take` led me to add the "primary function" tie-breaker for posts that express emotion but also assert a claim.

### Annotation assistance
I used Claude to pre-label batches of 20 examples at a time by providing the full label definitions and asking for one label per post with a one-sentence justification. I reviewed every pre-label and corrected roughly 15–20% of them, primarily cases where Claude labeled "one-stat hot takes" as `analysis`. All pre-labels were treated as drafts, not ground truth.

### Failure analysis
After collecting wrong predictions from the fine-tuned model, I will paste the misclassified examples into Claude and ask it to identify systematic patterns (e.g., consistent confusion of sarcastic posts, short posts, posts that use stats rhetorically). I will verify each suggested pattern by re-reading the examples myself before including it in the evaluation report.

---

## Stretch Features Planned
- [ ] Deployed interface (Gradio or simple HTML)
- [ ] Error pattern analysis (systematic, beyond listing 3 wrong predictions)
- [ ] Inter-annotator reliability (if a second annotator is available)

*This document will be updated before beginning any stretch feature.*
