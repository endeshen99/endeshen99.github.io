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

<p class="post-tldr">In language models, on-policy RL forgets less than supervised fine-tuning (SFT), and the gap has been traced to the data rather than the algorithm. As a control, I ran SFT on the policy’s own successful rollouts. The same supervised updates didn’t erase the old skill this time, but they also barely improved the new task. This suggests on-policy data could be one ingredient in a training recipe with little forgetting, though what kind of on-policy data actually teaches remains open.</p>

<hr class="post-rule">

A robotics founder once told me about a customer whose towel-folding robots were working well. The customer’s most urgent request was something the robots were never trained to do: spotting and setting aside towels that were slightly dirty. Requests like that are the norm once robots leave the lab: a model that went through extensive pretraining has to pick up small new skills quickly, without losing the ones it already has.

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

The gap comes from how this policy picks actions. π0.5 is a flow-matching model: it produces each action by starting from random noise and refining it over a few steps, and at deployment that refinement is deterministic. RL needs something the deterministic version doesn’t give: the probability of each action the robot took, so it can make good actions more likely. πRL&nbsp;<a class="cite" href="#ref-5">[5]</a>, the method I used, gets around this by adding a little randomness at every refinement step during training. So the policy RL trains and scores is a noisier version of the one that gets deployed, and the two can behave quite differently: the starting policy succeeded on the old task 0.88 of the time with the training noise, but only 0.70 without it.

One side effect I didn't predict: training on the policy's own successes nearly closed that gap. Afterward, old-task success was 0.84 under the training sampler and 0.80 under deployment decoding. Some of what the noisy sampler could do seems to carry over into the deterministic policy, though I don't yet know why.

The practical lesson: training curves aren't evidence of improvement. Track a self-improving robot under the exact protocol it will be deployed with. My RL budgets were small, far below published RL fine-tuning of these models, so this says nothing about RL at scale. But it's what pushed me to hold the objective fixed and change only the data.

## Distance from the start isn't the whole story

In language models, how far fine-tuning shifts a model's output distribution from where it started (its KL divergence) predicts how much it forgets.&nbsp;<a class="cite" href="#ref-2">[2]</a> I measured something similar for the robot: how different the fine-tuned policy's actions are from the starting policy's on the same observations.

Across final checkpoints, distance separated the outcomes cleanly. Every checkpoint that moved far lost the old skill, and every one that stayed close kept it.

But distance alone can't predict forgetting. After just 100 steps of demonstration training, the policy was about as close to the start as the final own-data policies, yet it had already lost most of the old skill (0.14 versus 0.77). Meanwhile, Demo + Own moved nearly twice as far as own data alone and kept the skill fully (0.70). Two updates at the same distance can differ in whether the old skill survives. Something about the data matters beyond how far it moves the policy.

<div class="row post-figure">
  <div class="col-sm mt-3 mt-md-0">
    {% include figure.liquid loading="eager" path="assets/img/blog/csir_fig3_prior_vs_displacement.png" class="img-fluid rounded z-depth-1" zoomable=true alt="Scatter plot of old-task success against displacement from the starting policy on a log scale. Own-data checkpoints cluster at moderate displacement with success near 0.7 to 0.8. Two Demo checkpoints at step 100 sit at similar displacement but near 0.15 success, marked with an arrow. Demo+Own sits farther right at about 0.6 to 0.7 success. Final Demo checkpoints sit far right at zero success. RL sits far left at about 0.7." %}
  </div>
</div>
<div class="caption">
  Old-task success against how far each checkpoint’s actions moved from the starting policy (log scale). Distance separates most outcomes, but not all: after 100 steps, demonstration training (Demo, step 100) is about as close to the start as the own-data runs, yet has already lost most of the skill, while demonstrations mixed with the policy’s own data (Demo+Own) move farther and keep it.
</div>

## What it means, limits, and what's next

Training on the policy’s own successes protected the old skill on all three new tasks, but it has a clear ceiling: it can only reinforce what the policy already sometimes does. It learned just one of the three new tasks, and only modestly.

Replaying old data during training also helped, and whose data it was mattered. When I mixed some of the policy’s own old-task successes into the demonstrations, the old skill held at 0.70, and the new task was learned as well as with demonstrations alone. Mixing in the same share of human demonstrations of the old task kept only part of it (0.59). My guess is that the policy’s own rollouts capture the skill as RL improved it, while the human demonstrations capture it as it was before.

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
