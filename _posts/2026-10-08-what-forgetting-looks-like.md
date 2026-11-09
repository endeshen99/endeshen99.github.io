---
layout: post
title: What forgetting looks like when a robot policy is fine-tuned on human demonstrations
description: A π0.5 policy lost a prior skill within 200 steps of fine-tuning on human demonstrations of a new task, well before it learned that task.
og_image: /assets/img/blog/csir_fig1_learning_curves.png
date: 2026-10-08 09:00:00-07:00
last_updated: 2026-11-05
tags: robot-learning continual-learning fine-tuning
categories: research
giscus_comments: false
related_posts: false
---

<p class="post-tldr"><strong>Update (Nov 2026)</strong> This work was accepted at the CoRL 2026 <a href="https://csircorl.github.io/website/">Workshop on Continually Self-Improving Robots (CSIR)</a>, Austin, Nov 12. The post was revised to match the camera-ready paper, which adds a full-precision replication, a measurement of how off-policy the demonstrations are, and a bf16 training defect found along the way. The paper is here: <a href="/assets/pdf/shen2026datasource.pdf">Data Source Decides Forgetting (PDF)</a>.</p>

<p class="post-tldr"><strong>TL;DR</strong> I fine-tuned a robot policy on a new task with 50 human demonstrations and tracked a skill it already had. The old skill was gone before the new one arrived: its success fell from 0.70 to 0.14 after 100 steps and to 0.00 after 200, while the new task only began improving around step 1,060. Neither early stopping nor a lower learning rate could save it. Drift from the starting policy, the kind of measure that predicts forgetting in language models, didn’t catch the early loss.</p>

<p class="post-tldr">In language models, on-policy RL forgets less than supervised fine-tuning (SFT), and the gap has been traced to the data rather than the algorithm. As a control, I ran SFT on the policy’s own successful rollouts. The same supervised updates didn’t erase the old skill this time, but they improved the new task less than demonstrations did (+0.15 vs +0.50 at full precision). This suggests on-policy data could be one ingredient in a training recipe with little forgetting, though what kind of on-policy data actually teaches remains open.</p>

<hr class="post-rule">

Once robots leave the lab, a model that went through extensive pretraining has to pick up small new skills quickly, without losing the ones it already has. Recent work found that on-policy reinforcement learning (RL) forgets far less than supervised fine-tuning on vision-language-action (VLA) models.&nbsp;<a class="cite" href="#ref-1">[1]</a> In language models, that difference has been traced to the data RL trains on, sampled from the model itself, rather than to the RL objective.&nbsp;<a class="cite" href="#ref-2">[2]</a> I wanted to look at this on a robot policy: I held the training objective fixed, changed only the data, and looked closely at how the forgetting unfolds.

## The setup

<div class="row post-figure">
  <div class="col-sm mt-3 mt-md-0">
    {% include figure.liquid loading="eager" path="assets/img/blog/csir_fig2_design.png" class="img-fluid rounded z-depth-1" zoomable=true alt="Schematic of the experiment in three panels. Left, Start: a π0.5 policy after on-policy RL on the prior task of placing moka pots, with prior-task success 0.69 and new-task success 0.35. Middle: four fine-tuning versions from the same start. Three share one SFT setup with the same objective, optimizer, budget, and initialization, and differ only in data: Own, the policy’s successful rollouts on the new task; Demo, 50 human demonstrations of the new task; Demo+Own, the demonstrations plus 17 of the policy’s own prior-task rollouts. The fourth, RL, is an objective ablation: on-policy GRPO on the new task with 320 rollouts, the same rollouts under the RL objective. Right, Evaluate: new-task success, prior-task success, and displacement from Start, drawn as the distance d between the Start point and the fine-tuned point, measured as action MSE." %}
  </div>
</div>
<div class="caption">
  <strong>The experiment.</strong> Every version starts from the same π0.5 policy, already improved by RL on the old task (moka pots), and is fine-tuned on a new task (stacking bowls) with the same training setup, changing only the data: the policy’s own successful rollouts (Own), 50 human demonstrations (Demo), or the demonstrations plus some of the policy’s own old-task successes (Demo+Own). RL on the new task is a comparison that changes the training objective instead. Each version is scored on both tasks, and on how far its actions moved from the starting policy.
</div>

The starting point is a π0.5&nbsp;<a class="cite" href="#ref-3">[3]</a> policy in the LIBERO&nbsp;<a class="cite" href="#ref-4">[4]</a> simulator that I had already improved with on-policy RL on one task: putting both moka pots on the stove. RL raised its success there from 0.55 to 0.70. That improved skill is what I wanted to protect.

Next, I fine-tuned it on a new task, stacking one bowl on another and placing them in a tray, from a different scene. Every version used the same training loss, optimizer, step budget, and starting point. Only the training data changed:

- **Own:** the policy's own successful attempts at the new task, with no data from the old task.
- **Demo:** 50 human demonstrations of the new task.
- **Demo + Own:** the demonstrations plus 17 of the policy's own successes on the old task, about a third of the training frames.
- **RL:** for comparison, the same RL recipe on the new task, as a self-improving robot could afford it.

Each checkpoint was scored on both tasks over the same seeded episodes, so every comparison is paired.

## What happened

Fifty human demonstrations taught the new task and erased the old one. Across runs, success on the new task rose from 0.35 to 0.71 while success on the old task fell from 0.70 to 0.00, and it hit zero in every demonstration run. The same objective on the policy's own successful rollouts kept the old skill in all six runs, which points to the data rather than the training objective. The rest of this post is about what the loss looked like.

| Training data   | New task (bowls) | Old task (moka pots) |
| --------------- | ---------------- | -------------------- |
| Starting policy | 0.35             | 0.70                 |
| RL              | 0.35             | 0.73                 |
| Own             | 0.41             | 0.77                 |
| Demo + Own      | 0.69             | 0.70                 |
| Demo            | 0.71             | 0.00                 |

| Full precision (fp32 master weights) | New task (bowls) | Old task (moka pots) |
| ------------------------------------ | ---------------- | -------------------- |
| Starting policy                      | 0.40             | 0.70                 |
| RL/5it                               | 0.51             | 0.69                 |
| Own                                  | 0.55             | 0.79                 |
| Demo                                 | 0.90             | 0.05                 |

The second table is a replication with full-precision training, explained below: the ordering is unchanged, and the Own and RL gains are now visible.

The old skill went to zero in every Demo run. It held or improved in all six Own runs. The same split held on two more new tasks, one closely related to the old task and one unrelated: demonstrations nearly erased the old skill (0.05 and 0.00), while the policy's own data kept it (0.69 and 0.71) but learned neither new task.

In one run I looked at closely, the failures looked less like a forgotten task than a lost motor skill: the policy still reached for the right moka pot, then closed the gripper late and lifted it in only 40% of attempts.

<div class="row post-figure">
  <div class="col-sm mt-3 mt-md-0">
    {% include figure.liquid loading="eager" path="assets/img/blog/csir_fig1_learning_curves.png" class="img-fluid rounded z-depth-1" zoomable=true alt="Two line charts of task success against fine-tuning step. Left, trained on 50 human demonstrations: the old task drops from 0.7 to 0 within 200 steps while the new task rises to 0.7 by step 1590. Right, trained on 143 of the policy's own rollouts: the old task stays near 0.75 and the new task rises modestly to about 0.45. Circled points mark matched displacement from the starting policy: demonstrations at step 100 with d = 7.0, and the final own-rollout checkpoint with d = 8.1. Dotted lines show the starting policy's success on each task." %}
  </div>
</div>
<div class="caption">
  Task success during supervised fine-tuning (SFT), measured at each training step. Orange is the old task, putting the moka pots on the stove; blue is the new task, stacking bowls. Left: trained on 50 human demonstrations. Right: trained on 143 of the policy's own successful rollouts. Bars are 95% confidence intervals. The circled points sit at about the same distance from the starting policy (d, action MSE ×10<sup>−3</sup>): demonstrations after 100 steps have already lost the old skill, while the own-rollout run at the end of training has kept it.
</div>

## Forgetting comes before learning

Under human demonstrations, the old skill is gone before the new one arrives. After 100 training steps, old-task success had fallen to 0.14; after 200, to 0.00. The new task fell too at first, and only started improving from around step 1,060.

That ordering rules out the obvious fixes. Stopping training early, once the new task looks good, can't help, because the old skill is already gone by then. A lower learning rate didn't help either: demonstration training at a fifth of the rate still erased the old skill and never got far enough to learn the new task.

Under the policy's own data, the old skill stayed between 0.73 and 0.78 at every checkpoint, and the small new-task gain came in the second half of training. The old skill even ended a little above where it started in all six runs. My guess is that this comes from the sampler: the starting policy succeeds on the old task 0.88 of the time under the noisy training sampler but 0.70 under the deterministic one used at evaluation, and after training on its own successes the two nearly agree (0.84 and 0.80), as if the successes carried some of the sampler’s competence into the deterministic policy.

## A precision trap

I expected RL on the new task to be the interesting condition. Its training curves looked good, but evaluated the way the policy is actually deployed, the new task hadn't moved (0.35 before and after). In the first version of this post I blamed the sampler. The real cause was simpler: most of the model was never able to change.

Every run in the original study trained in bf16 with no fp32 copy of the weights. bf16 stores a number with only about three significant digits, so a weight of size 0.02 can only move in steps of roughly 0.0001. The optimizer's update at a learning rate of 5e-6 is far smaller than that, so each step rounds back to the old value and nothing accumulates. Only weights smaller than about 0.002 could move at all. In the action expert, that left 97.6% of the attention and MLP weights untouched from start to finish, in every condition. The training was real; it just happened in a small corner of the model.

<div class="row post-figure post-figure-narrow">
  <div class="col-sm mt-3 mt-md-0">
    {% include figure.liquid loading="eager" path="assets/img/blog/csir_precision_poster.png" class="img-fluid rounded z-depth-1" zoomable=true alt="Bar chart of the fraction of the action expert's bf16 attention and MLP entries that changed during training, grouped by entry magnitude at Start, for RL after 15 iterations, the final Own checkpoint, and the final Demo checkpoint. Entries below 0.001 changed in 98 to 99 percent of cases, entries between 0.001 and 0.002 in 81 to 89 percent, and entries at or above 0.002, which is the 2 to the minus 9 cutoff, changed in none of the three conditions." %}
  </div>
</div>
<div class="caption">
  Fraction of the action expert’s bf16 attention and MLP entries that changed during training, by entry magnitude at the starting policy: no entry at or above 2<sup>−9</sup> moved in any condition.
</div>

I re-ran the three main conditions in full precision. Under that protocol the starting policy scores 0.40 on the new task and 0.70 on the old one; RL reaches 0.51 and 0.69, own-rollout SFT reaches 0.55 and 0.79, and demonstrations reach 0.90 and 0.05. The ordering is the same as before and the gaps are larger: RL and SFT on the same rollouts learn the same amount and both keep the old skill, while demonstrations learn more and erase it. The sampler mismatch I originally blamed is real but minor (the starting policy scores 0.88 on the old task with the training noise, 0.70 without).

This is the default in the RLinf and πRL configs (`precision: null`, no fp32 master weights). If you fine-tune with either, check yours.

## Distance from the start isn't the whole story

In language models, how far fine-tuning shifts a model's output distribution from where it started (its KL divergence) predicts how much it forgets.&nbsp;<a class="cite" href="#ref-2">[2]</a> I measured something similar for the robot: how different the fine-tuned policy's actions are from the starting policy's on the same observations.

Across final checkpoints on all three task pairs, distance separated the outcomes cleanly: every checkpoint displaced by more than 20 (action MSE, ×10<sup>−3</sup>) lost the old skill, every one below 15 kept it, and the only one in between, human-demonstration replay of the old task at 15.8, kept it partly (0.59).

But distance alone can't predict forgetting. After just 100 steps of demonstration training, the policy was about as close to the start as the final own-data policies, yet it had already lost most of the old skill (0.14 versus 0.77). Meanwhile, Demo + Own moved nearly twice as far as own data alone and kept the skill fully (0.70). Two updates at the same distance can differ in whether the old skill survives. Something about the data matters beyond how far it moves the policy.

<div class="row post-figure post-figure-narrow">
  <div class="col-sm mt-3 mt-md-0">
    {% include figure.liquid loading="eager" path="assets/img/blog/csir_fig3_prior_vs_displacement.png" class="img-fluid rounded z-depth-1" zoomable=true alt="Scatter plot of old-task success against displacement from the starting policy on a log scale. Three RL checkpoints sit far left, at displacement 0.2 to 1.5, with success 0.7 to 0.8. Own-data checkpoints cluster at displacement about 8 with success near 0.7 to 0.8, labeled Own, final. Two Demo checkpoints at step 100 sit at similar displacement but near 0.15 success, labeled Demo, step 100. Demo+Own sits farther right at about 0.6 to 0.7 success. Final Demo checkpoints sit far right, at displacement 30 to 45, at zero success, labeled Demo, final." %}
  </div>
</div>
<div class="caption">
  Old-task success against how far each checkpoint’s actions moved from the starting policy (log scale). Distance separates most outcomes, but not all: after 100 steps, demonstration training (Demo, step 100) is about as close to the start as the own-data runs, yet has already lost most of the skill, while demonstrations mixed with the policy’s own data (Demo+Own) move farther and keep it.
</div>

What differs about the data is how far its targets sit from what the policy would already do. For every frame in each training set, I gave the starting policy the set’s own observation and measured the mean squared error between the stored action chunk and the policy’s own prediction, in the units of the displacement metric above. For the human demonstrations the gap is 148.8 [143.4, 154.2]; for the policy’s own rollouts it is 30.1 [27.1, 32.9]. Part of each number is the policy’s own sampling noise, which puts a floor of 24.4 under the demonstrations and 15.8 under the rollouts. The demonstrations are about five times further off-policy on the mean and sixteen times on the median. The gap is concentrated in the middle of each episode, where the object is grasped and carried, and over half of it (57%) is gripper timing: the demonstrator closes earlier and holds longer than the policy does. Even on the six arm channels alone the numbers are 67.1 against 8.1, about eight times. The gap is spread across all 50 demonstrations rather than concentrated in a few outliers. That gripper offset also lines up with the failure I saw earlier: the erased skill breaks at exactly the point where the gripper should close. Demonstrations ask the policy to produce actions it would not produce on its own, and that is the difference the forgetting tracks.

<div class="row post-figure post-figure-narrow">
  <div class="col-sm mt-3 mt-md-0">
    {% include figure.liquid loading="eager" path="assets/img/blog/csir_offpolicy_poster.png" class="img-fluid rounded z-depth-1" zoomable=true alt="Bar chart of chunk mean squared error from the starting policy for two training sets. Demo, 50 human demonstrations, is 148.8 with a resampling floor of 24.4; Own, 38 of the policy's own rollouts, is 30.1 with a resampling floor of 15.8. Error bars are 95 percent intervals and dashed lines mark the floors." %}
  </div>
</div>
<div class="caption">
  How far each training set’s action targets sit from the starting policy’s own actions on the same observations (mean chunk MSE, ×10<sup>−3</sup>, with 95% bootstrap intervals over episodes). Dashed lines are the resampling floor: the gap between two draws of the starting policy at the same frame.
</div>

## What it means, limits, and what's next

Training on the policy’s own successes protected the old skill on all three new tasks. It learns less than demonstrations do because it can only reinforce what the policy already sometimes does, and at full precision its gain on the new task matched RL’s while both kept the old skill.

Replaying old data during training also helped, and whose data it was mattered. When I mixed some of the policy’s own old-task successes into the demonstrations, the old skill held at 0.70, and the new task was learned as well as with demonstrations alone. Mixing in the same share of human demonstrations of the old task kept only part of it (0.59). My guess is that the policy’s own rollouts capture the skill as RL improved it, while the human demonstrations capture it as it was before.

<div class="row post-figure post-figure-narrow">
  <div class="col-sm mt-3 mt-md-0">
    {% include figure.liquid loading="eager" path="assets/img/blog/csir_replay_poster.png" class="img-fluid rounded z-depth-1" zoomable=true alt="Bar chart of old-task success at the final checkpoint for three training sets: the demonstrations alone, 0.00; the demonstrations plus 17 human demonstrations of the old task, 0.59; the demonstrations plus 17 of the policy's own old-task rollouts, 0.70. A dotted line marks the starting policy at about 0.70. Error bars are 95 percent intervals." %}
  </div>
</div>
<div class="caption">
  Old-task success at the end of training when the 50 demonstrations are mixed with replay of the old task: none, 17 human demonstrations of it, or 17 of the policy’s own successful rollouts of it. The dotted line is the starting policy. Bars are 95% confidence intervals.
</div>

Replay has a cost, though: it protects only what gets replayed. A robot with many skills would need old data for every one of them, and that cost grows with each skill it learns. Training on the policy’s own rollouts of the new task protected the old skill without any old-task data at all. If that holds across many skills, which I haven’t tested, its cost wouldn’t grow with the robot’s repertoire.

Next, I want to test the data that sits in between: mostly the robot’s own behavior, with simulated human corrections only where it fails. Comparing those trajectories with pure human demonstrations would show whether data that stays close to what the policy already does can teach new tasks without erasing old ones. If the pattern above holds, corrections should learn faster than the policy’s own data while forgetting less than demonstrations.

I’m also curious about scale. Recent work found that pretrained VLAs are surprisingly resistant to forgetting in continual learning,&nbsp;<a class="cite" href="#ref-6">[6]</a> yet here 50 demonstrations erased a skill within 200 steps. I’d like to measure directly how resistance to forgetting changes with the size of the pretrained model and the amount of pretraining.

Limits: simulation only, one model family, one old skill tested against three new tasks, and single runs on two of those pairs. The old task also appears in the base model's training data, which may make it easier to retain than a skill the model never saw.

## References

<ol class="references">
  <li id="ref-1">Hu et al., “Simple Recipe Works: Vision-Language-Action Models are Natural Continual Learners with Reinforcement Learning”, RLC 2026. <a href="https://arxiv.org/abs/2603.11653">arXiv:2603.11653</a></li>
  <li id="ref-2">Shenfeld et al., “RL’s Razor: Why Online Reinforcement Learning Forgets Less”, arXiv 2025. <a href="https://arxiv.org/abs/2509.04259">arXiv:2509.04259</a></li>
  <li id="ref-3">Physical Intelligence, “π0.5: a Vision-Language-Action Model with Open-World Generalization”, arXiv 2025. <a href="https://arxiv.org/abs/2504.16054">arXiv:2504.16054</a></li>
  <li id="ref-4">Liu et al., “LIBERO: Benchmarking Knowledge Transfer for Lifelong Robot Learning”, NeurIPS 2023. <a href="https://arxiv.org/abs/2306.03310">arXiv:2306.03310</a></li>
  <li id="ref-5">Chen et al., “πRL: Online RL Fine-tuning for Flow-based Vision-Language-Action Models”, arXiv 2025. <a href="https://arxiv.org/abs/2510.25889">arXiv:2510.25889</a></li>
  <li id="ref-6">Liu et al., “Pretrained Vision-Language-Action Models are Surprisingly Resistant to Forgetting in Continual Learning”, arXiv 2026. <a href="https://arxiv.org/abs/2603.03818">arXiv:2603.03818</a></li>
</ol>
