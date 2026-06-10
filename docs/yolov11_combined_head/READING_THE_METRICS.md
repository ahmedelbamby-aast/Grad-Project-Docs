# How to Read the Metrics — a Field Guide

*You just finished a training run. You open `runs/combined_m_prod_250_na/` and
see a pile of numbers and PNGs. This document walks you through every one of
them — what it means, how to read it, whether it's good or bad, and the small bit
of math that proves why it measures what we claim. Written as a story you can
follow top to bottom.*

---

## 0. The one thing to understand first

Our model does **two different jobs at once**, and they need **two different
kinds of metric**:

1. **Find the people and say student/teacher** → this is *detection*, scored with
   **mAP** (mean Average Precision), the standard object‑detection yardstick.
2. **For a found person, say posture and gaze** → this is *classification*, scored
   with **accuracy** and **confusion matrices**.

If you only remember one rule: **mAP tells you "did we find them and get the
role right"; accuracy/confusion tells you "for the ones we found, did we get the
posture and gaze right."** Read them together — never one alone.

---

## 1. `results.csv` and `results.png` — the heartbeat

Every epoch writes one row to `results.csv` and redraws `results.png` (a grid of
little curves). Read them like an EKG: you want the **loss curves going down**
and the **metric curves going up**, both **smoothly**.

| Column | Plain meaning | Want |
|---|---|---|
| `train/box_loss`, `train/dfl_loss` | how wrong the boxes are | ↓ |
| `train/cls_loss` | how wrong role+posture+gaze are | ↓ |
| `val/*_loss` | same, on **unseen** images | ↓ (if it rises while train falls → **overfitting**) |
| `metrics/mAP50(B)` | detection quality @ easy overlap | ↑ |
| `metrics/mAP50-95(B)` | detection quality, strict | ↑ |
| `metrics/posture_acc` | posture correctness | ↑ |
| `metrics/gaze_acc` | gaze correctness | ↑ |
| `lr/pg0…` | learning rate schedule | warms up, then decays |

**How to read the shape, not just the last number.** The single most useful
habit: compare `train` loss vs `val` loss. If both go down together → healthy
learning. If `train` keeps dropping but `val` flattens or **rises** → the model is
memorising the training set (overfit) and you should trust an **earlier**
checkpoint. This is exactly why we keep `best.pt` (the epoch with the best
validation score), not just `last.pt`.

> **Real example from this project:** an early 250‑epoch run had a flat learning
> rate; its `val` precision *collapsed* in late epochs (over‑predicting boxes) and
> its best mAP stalled at 0.486. We switched to **cosine LR decay** so the rate
> glides smoothly to near‑zero, which keeps late epochs stable. The lesson is in
> the *curve shape*, not any single row.

---

## 2. `mAP50` and `mAP50‑95` — the detection score

### What it is
For each prediction we ask: does it overlap a real box of the **same role** by at
least some amount (IoU ≥ threshold)? Matches are **true positives (TP)**,
unmatched predictions are **false positives (FP)**, missed real boxes are **false
negatives (FN)**. Sweep the confidence threshold from high to low and you trace a
**precision–recall (PR) curve**; **Average Precision (AP)** is the **area under
that curve**.

```
Precision = TP / (TP + FP)        "of what I flagged, how much was right"
Recall    = TP / (TP + FN)        "of what existed, how much I caught"
AP_c(τ)   = ∫₀¹ precision_c(recall) d(recall)     at IoU threshold τ
mAP50     = average over roles of AP at IoU 0.50
mAP50-95  = average over roles of AP averaged over IoU 0.50, 0.55, …, 0.95
```

### How to read it
- **mAP50** is the forgiving score (boxes only need to roughly overlap). Good for
  "are we finding people at all."
- **mAP50‑95** is the strict score (boxes must overlap *tightly*). Lower is
  normal; it rewards precise localisation.
- A number near **0** means we're missing people or mis‑placing boxes; near **1**
  means near‑perfect detection. For a crowded classroom, **0.5–0.7 mAP50** is a
  solid working detector.

### Why it's the right proof
AP is a **single number that summarises the entire precision/recall trade‑off** —
you don't have to pick a confidence threshold to compare models, because AP
integrates over *all* of them. Averaging across IoU thresholds (50‑95) further
removes the arbitrariness of "how much overlap counts as correct." It is the
field‑standard, peer‑reviewed detection metric (COCO), which is exactly why we
report it.

### Why **role**, not the full label
We score detection at the **role** level (student/teacher) because role is the one
attribute **every** person has. If we instead scored the full
`student|sitting|down` combination, a correctly‑found **teacher** (who has *no*
posture/gaze) could never "match," and recall would crater for reasons that have
nothing to do with model quality. Scoring role keeps the detection metric
**well‑defined for every box**.

---

## 3. `posture_acc` and `gaze_acc` — the attribute scores

### What it is
Take every prediction that **correctly found a person** (IoU ≥ 0.5 with a real
box). For those, compare the predicted posture/gaze (the group's arg‑max) to the
truth. Accuracy is just the fraction correct:

```
acc =  (number of matched boxes whose predicted attribute == true attribute)
       ───────────────────────────────────────────────────────────────────
                       (number of matched boxes)
```

The printed line also shows `n=` — the **support**, i.e. how many boxes the score
is based on. Always read `n`: an accuracy on `n=4` means nothing.

### How to read it — and the trap
A **high single number can lie** when classes are imbalanced. In our data
`sitting` is ~77 % of postures and `down` is ~51 % of gazes, so a lazy model that
always guesses the majority already scores ~0.77 / ~0.51. So:
- **posture_acc ≈ 0.99** is genuinely good (well above the 0.77 floor).
- **gaze_acc ≈ 0.80** is good *overall*, but it's carried by the common classes.
  To see the truth for **rare** gazes (`up`, `backward`, `left`), you must look at
  the **per‑class numbers** (next sections), not this single average.

### Why it's a valid proof
Each attribute group is a **categorical variable**, and the soft‑max is its
**maximum‑likelihood model**; arg‑max is the Bayes‑optimal guess under 0/1 loss.
Accuracy is then the **empirical 0/1 risk** — an unbiased estimate of
"P(correct)" on the population we evaluate. That is the textbook way to score a
classifier.

### Note on masking (who is counted)
Teacher posture/gaze is **masked** (not trained, not scored — the product is not
interested in it), so these accuracies count **only boxes that truly have the
attribute**: students with a posture/gaze label (~10.7k of ~17.3k matched
boxes). The `n` is smaller than in the retired N/A-class runs, but the number is
*honest* — no free "n/a" points inflate it. (Why the N/A design was retired:
[DECISIONS_JOURNAL.md](DECISIONS_JOURNAL.md), Chapter 4.)

---

## 4. The confusion matrices — where the detail lives

A confusion matrix is a grid: **rows = the truth, columns = what the model said.**
The **diagonal is correct**; everything off the diagonal is a specific mistake
("true `left`, predicted `right`"). The **row‑normalised** version turns each row
into fractions, so the diagonal *is* the per‑class **recall**.

### 4.1 `role_confusion_matrix`, `posture_confusion_matrix`, `gaze_confusion_matrix`
These are the clean, honest views — one per group (2×2 role, 2×2 posture, 6×6
gaze; teachers are masked out). **This is where you catch the imbalance problem.** A bright
diagonal cell on `down` with a near‑empty `up` row tells you instantly: "the model
is great at the common gaze and blind to the rare one." Read the **normalised**
one for fairness across classes, the **count** one to see how much data backed it.

### 4.2 `grouped_all_classes_confusion_matrix` — and its grey cells
This is the "everything in one picture" matrix, **block‑diagonal**: a role block,
a posture block, a gaze block. The **off‑block cells are grey "N/A," not zero**,
*on purpose*. Here is the subtle point that trips people up:

> The grey cells do **not** mean "a student can't be sitting." They mean the model
> **physically cannot confuse a role with a posture**, because role and posture
> come from **separate soft‑maxes** — there is no shared decision to confuse. The
> three blocks are three independent confusion matrices stacked into one frame.

In other words, the grey N/A region is a **visual proof of the mutual‑exclusivity
design**: confusion can only ever happen *inside* a group, never across groups.

### 4.3 `per_class_recall.png` — the no‑nonsense summary
One bar per class, coloured by group, with the support `n=` on each bar. This is
the figure to screenshot when someone asks "how good is it, per class?" — it
shows all ten classes at once **without** the misleading off‑block zeros. Read
it as: tall bar = the model reliably catches that class; short bar over a tiny
`n` = a rare class the dataset barely teaches.

---

## 5. `role_conditioned_attributes.png` — does it *understand* the structure?

This is the figure that answers the deepest question: **has the model learned
that students have postures and teachers do not?** Two heatmaps, rows = the
**true role**, columns = the **predicted** posture / gaze:

- The **student row** should spread across the real attributes (sitting/standing,
  the six gazes) — it learned them.
- The **teacher row** is **untrained by design** (masked — the product is not
  interested in teacher posture/gaze), so whatever it shows is arbitrary and is
  **ignored at decode**. Do not read meaning into it; it is shown only so you can
  *see* that it is arbitrary rather than secretly biased.

`attribute_confidence_by_role.png` is the companion: it shows how *sure* the
model is per attribute, split by role — the student distributions are the ones
that matter; teacher confidence on posture/gaze is meaningless by construction.

---

## 6. The prediction montages — the human sanity check

`val_batch*_pred.jpg` (model) vs `val_batch*_labels.jpg` (truth) draw the boxes on
real validation images, each labelled with the **full composite**
(`student | sitting | look_right`). Numbers can hide problems that your eyes catch
in two seconds: boxes in the wrong place, duplicate boxes, a whole region missed.
Always glance at these — they are the reality check on every metric above.

---

## 7. A reading recipe (do this every time)

1. **`results.png`** — are train & val losses both going **down**? Is val
   tracking train (no overfit)? Are mAP/accuracy going **up**?
2. **`mAP50` / `mAP50‑95`** (last + best row) — is detection in a usable range,
   and did `best.pt` beat your previous run?
3. **`posture_acc` / `gaze_acc`** with their `n` — good overall?
4. **per‑group + `per_class_recall.png`** — is the overall accuracy *honest*, or
   is it the majority class carrying weak rare classes?
5. **`role_conditioned_attributes.png`** — is the **student row** structured
   (and remember the teacher row is masked/arbitrary by design)?
6. **`val_batch*_pred.jpg`** — do the pictures actually look right?

If 1–6 are green, you have a model you can trust **and** explain.

---

## 8. Cheat‑sheet of "good vs worrying"

| Signal | Good | Worrying | Likely cause / fix |
|---|---|---|---|
| val loss vs train loss | fall together | val rises, train falls | overfit → use `best.pt`, more aug, fewer epochs |
| mAP curve | smooth ↑ | spiky / collapses late | LR too high/flat → cosine LR |
| precision suddenly low, recall high | — | yes | model over‑predicting boxes (late‑epoch instability) |
| posture/gaze acc high but rare‑class recall ~0 | expected on rare classes | misread as "great" | imbalance → read per‑class, not the average; class weights + focal + direction‑aware flip fight it |
| `look_left`/`look_right` recall near zero | — | yes | mirrored‑flip label corruption → must use the direction‑aware flip (journal Ch. 6) |
| `n=` tiny on a class | — | yes | not enough data for that class — collect more |

---

*Companion document: `ARCHITECTURE.md` (how the model is built). Together they let
you both **trust** the numbers and **explain** them.*
