---
layout: post
title: What forgetting looks like when a robot policy is fine-tuned on human demonstrations
description: A π0.5 policy lost a prior skill within 200 steps of fine-tuning on human demonstrations of a new task, well before it learned that task.
og_image: /assets/img/blog/csir_fig1_learning_curves.png
date: 2026-10-08 09:00:00-07:00
tags: robot-learning continual-learning fine-tuning
categories: research
giscus_comments: false
related_posts: false
---

<p class="post-tldr"><strong>TL;DR</strong> I fine-tuned a robot policy on a new task with 50 human demonstrations and tracked a skill it already had. The old skill was gone before the new one arrived: its success fell from 0.70 to 0.14 after 100 steps and to 0.00 after 200, while the new task only began improving around step 1,060, so neither early stopping nor a lower learning rate could save it. Drift from the starting policy, the kind of measure that predicts forgetting in language models, didn’t catch the early loss either.</p>

<p class="post-tldr">In language models, on-policy RL forgets less than supervised fine-tuning (SFT), and the gap has been traced to the data rather than the algorithm. As a control, I ran SFT on the policy’s own successful rollouts. The same supervised updates didn’t erase the old skill this time, suggesting that data close to the policy’s own behavior limits forgetting, and that RL isn’t the only way to avoid it. But these rollouts also barely improved the new task, so they aren’t a recipe on their own. To separate the two effects, the next step is data that stays close to the policy but still teaches: the robot’s own attempts, corrected by a stronger model or a human only where it fails. I leave that for future work.</p>

<hr class="post-rule">

A robotics founder once told me about a customer whose towel-folding robots were working well, and who then asked them to also set aside towels that were slightly dirty. Requests like that are the norm once robots leave the lab: a model that went through extensive pretraining has to pick up small new skills quickly, without losing the ones it already has.

Recent work found that on-policy reinforcement learning (RL) forgets far less than supervised fine-tuning on vision-language-action (VLA) models.&nbsp;<a class="cite" href="#ref-1">[1]</a> In language models, that difference has been traced to the data RL trains on, sampled from the model itself, rather than to the RL objective.&nbsp;<a class="cite" href="#ref-2">[2]</a> I wanted to look at this on a robot policy: I held the training objective fixed, changed only the data, and looked closely at how the forgetting unfolds.

## The setup

<div class="row post-figure">
  <div class="col-sm mt-3 mt-md-0">
    {% include figure.liquid loading="eager" path="assets/img/blog/csir_fig2_design.png" class="img-fluid rounded z-depth-1" zoomable=true alt="Schematic of the experiment in three panels. Left, Start: a π0.5 policy after on-policy RL on the prior task of placing moka pots, with prior-task success 0.70 and new-task success 0.35. Middle: four fine-tuning versions from the same start. Three share one SFT setup and differ only in data: Own, the policy’s successful rollouts; Demo, 50 human demonstrations; Demo+Own, the demonstrations plus 17 of the policy’s own prior-task rollouts. The fourth, RL, uses on-policy GRPO with 320 rollouts as an objective ablation. Right, Evaluate: new-task success, prior-task success, and displacement from Start measured as action MSE." %}
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

| Training data | New task (bowls) | Old task (moka pots) |
| --- | --- | --- |
| Starting policy | 0.35 | 0.70 |
| RL | 0.35 | 0.73 |
| Own | 0.41 | 0.77 |
| Demo + Own | 0.69 | 0.70 |
| Demo | 0.71 | 0.00 |

The old skill went to zero in every Demo run. It held or improved in all six Own runs. The same split held on two more new tasks, one closely related to the old task and one unrelated: demonstrations nearly erased the old skill (0.05 and 0.00), while the policy's own data kept it (0.69 and 0.71) but learned neither new task.

In one run I looked at closely, the failures looked less like a forgotten task than a lost motor skill: the policy still reached for the right moka pot, then closed the gripper late and lifted it in only 40% of attempts.

<div class="row post-figure">
  <div class="col-sm mt-3 mt-md-0">
    {% include figure.liquid loading="eager" path="assets/img/blog/csir_fig1_learning_curves.png" class="img-fluid rounded z-depth-1" zoomable=true alt="Two line charts of task success against fine-tuning step. Left, trained on 50 human demonstrations: the old task drops from 0.7 to 0 within 200 steps while the new task rises to 0.7 by step 1590. Right, trained on 143 of the policy's own rollouts: the old task stays near 0.75 and the new task rises modestly to about 0.45." %}
  </div>
</div>
<div class="caption">
  Task success during supervised fine-tuning (SFT), measured at each training step. Orange is the old task, putting the moka pots on the stove; blue is the new task, stacking bowls. Left: trained on 50 human demonstrations. Right: trained on 143 of the policy's own successful rollouts. Bars are 95% confidence intervals.
</div>

## Forgetting comes before learning

Under human demonstrations, the old skill is gone before the new one arrives. After 100 training steps, old-task success had fallen to 0.14; after 200, to 0.00. The new task fell too at first, and only started improving from around step 1,060.

That ordering rules out the obvious fixes. Stopping training early, once the new task looks good, can't help, because the old skill is already gone by then. A lower learning rate didn't help either: demonstration training at a fifth of the rate still erased the old skill and never got far enough to learn the new task.

Under the policy's own data, the old skill stayed between 0.73 and 0.78 at every checkpoint, and the small new-task gain came in the second half of training.

## The surprise: RL gains that didn't show up at deployment

I expected RL on the new task to be the interesting arm. Its training curves looked promising: success on its own training rollouts rose from 0.25 in the first iteration to 0.61 in the last. But when I evaluated it the way the policy is actually deployed, the new task hadn't moved at all (0.35 before and after), and neither had the old one.

The gap comes from sampling. RL on this kind of policy needs a noisy sampler during training so each action has a likelihood to optimize. Deployment decodes deterministically, without that noise. The two can disagree a lot: the starting policy succeeded on the old task 0.88 of the time under the training sampler, but 0.70 under deployment decoding.

One side effect I didn't predict: training on the policy's own successes nearly closed that gap. Afterward, old-task success was 0.84 under the training sampler and 0.80 under deployment decoding. Some of what the noisy sampler could do seems to carry over into the deterministic policy, though I don't yet know why.

The practical lesson: training curves aren't evidence of improvement. Track a self-improving robot under the exact protocol it will be deployed with. My RL budgets were small, far below published RL fine-tuning of these models, so this says nothing about RL at scale. But it's what pushed me to hold the objective fixed and change only the data.

## Distance from the start isn't the whole story

In language models, how far fine-tuning shifts a model's output distribution from where it started (its KL divergence) predicts how much it forgets. I measured something similar for the robot: how different the fine-tuned policy's actions are from the starting policy's on the same observations.

Across final checkpoints, distance separated the outcomes cleanly. Every checkpoint that moved far lost the old skill, and every one that stayed close kept it.

But distance alone can't predict forgetting. After just 100 steps of demonstration training, the policy was about as close to the start as the final own-data policies, yet it had already lost most of the old skill (0.14 versus 0.77). Meanwhile, Demo + Own moved nearly twice as far as own data alone and kept the skill fully (0.70). Two updates at the same distance can differ in whether the old skill survives. What matters is the data.

## What it means, limits, and what's next

The policy's own data is a simple, safe ingredient, but it has a ceiling. It protected the old skill on all three new tasks, yet it can only teach what the policy already sometimes does, so it learned just one of the three new tasks, modestly.

Replay works, and whose data is replayed matters. Mixing the policy's own old-task successes into the demonstrations kept the old skill at 0.70 while learning as much as demonstrations alone. The same share of human demonstrations of the old task kept it only partly (0.59). My guess is that the policy's own rollouts carry the improved version of the skill, while the human demonstrations carry the version from before RL improved it.

Next, I want to test the data that sits in between: mostly the robot’s own behavior, with simulated human corrections only where it fails. Comparing those trajectories with pure human demonstrations would show whether data that stays close to what the policy already does can teach new tasks without erasing old ones. If the pattern above holds, corrections should learn faster than the policy’s own data while forgetting less than demonstrations.

I’m also curious about scale. Recent work found that pretrained VLAs are surprisingly resistant to forgetting in continual learning,&nbsp;<a class="cite" href="#ref-5">[5]</a> yet here 50 demonstrations erased a skill within 200 steps. I’d like to measure directly how resistance to forgetting changes with the size of the pretrained model and the amount of pretraining.

Limits: simulation only, one model family, one old skill tested against three new tasks, and single runs on two of those pairs. The old task also appears in the base model's training data, which may make it easier to retain than a skill the model never saw.

## References

<ol class="references">
  <li id="ref-1">Hu et al., “Simple Recipe Works: Vision-Language-Action Models are Natural Continual Learners with Reinforcement Learning”, RLC 2026. <a href="https://arxiv.org/abs/2603.11653">arXiv:2603.11653</a></li>
  <li id="ref-2">Shenfeld et al., “RL’s Razor: Why Online Reinforcement Learning Forgets Less”, arXiv 2025. <a href="https://arxiv.org/abs/2509.04259">arXiv:2509.04259</a></li>
  <li id="ref-3">Physical Intelligence, “π0.5: a Vision-Language-Action Model with Open-World Generalization”, arXiv 2025. <a href="https://arxiv.org/abs/2504.16054">arXiv:2504.16054</a></li>
  <li id="ref-4">Liu et al., “LIBERO: Benchmarking Knowledge Transfer for Lifelong Robot Learning”, NeurIPS 2023. <a href="https://arxiv.org/abs/2306.03310">arXiv:2306.03310</a></li>
  <li id="ref-5">Liu et al., “Pretrained Vision-Language-Action Models are Surprisingly Resistant to Forgetting in Continual Learning”, arXiv 2026. <a href="https://arxiv.org/abs/2603.03818">arXiv:2603.03818</a></li>
</ol>
