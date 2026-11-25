---
title: 'What I learnt fitting drift diffusion models (and moving from HDDM to HSSM)'
date: 2026-11-25
permalink: /posts/2026/11/ddm-hddm-hssm/
tags:
  - drift diffusion model
  - HDDM
  - HSSM
  - Bayesian statistics
  - MCMC
  - cognitive modelling
---
Since mid-2026 I've been working as a research assistant in the Emotion and Cognition Lab at the Centre for Neuroscience, IISc. The question behind the project is how emotional distractors (pleasant or unpleasant pictures you're told to ignore) interfere with a visual search task, and how that depends on perceptual load. I began with the usual tools: repeated-measures ANOVA, then linear mixed-effects models on trial-level reaction times (RTs). Then I moved to the drift diffusion model (DDM), first in **HDDM** and later in **HSSM**.

This post covers what I learnt along the way. Some of it is the theory. Most of it is the practical side that tutorials don't cover: why a model won't converge, why a model comparison criterion can mislead you, and why a fit that should take 6 hours took 26.

---

## 1. Why model reaction times with a DDM at all?

Mean RT tells you that a condition is slower. It doesn't tell you **why**. A person might be slower because:

- they take in information from the stimulus more slowly (the distractor really is getting in the way),
- they become more cautious and wait for more evidence before responding, or
- something outside the decision itself takes longer, such as encoding the display or pressing the key.

These are different psychological stories, and an ANOVA on mean RT can't separate them. The DDM can, because it uses the **whole RT distribution, for correct and error responses together**, and not just the mean.

A concrete example from our data: one distractor pair produced the same mean RT as two others. On RT alone they looked identical. The DDM suggested they got there by different routes. One had a higher drift rate (better evidence) offset by a higher threshold (more caution). That pair was also the most accurate, which fits the DDM's account. So two conditions can look identical in an LME and still differ inside the decision process.

## 2. The model in one paragraph

On each trial, evidence builds up noisily over time, starting at a point between two boundaries. When it reaches one boundary, the matching response is made. The parameters are:

| Parameter | What it means | What changes it |
|---|---|---|
| **v**, drift rate | how fast evidence builds up, i.e. how good the information is | stimulus difficulty, interference, attention |
| **a**, boundary separation | how much evidence you need before responding, i.e. caution | speed vs accuracy instructions, strategy |
| **t**, non-decision time | encoding and motor time added to every RT | sensory or motor delays |
| **z**, starting point | bias towards one response | prior expectation, payoffs |

Two coding choices matter a lot:

- **Accuracy coding**: the boundaries are "correct" and "error". **Stimulus coding**: the boundaries are the two responses (e.g. "X" and "N"). With accuracy coding a response bias can't be identified, so z is fixed at 0.5. Our task (letter search, two keys) used accuracy coding with z = 0.5.
- **Lapses**: some trials are guesses or attention slips. HDDM mixes in a 5% "outlier" component by default (`p_outlier = 0.05`). Keep this in mind, because it matters later.

## 3. Hierarchical and Bayesian, and why that matters

With around 50 trials per participant per condition, fitting a DDM to each person separately gives very noisy estimates. Hierarchical models estimate every participant's parameters together with a group distribution. Each person's estimate is pulled towards the group (**shrinkage**), most strongly when their data are thin. You get better individual estimates and properly computed group effects.

Lessons I learnt the hard way:

- **`depends_on` vs regression models.** In HDDM, `depends_on={'v': 'condition'}` gives each condition its own group distribution, as if the conditions were unrelated. In a within-subject design, each person's conditions belong together. A regression model (`HDDMRegressor`, e.g. `v ~ 1 + C(condition)`) gives each participant an intercept (the baseline condition) plus offsets, which matches the design.
- **Treatment coding is fine, and dropping the intercept does not fix mixing.** I tried cell-means coding (`0 + C(condition)`) to see whether it would improve the sampling of the condition terms. The intercept–offset correlation was weak (about −0.1), so that wasn't the problem. Removing the shared intercept also throws away the information that each person is measured in every condition, which would widen the intervals on the contrasts.
- **Build a model space and don't fit just one model.** We fitted 8 models: condition effects on v only, a only, t only, every pair, all three (`M_vat`), and a null model with no condition effects. The question becomes "which parameters does the manipulation affect?" and not "does this one model fit?".

## 4. Convergence: R̂, ESS and the funnel

MCMC gives you samples from the posterior, but only if the chains have converged and mixed. The two numbers to check:

- **R̂ (R-hat)** compares variation between chains with variation within chains. Modern guidance is R̂ < 1.01. HDDM-era papers often accepted 1.1.
- **Effective sample size (ESS)** is how many independent draws your correlated samples are worth. Aim for at least 400 on the quantities you report.

What I ran into with HDDM:

- **35,000 iterations, 4 chains, and still an ESS of 28 for some terms.** The poorly mixing terms were always the **condition offsets**, never the intercepts (ESS 900–1,700).
- **The cause was the hierarchical funnel.** When the data give no evidence that participants differ in a condition effect, the between-subject SD of that effect shrinks towards zero. All 55–65 subject offsets are then locked to the group offset (correlation about 0.99), and they can only move together. HDDM's sampler updates **one node at a time**, so it crawls along this narrow funnel in tiny steps.
- **Thinning doesn't help.** Keeping every 5th draw only discards correlated samples. I estimated that thinning 1 instead of 5 would barely raise ESS for these terms, and I dropped a refit after working this out. More chains help linearly at best (5 chains ≈ 1.25×).
- **Parameters trade off.** Within a condition, the a and t offsets were correlated at about −0.7. The model can explain a slow condition partly by caution and partly by non-decision time, which slows mixing and makes the a vs t split uncertain.
- **Bimodal participants.** One participant's posterior had two modes: low drift, low threshold and long t, or high drift, high threshold and short t. Their raw data looked completely normal. The group results didn't change depending on which mode was sampled, so they stayed in the sample. If you see one participant with a high R̂, look at their chains before you exclude them.

## 5. Checking the model: three kinds of validation

Model fitting doesn't stop at convergence. I came to see these three checks as essential.

### Posterior predictive checks (PPC)

Simulate data from the fitted posterior and compare it with the real data: accuracy, RT quantiles, per condition and per participant.

- My first PPC said the model predicted accuracy far too high. **It wasn't a bad fit.** HDDM's simulator leaves out the 5% lapse component that the fit assumes. With the lapses added back, predicted and observed accuracy agreed to within 0.03%. *Check that your simulator matches your likelihood.*
- Real misfits remained: the predicted slow tail was 10–30 ms too long, and **errors were faster than predicted**. This is a known limitation of the simple DDM. Fast errors need trial-to-trial variability in starting point (sz), which we didn't model.
- Side-by-side PPC plots of three models looked identical, because the differences were a pixel or two wide. **Plotting predicted minus observed** (residuals) made the meaningful differences visible.
- A targeted PPC on the **leading edge** (each participant's 10th-percentile correct RT, relative to their own mean) separated models that the overall plots couldn't. A drift-only model could not produce the condition shifts in the fastest responses.

### Parameter recovery

Simulate data with known parameters (I used the posterior means from the real fit, the real design and real trial counts), refit, and check whether you get the true values back. I ran 4 datasets.

- **Condition contrasts were recovered well**: about 92% of true values fell inside the 95% interval, close to nominal.
- **Absolute levels were biased.** Drift and threshold came out too low and t too high, because the simulated lapses didn't match the lapse density in the likelihood. Interpret differences between conditions, not raw parameter values.
- **Between-subject SDs were underestimated**, so effect sizes (d = contrast ÷ between-subject SD) for threshold were **inflated by about 30%**. I wouldn't have known this without recovery.
- It gave me a **power curve**. Effects of |d| ≥ 0.10 were detected in roughly 60–75% of datasets, and smaller ones rarely. So a non-credible small effect is *not* evidence that two conditions are equal.
- About 8% of near-zero contrasts came out "credible", mostly threshold contrasts. Small isolated a effects need caution.

One dataset is a single draw. Run several before quoting coverage rates.

### Model recovery

Simulate from each candidate model, fit every model to every dataset, and check whether the comparison criterion picks the model that generated the data (a confusion matrix). This was the most informative and humbling check of all.

- The null model was correctly chosen on null data and **always lost clearly when real effects existed**. So the comparison detects that the conditions differ.
- When it got the model wrong, it **always picked a slightly more complex model**, by tiny margins (ΔDIC about 6, Δelpd about 0.5). Extra parameters cost very little when their true effects are zero.
- From this I calibrated a threshold. An unneeded effect never gained more than about 6.6 DIC, while a genuinely missing threshold effect cost 12–14 DIC. On the real data, the full model beat its nearest rival by 9.7, which falls *between* those values. That makes it suggestive, not decisive.
- Effects on a and t were only weakly separable. **The model comparison could not reliably say whether a condition effect was on caution or on non-decision time.** That changed how strongly I could word the conclusions.

## 6. Model comparison: DIC, LOO, WAIC, BIC and Bayes factors

I compared all of these, and each one taught me something.

| Criterion | What it does | What I found |
|---|---|---|
| **DIC** | HDDM's default; deviance + effective number of parameters | Known to favour complex hierarchical models, but model recovery showed it was well calibrated here (spurious gains ≤ 6.6). Rule of thumb: Δ ≤ 10 is a tie. |
| **PSIS-LOO** | approximate leave-one-out cross-validation, from pointwise log-likelihoods | Tied all 7 models with condition effects (differences < 1–2 SE) and clearly rejected the null. About 40 trials with Pareto k > 0.7 drove its nominal preference, and so did two participants. |
| **WAIC** | another approximation to out-of-sample fit | Unreliable here (over 100 trials with p_waic > 0.4). |
| **BIC** | likelihood + k·ln(n) | Picked the null model every time. For hierarchical models, *k* (1,026 nodes, or about 440 effective parameters?) and *n* (15,688 trials or 55 participants?) aren't well defined, and the penalty size varied 17-fold depending on the choice. It ranks models by parameter count, not by the data. |
| **Bayes factors** | ratio of marginal likelihoods | HDDM doesn't compute marginal likelihoods. A Savage–Dickey check showed that with HDDM's very wide default priors on condition offsets (SD 15, 100–2,000× the real effects), real effects come out as "evidence for no effect" (Lindley's paradox). |

What I took away:

- **Use ties.** Rank models as "1 + the number of models meaningfully better", with each criterion's own threshold. "Tied first on DIC, LOO and WAIC" is more honest than a strict ranking.
- **Check the log-likelihood you rebuild.** HDDM doesn't save pointwise log-likelihoods, so I rebuilt them for LOO. I then checked them against HDDM's own likelihood nodes and they matched to about 1e-12. Mapping regression coefficients to trials is easy to get subtly wrong, and nothing would warn you.
- **Monte Carlo noise can be as large as the gaps between models.** Computing LOO from each chain separately gave a spread of 6–17 elpd, as big as the differences between the top models. So it isn't fair to give only the "winning" model extra samples. Refit all the contenders the same way, or say they can't be distinguished.
- **Base claims about parameters on posterior contrasts.** Model comparison told me that conditions differ and that drift is involved. Whether threshold and t also change had to come from the posterior contrasts, effect sizes and PPCs, read in light of the recovery results.

## 7. HDDM vs HSSM

HDDM has been the standard Python package for hierarchical DDMs for over a decade. **HSSM** (Hierarchical Sequential Sampling Models) is its successor, built on PyMC 5, with JAX-based samplers. I started in HDDM and moved the main model to HSSM once the mixing problem became clear.

| | HDDM | HSSM |
|---|---|---|
| Backend | PyMC 2 (old; runs in a pinned Docker image) | PyMC 5 + JAX / numpyro |
| Sampler | slice sampling, one node at a time | **NUTS** (Hamiltonian Monte Carlo), all parameters move jointly |
| Parameterisation | centred | **non-centred** available, which is the standard fix for the funnel |
| Model syntax | `HDDMRegressor` with patsy formulas | `bambi`-style formulas, including `(1 + C(cond) \| subj)` random effects |
| Output | traces in a pickled database; convert to ArviZ yourself | ArviZ `InferenceData` natively, with optional pointwise log-likelihood |
| Boundary **a** | full boundary separation | **half** the separation (the likelihood uses 2a), so **double HSSM's a to compare with HDDM** |
| Default priors on offsets | very wide (Normal SD 15), fine for slice sampling | you set them; weakly informative priors work much better with NUTS |
| Models beyond the DDM | mainly the DDM and a few variants | many sequential sampling models (LBA, race models, angle/collapsing bounds, …) |

### What happened when I refit the same model

Same model, same data (15,688 trials, 55 participants), and the same priors except the condition offsets (Normal(0, 1), with HalfNormal(0.5) on their SDs):

- **HDDM:** 35,000 iterations × 4 chains → condition-offset ESS of **90–205**, max R̂ ≈ 1.10.
- **HSSM (NUTS):** 1,000 warm-up + 1,000 draws × 4 chains → condition-offset ESS of **812–3,425**, every group-level R̂ ≤ 1.01, **0 divergences**. Autocorrelation fell to about zero within 1–3 draws.

That's **10–20× more effective samples from about a ninth as many draws**. Once I doubled HSSM's a, the estimates themselves agreed closely: the threshold intercept was 2.24 vs 2.25, a drift offset −0.353 vs −0.350, a threshold offset 0.116 vs 0.116. Both samplers found the same posterior. HSSM just explored it far more efficiently.

A prior-sensitivity check (how much the posterior contracts relative to the prior, and how much the prior density varies across the 94% HDI) showed the data, not the priors, were driving every group-level estimate.

### Practical HSSM lessons

- **The 26-hour fit that was really a 6-hour fit.** NUTS sampling took 5.7 h. The other 20 h went on HSSM computing the trial-wise log-likelihood afterwards, one draw at a time, in PyTensor's pure-Python fallback (the container had no C compiler, so each draw took 18 s). Running that step under `pytensor.config.change_flags(mode="JAX")` gave identical values (to 5e-15) at 0.09 s per draw. **If a step is mysteriously slow, check which backend it's actually running on.**
- **Don't run fits in parallel on a memory-bound machine.** Benchmarking the gradient showed that two fits at once each ran at half speed. The job was limited by memory bandwidth, not CPU cores. A simple queue that runs fits one after another was just as fast and much easier to manage.
- **Use the numpyro sampler when there's no compiler.** PyMC's default NUTS needs a C compiler. numpyro's runs on JAX.
- Check divergences, tree depth and E-BFMI as well as R̂ and ESS. These are NUTS-specific diagnostics with no HDDM equivalent.

### Which should you use?

If you're starting now, I'd use **HSSM**: modern sampler, proper diagnostics, ArviZ output, and better handling of hierarchical condition effects. HDDM is still worth knowing because a lot of the published literature uses it, and its tutorials explain DDM concepts well. If you compare results across the two, **remember the factor of 2 on a**.

## 8. The unglamorous side: compute and bookkeeping

A lot of what I learnt wasn't statistics at all:

- **Smoke-test everything.** Run 40–50 iterations end to end (fit, save, reload, summarise) before launching a 50-hour fit. Smoke-test run-time estimates can be badly inflated by start-up overhead, so estimate from a comparable full fit where you can.
- **Plan for disk and RAM.** One HDDM fit wrote about 2 GB of trace files. Gzipping the trace databases after saving the ArviZ file shrank them about 5×. HDDM also merges all chains into one process at the end of a fit, which is when memory peaks.
- **Containers save you.** HDDM needs old Python and PyMC 2. HSSM needs new ones. Two Docker containers, each with the repo mounted, kept them from conflicting.
- **Keep a dated log of decisions and results**, with why each decision was made. Several conclusions changed as more checks came in. For example, a model-recovery result I first read as "threshold absorbs t effects" turned out to come from a simulated dataset with almost no t effect. Being able to trace and correct the reasoning mattered as much as the results.
- **Never overwrite outputs.** Version every output folder (`rec01`, `rec02`, `analysis_25fits`, …). More than once a partial re-run would otherwise have silently replaced complete results.

## 9. Takeaways

1. A DDM tells you *how* a condition changes behaviour, not just *that* it does, but only if you validate it.
2. Fit a **model space**, and use regression-style within-subject models, not unrelated conditions.
3. Low ESS on condition effects in HDDM is usually the **funnel**. More iterations or less thinning barely help. NUTS with a non-centred parameterisation (HSSM) fixes it.
4. **PPC, parameter recovery and model recovery** are not optional. Each one changed my conclusions.
5. Interpret **condition contrasts**, not absolute parameter values, and check whether your effect sizes are biased.
6. No single model comparison criterion is the answer. **Calibrate DIC/LOO with model recovery**, treat small differences as ties, and don't use BIC for hierarchical models.
7. Be explicit about what the data can't tell apart. In our case that was threshold vs non-decision time.

## Reading that helped

- Ratcliff & McKoon (2008), *The diffusion decision model: theory and data for two-choice decision tasks*
- Ratcliff, Smith, Brown & McKoon (2016), *Diffusion decision model: current issues and history*
- Voss & Voss, *fast-dm* documentation (a good intuition for the parameters)
- Wiecki, Sofer & Frank (2013), *HDDM: Hierarchical Bayesian estimation of the drift-diffusion model in Python*
- The HSSM documentation and tutorials (LNCC lab, Brown University)
- *A practical introduction to using the drift diffusion model of decision-making in cognitive psychology, neuroscience, and health sciences* (Myers, Interian & Moustafa, 2022)
- Introductory MCMC and Bayesian statistics material (for R̂, ESS and why chains mix badly)
- Vehtari, Gelman & Gabry (2017), *Practical Bayesian model evaluation using leave-one-out cross-validation and WAIC*

---

*This work is part of ongoing, unpublished research in the Emotion and Cognition Lab (Prof. Srikanth Padmala), Centre for Neuroscience, IISc. The numbers above illustrate methods and lessons. They are not final results.*
