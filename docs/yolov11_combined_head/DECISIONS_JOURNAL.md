# The COMBINED Head — Decisions Journal (the story of every choice)

> **What this file is.** A living diary of the mini-project ("head project"):
> every decision we made, whether we **accepted** or **refused** it, and the
> exact reason — written as a story in simple English, with the tiny technical
> details spelled out. Newest chapters at the bottom. **Rule: every future
> decision gets a chapter here, the same day it is made.**
>
> Companion docs: [ARCHITECTURE.md](ARCHITECTURE.md) (how the model works now)
> and [READING_THE_METRICS.md](READING_THE_METRICS.md) (how to read the figures).

---

## Chapter 1 — The problem, and the design we picked

We had a Roboflow dataset with 10 flat classes (`student`, `teacher`,
`student_sitting`, `student_standing`, and six `student_looking_*` directions),
where every student is drawn as **three nearly identical boxes** — one per
attribute. We wanted ONE model that looks at a person once and answers three
questions at the same time: *who are they* (role), *how do they sit* (posture),
*where do they look* (gaze).

**Decision: the grouped multi-attribute head ("Option B"). ACCEPTED.**
Instead of inventing a new network, we keep a 100% stock YOLO detection head and
simply *re-read* its class outputs as three groups, each squashed by its own
softmax: `role = logits[0:2]`, `posture = logits[2:4]`, `gaze = logits[4:10]`.
A softmax is a probability list that sums to 1, so taking the biggest value
always returns **exactly one** answer per group. Mutual exclusivity ("a person
cannot be sitting AND standing") comes from mathematics, not from cleanup code.

**Decision: five binary yes/no groups (the original prototype idea). REFUSED.**
Three binary gaze switches (`fwd/back`, `left/right`, `up/down`) can all turn on
at once and answer "forward + left + up" — which breaks "exactly one gaze".
Gaze must be ONE 6-way softmax.

**Why this matters for deployment:** because we never touched the network graph,
the exported ONNX/TensorRT engine is a plain YOLO engine — no custom layers, no
plugins. All the cleverness is a few `argmax` calls after the engine returns.

## Chapter 2 — Smuggling three labels inside one number

YOLO's data loader only carries ONE class id per box. We needed three. So we
pack the triple into a single integer — a **composite id**:

```
composite = role*21 + posture_code*7 + gaze_code      (range 0..41)
posture_code: 0=sitting 1=standing 2=UNKNOWN
gaze_code   : 0=fwd 1=back 2=left 3=right 4=up 5=down 6=UNKNOWN
```

The dataset *pretends* to have 42 classes so the loader's checks pass; our
trainer forces the real head back to 10 logits; the loss unpacks the composite
with small lookup tables. **ACCEPTED** because the stock loader, mosaic and all
augmentations keep working with zero custom data code. The three overlapping
boxes per student are IoU-merged (≥ 0.8) into one box during conversion.

**Decision: anchor assignment keyed on ROLE. ACCEPTED.** YOLO's assigner
(which decides which prediction cell is responsible for which ground-truth box)
needs one class id per box. Role is the only attribute every person has, so
assignment happens "as if" this were a 2-class detector; posture and gaze ride
along in the loss only.

## Chapter 3 — Nano or medium?

**Decision: `yolo11m` (medium, ~20M params) with COCO-pretrained backbone/neck.
ACCEPTED. Nano REFUSED.** Six gaze directions in a crowded classroom is a
fine-grained problem; nano underfit it in early tests. Medium still runs in
real time on the production RTX 5090, and the COCO transfer (643/649 weight
tensors) means the model starts out already knowing what humans look like.

## Chapter 4 — The teacher problem: three attempts to say "not applicable"

A teacher has **no posture or gaze label** in this dataset — the attributes
simply do not apply. We tried three designs over the project's life:

**Attempt 1 — masking (the original). ACCEPTED at first.** When a box's posture
or gaze is UNKNOWN, that group's loss is skipped (`target = -1` → no gradient).
Simple, but the model never *learns* anything about teachers' attributes, so its
posture/gaze output for a teacher is arbitrary noise we must remember to ignore.

**Attempt 2 — explicit N/A classes. ACCEPTED, then RETIRED by evidence.** We
grew the head to 12 logits (`posture_na`, `gaze_na`) so a teacher trains toward
"posture = n/a, gaze = n/a" and the rule lives inside the weights. It sounded
elegant. Then the 201-epoch production run (`combined_m_prod_250_na`) told us
the truth:

- the two N/A classes had ~6,000 validation samples each and recall ~0.88 —
  they are **free wins** (mostly "is it a teacher?" in disguise), and they
  swallowed a huge share of the label mass and gradient;
- meanwhile the real minority classes starved: `standing` 0.30 recall,
  `look_backward` 0.08, `look_left` 0.002, `look_up` 0.00;
- and the headline "posture accuracy 0.80" was **inflated** — a third of it was
  the model scoring easy n/a points, not reading actual student posture.

**Attempt 3 — back to masking, deliberately (current design). ACCEPTED.**
The product question was the decider: *we are simply not interested in a
teacher's posture or gaze.* So the head went back to **10 logits**, UNKNOWN
groups give **zero gradient**, and 100% of the posture/gaze capacity is spent on
students. The decode layer ignores teacher posture/gaze, which is honest — the
model genuinely has no opinion there. The N/A experiment was not wasted: it
*proved* with figures that explicit n/a classes hurt minority classes more than
they help interpretability.

## Chapter 5 — The class-imbalance war (three weapons)

The per-class numbers above are textbook class imbalance: `look_down` has 5,314
validation samples, `look_up` has **4**. One loss treats every sample equally,
so the model rationally ignores the rare classes. We fight back on three fronts
(all added 2026-06-10):

**Weapon 1 — learned class weights. ACCEPTED. Hardcoded weights REFUSED.**
At training start, the trainer counts every class in the *actual train labels*
and computes bounded inverse-frequency weights:
`w_c = sqrt(total / (C · count_c))`, clamped to `[0.5, 4.0]`. Rare classes get
their loss multiplied (up to 4×), common ones damped (down to 0.5×). Real
values from the full dataset: posture `[0.79, 1.55]`, gaze
`[0.74, 4.0, 1.58, 1.31, 4.0, 0.57]` (backward and up hit the 4.0 ceiling).
We refused hardcoding the numbers because the dataset can change — the weights
are re-learned from whatever labels are present, every run. The `sqrt` and the
clamp stop a 4-sample class from exploding the loss.

**Weapon 2 — focal modulation. ACCEPTED.** The posture/gaze cross-entropy is
multiplied by `(1 − p_true)^γ` with γ = 1.5 (tunable via the
`COMBINED_FOCAL_GAMMA` environment variable, not hardcoded in spirit). In plain
words: when the model is already confident and right (easy majority sample),
its loss shrinks toward zero; when it is unsure or wrong (usually a rare
class), the loss keeps full size. The gradient budget shifts from "polishing
look_down" to "finally learning look_backward". Role is left at γ = 0 — it is
nearly balanced and is also the detection-confidence signal, so we don't starve
it. Computed in float32 even under AMP (half precision) for numeric safety.

**Weapon 3 — the direction-aware flip (this one is a bug fix AND a weapon).**
See the next chapter — it is the biggest single finding of the project.

**Oversampling rare-class images — first REFUSED, then ACCEPTED as weapon 4
(its own chapter below).** When weapons 1–3 went in, we deliberately held
oversampling back: it interacts with mosaic, the 4-sample `look_up` class risks
being memorised, and we change one variable at a time so improvements stay
attributable. The owner then asked for it explicitly, so it was designed and
implemented as the *next* run's single new variable — see Chapter 11.

## Chapter 6 — The mirror that lied: the horizontal-flip label bug

This is the project's most important catch, and it came from a simple
non-technical question: *"isn't flipping dangerous for left/right classes?"*
Yes. Very.

YOLO trains with `fliplr=0.5`: **half of all training images are mirrored**.
The stock flip transform mirrors the picture and the boxes — but **never touches
the class labels**. Think about what a mirror does: a student who looks to the
camera's **left** appears, in the mirrored image, to look to the camera's
**right** — but the label still says `look_left`. At 50% flip probability,
roughly **half of all left/right training evidence was teaching the model the
exact opposite of the truth.** The two signals cancel out. That is almost
certainly why `look_left` recall was 0.002 — the model had learned that
"left vs right" is unlearnable noise and gave up.

**Decision: turn flipping off? REFUSED.** Flipping is one of the strongest,
cheapest augmentations, and for rare direction classes it can *double* the
training data — if the label flips with the image.

**Decision: re-implement the flip direction-aware. ACCEPTED.** We subclassed the
flip transform: when (and only when) the image is actually mirrored, the
composite label is remapped through a lookup table that swaps
`look_left ↔ look_right`. Everything else is mirror-invariant and untouched:
`forward/backward` (the depth axis doesn't change in a mirror), `up/down` (the
vertical axis doesn't change), posture, role, and UNKNOWN. We verified three
properties on the production box before training: the swap is correct, applying
the flip twice returns the original label (an "involution" — a sanity property
every flip must have), and teacher composites pass through unchanged.

**Decision: vertical flip force-disabled. ACCEPTED.** A vertical flip would
corrupt `look_up ↔ look_down` the same way. It is off by default (`flipud=0`),
but our trainer now *refuses* it with a warning even if someone enables it —
until someone implements an up/down-aware version.

The net effect: the mirror stops lying, and every mirrored image becomes a
**correctly-labelled free extra sample** for the rarest classes.

## Chapter 7 — The optimizer that silently ignored us

We wanted a bigger learning rate (`lr0=0.02`) so the model explores more of the
weight space, a longer warmup (8 epochs) so the pretrained backbone is not
wrecked by that big rate in the first minutes, and a cosine schedule for a
smooth landing. We passed `lr0=0.02` … and the log said:

```
optimizer: 'optimizer=auto' found, ignoring 'lr0=0.02' and 'momentum=0.937'
optimizer: MuSGD(lr=0.01, momentum=0.9)
```

Ultralytics' default `optimizer=auto` **silently overrides your learning rate**
and picks its own. Our "bigger LR" run was secretly running at the old 0.01.

**Decision: explicit `--optimizer SGD`. ACCEPTED.** We killed the mislaunched
run, added an `--optimizer` flag, and relaunched. The log now reads
`SGD(lr=0.02, momentum=0.937)` — the settings we asked for are the settings we
got. Lesson recorded: *always read the optimizer line of the log; "auto" means
"my hyper-parameters, not yours".*

## Chapter 8 — Stop training when learning stops

The 201-epoch N/A run's curves showed validation classification loss bottoming
around epoch ~45 and box loss around ~75 — then both **rising** while precision
oscillated. Everything after ~epoch 75 was the model memorising the train set
(overfitting). Patience 200 burned ~4 hours of a shared GPU for nothing.

**Decision: `patience 60`, `epochs 150`. ACCEPTED.** Early-stop within the
window the evidence showed, and size the cosine schedule to a realistic run
length so the learning rate actually completes its descent.

**Decision: `batch 9`, fixed. ACCEPTED.** The GPU is shared with the production
Triton server (~8 GB) and other people's workloads. Auto-batch once probed into
out-of-memory and once picked a uselessly tiny batch on a small GPU; a fixed,
proven batch is the polite and reproducible choice on shared hardware.

## Chapter 9 — Trust the numbers only if the figures can't fool you

Two figure decisions worth remembering:

- **All classes in every figure. ACCEPTED** after we discovered the default
  confusion matrix only showed `student`/`teacher` (the matrices were built from
  role-level matching). We now emit a combined all-classes matrix, per-group
  zoom matrices, per-class recall bars with support counts, role-conditioned
  heatmaps ("given the true role, what does the model predict?"), and montages
  labelled with the full `student|sitting|right` composite.
- **Grey "N/A" cells instead of zeros. ACCEPTED** after a misreading: off-block
  zeros in the combined matrix looked like "these classes are never confused",
  when really they are *structurally impossible* (different softmax groups).
  Impossible cells are now grey, so the figure cannot whisper a false story.

Also: validation **on** every epoch, with a custom grouped validator (role-level
mAP through the stock machinery + masked posture/gaze accuracy on IoU-matched
boxes) — the stock validator cannot score composite labels against 10 logits.
After the masking revert, attribute accuracy is computed **only on boxes that
truly have that attribute** (~10.7k student boxes, teachers excluded) — smaller
n, honest number.

## Chapter 10 — The run ledger

| Run | Head | Key settings | Result | Verdict |
|---|---|---|---|---|
| WSL 512px tests (×3) | 10-logit masked | batch 8 best | mAP50 ≈ 0.64 @512 | smoke-tests passed; moved to prod |
| `combined_m_prod_250` | 10-logit masked | flat LR, patience 200 | mAP50 0.486, late precision collapse | revealed flat-LR problem |
| `combined_m_prod_250_na` | **12-logit N/A** | cosine, patience 200, auto-optimizer | mAP50 0.459; posture 0.73 / gaze 0.62 (inflated by n/a); left 0.002, up 0.00 | **N/A design retired by this evidence** |
| `combined_m_prod_v3_masked` | 10-logit masked | SGD intended 0.02 — **auto ignored it** | killed at epoch ~10 | mislaunched; optimizer bug found |
| `combined_m_prod_v4_dirflip` | 10-logit masked | SGD lr0 0.02, warmup 8, cosine, patience 60, class weights, focal 1.5, **direction-aware flip** | running (150 epochs, ~5 h) | the everything-fixed run; baseline for v5 |
| `combined_m_prod_v5_oversample` | 10-logit masked | = v4 + **repeat-factor oversampling** (cap 3, median threshold, weights learned post-duplication); **owner relaunched mid-flight with `--optimizer Adamw --cache ram`** (replacing the SGD launch), so v5 now carries THREE deltas vs v4: oversampling + AdamW@0.02 + RAM cache | running | owner's variant; compare to v4 knowing the optimizer changed too |

**What to look at when v4 finishes:** `per_class_recall.png` for `look_left`,
`look_right`, `look_backward`, `standing` (the four corrupted/starved classes);
the role-conditioned heatmaps; and whether val losses now bottom later than
epoch ~75 (the LR/warmup change working).

## Chapter 11 — Weapon 4: repeat-factor oversampling (built, queued for v5)

The owner asked to apply an oversampling strategy. **Which strategy? Decision:
LVIS-style repeat-factor sampling at the IMAGE level. ACCEPTED.** It is the
standard remedy for long-tailed *detection* datasets, and it fits our pipeline
because it works by simple duplication: for each attribute class `c` we measure
`f(c)` = the fraction of train images containing `c`; classes rarer than a
threshold `t` get a repeat factor `r(c) = sqrt(t / f(c))`, capped at **3**; an
image repeats according to its **rarest** present class. The duplicated entries
are ordinary dataset rows, so mosaic and the direction-aware flip treat them as
normal images — every extra copy arrives with different augmentation, which is
exactly what a rare class needs.

The tiny details that make it safe:

- **The square root** keeps the boost gentle — a class 100× rarer than the
  threshold asks for 10×, and the **cap of 3** stops the 4-sample `look_up`
  from flooding the epoch with memorised copies.
- **Deterministic** — repeats are computed by rounding, no randomness, so the
  run is reproducible (`deterministic=True` stays honest).
- **No double-compensation** — this was the subtle trap. We already multiply
  rare-class losses by learned class weights (weapon 1). If both mechanisms saw
  the *original* distribution, a rare class would be boosted twice (up to
  3 × 4 = 12×). So the class weights are now computed **after** duplication:
  they see the already-rebalanced label counts and automatically shrink for
  boosted classes. The two weapons share the work instead of stacking blindly.
- **Tunable from the environment, not hardcoded** — `COMBINED_OVERSAMPLE_MAX`
  (default 3; set ≤ 1 to disable) and `COMBINED_OVERSAMPLE_T` (default −1 =
  the median class frequency decides what counts as "rare").
- **Tested before trusted** — a unit test on the production box fed the
  sampler a fake 10-image dataset: the two rare-class images tripled, the
  eight common ones stayed single, all parallel bookkeeping arrays stayed
  consistent, and the disable switch worked.

**Decision: do NOT restart the running v4. ACCEPTED.** v4 is the
"everything-fixed, no oversampling" run — it is the *baseline that makes v5
attributable*. The code is synced and `prod_run_v5_oversample.sh` is staged;
v5 launches when v4 finishes, identical in every way except this one new
variable. The cost to expect: the epoch gets ~20-40% longer (more images per
epoch), which is the price of the extra rare-class exposure.

---

*This journal is the project's memory. If a decision is not written here with
its reason, it didn't happen.*
