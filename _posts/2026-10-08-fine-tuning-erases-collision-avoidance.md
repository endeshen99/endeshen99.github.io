---
layout: post
title: Fine-tuning on obstacle-free demos erases a VLA's learned collision avoidance
description: A safe π0.5 policy fine-tuned on 150 obstacle-free demonstrations went from a 2.5% to a 30.8% collision rate while its skill on the original tasks largely held.
og_image: /assets/img/blog/spais_fig5_filmstrips.png
date: 2026-10-08 10:00:00-07:00
tags: robot-learning safety fine-tuning
categories: research
giscus_comments: false
related_posts: false
---

<p class="post-tldr"><strong>TL;DR</strong> I took a robot policy trained to avoid obstacles and fine-tuned it on ordinary task demonstrations that contained no obstacles. Its collision rate rose from 2.5% to 30.8%, while its skill on the original tasks largely held. The avoidance started eroding within the first 200 training steps, before task success dropped. Telling the robot to avoid obstacles did nothing; mixing the safe policy’s own rollouts into training limited the damage, and 200 further steps on them restored it. Learned safety here behaves like a motion habit, so it needs re-testing after every update.</p>

<hr class="post-rule">

Robot policies are increasingly trained to behave safely: avoid obstacles, keep clear of people. That behavior is usually measured once, on the model as released. But a deployed policy rarely stays as released. Teams fine-tune it on demonstrations of their own tasks, and the safety behavior is assumed to come along.

For language models, we already know it often doesn't: fine-tuning an aligned model on a small amount of harmless data can erode its refusals.&nbsp;<a class="cite" href="#ref-1">[1]</a> I wanted to know whether the same thing happens to a robot policy.

## The setup

I started from a π0.5&nbsp;<a class="cite" href="#ref-2">[2]</a> policy released by LIBERO-Safety&nbsp;<a class="cite" href="#ref-3">[3]</a>, a benchmark that trained it on about 19,700 collision-free demonstrations to steer around obstacles on a tabletop. Call it the safe policy.

Then I did what a downstream user would do: fine-tuned it on 150 human demonstrations of three of the same tasks (a bowl, a book, and a moka-pot task), in their original scenes, which have no obstacles. Nothing in this data teaches the robot to collide. It just never shows an obstacle.

I scored every checkpoint two ways on the same seeded episodes, so each comparison is like-for-like:

- **Safety:** the obstacle scenes. Did the robot or the object it carries touch an obstacle?
- **Skill:** the original scenes. Did it finish the task?

One practical lesson came before any results. The benchmark's built-in collision flag never fired, even when I scripted the gripper straight into an obstacle. I computed collisions from the simulator's contact list instead, which reported 14.2% where the flag reported 3.0% on the same episodes. Anyone evaluating safety should check that the collision metric can actually detect a collision.

## What happened

Fine-tuning raised the collision rate from 2.5% to 30.8% across 120 paired episodes. Safe success, meaning finishing the task without touching anything, fell from 85.8% to 26.7%.

The bowl task shows it most cleanly, because its obstacle scene is the original scene plus one obstacle and nothing else. After fine-tuning, the robot still completed the bowl task 90% of the time, but its collision rate went from 0% to 25%. In the original scene, its skill was unchanged at 95%.

In one paired episode, both policies pick up the bowl and place it on the plate in about the same number of steps. The safe policy passes beside the obstacle. The fine-tuned one clips it on the way down. Same task, same success, different path.

<div class="row post-figure">
  <div class="col-sm mt-3 mt-md-0">
    {% include figure.liquid loading="eager" path="assets/img/blog/spais_fig5_filmstrips.png" class="img-fluid rounded z-depth-1" zoomable=true alt="Two filmstrips of a robot arm moving a bowl to a plate past a dark box. Top row, labeled S1: the safe policy passes beside the box. Bottom row, labeled Demo: the fine-tuned policy clips the box at step 97, outlined in red." %}
  </div>
</div>
<div class="caption">
  The same bowl episode under the safe policy (top, labeled S1) and after fine-tuning (bottom, labeled Demo). Both finish the task; only the fine-tuned policy clips the obstacle, at step 97.
</div>

On bowl, the robot didn't get worse at its job. It stopped doing the one thing nobody showed it during fine-tuning. Skill held on the book task too; the moka task was the exception, where the policy lost both its avoidance and its skill.

## Avoidance goes first, and asking doesn't bring it back

The safety behavior is the first thing fine-tuning removes. On the static-obstacle scenes, the collision rate rose from 5% to 19.2% within the first 200 training steps, while task success barely moved (85% to 79.2%). Success only dropped later, by which point collisions had reached 42.5%.

So a quick check of task success after fine-tuning would have looked fine. The safety loss is invisible unless it is tested for specifically.

I also tried simply telling the robot. Appending "avoiding the obstacles" to its instruction changed nothing: collisions went from 51.7% to 50%. In hindsight this makes sense. The safe policy's instructions never mentioned obstacles, so avoidance was never tied to language in the first place. It was a reaction to what the robot saw, and after fine-tuning it no longer reacts.

## Replay helps, and the loss is shallow

Mixing the safe policy's own rollouts into the fine-tuning data, at about a third of the training frames, limited the damage without preventing it:

| Fine-tuning data | Collision rate | Safe success |
| --- | --- | --- |
| Safe policy, before fine-tuning | 2.5% | 85.8% |
| Human demos only | 30.8% | 26.7% |
| Demos + own rollouts, without obstacles | 24.2% | 52.5% |
| Demos + own rollouts, with obstacles | 15.8% | 62.5% |

Rollouts that contained the hazard protected more, and a second training seed reproduced that ordering. On bowl, both replay versions held collisions near the original level. The moka task collided under every version, which is why the pooled numbers stay high.

The loss also turned out to be shallow. Starting from the degraded policy, 200 more training steps on the safe policy's own rollouts brought the collision rate back to 4.2%. What 2,118 steps of fine-tuning removed, 200 steps restored.

The damage is also local. Near obstacles, the fine-tuned policy's actions drifted from the safe policy's about 1.6 times more than elsewhere. The forgetting isn't uniform drift that happens to break avoidance; it sits right where avoidance lives.

## What this suggests

In this setup, learned safety behaves less like an understanding of obstacles and more like a motion habit, tied to the data that installed it. A safety evaluation of the released model says little about a fine-tuned one, and task success gives no warning, so safety has to be re-checked after every update. As more safety behavior is trained directly into robot foundation models, that check will matter more.

## Limits and what's next

This is one base model, one benchmark, three fine-tuning tasks, and simulation only, with one or two training seeds per setup. The skill results rest mainly on two tasks, since moka's skill collapsed under every version.

The question I'd most like to answer next is what would make safety behaviors durable under routine fine-tuning, and how we would check that they were. One candidate is safety learned as a concept rather than a habit. Probing work&nbsp;<a class="cite" href="#ref-4">[4]</a> suggests robot policies already carry internal signals for ideas like an upcoming task failure. If a similar representation of “unsafe” exists, or can be installed, I could check whether it survives fine-tuning, and test whether safety anchored to it lasts longer than safety anchored to motion.

That representation need not come from language. It could be conditioned on what the robot sees, so that the policy first registers an obstacle in the image and then plans around it, rather than reproducing a swerve it learned from one set of trajectories. It would also give a cleaner diagnostic than collision counts: if the “unsafe” signal still fires after fine-tuning but the robot collides anyway, the concept survived and only the motion was lost.

## References

<ol class="references">
  <li id="ref-1">Qi et al., “Fine-tuning Aligned Language Models Compromises Safety, Even When Users Do Not Intend To!”, ICLR 2024. <a href="https://arxiv.org/abs/2310.03693">arXiv:2310.03693</a></li>
  <li id="ref-2">Physical Intelligence, “π0.5: a Vision-Language-Action Model with Open-World Generalization”, arXiv 2025. <a href="https://arxiv.org/abs/2504.16054">arXiv:2504.16054</a></li>
  <li id="ref-3">Cui et al., “LIBERO-Safety: A Comprehensive Benchmark for Physical and Semantic Safety in Vision-Language-Action Models”, ECCV 2026. <a href="https://arxiv.org/abs/2606.23686">arXiv:2606.23686</a></li>
  <li id="ref-4">Gu et al., “SAFE: Multitask Failure Detection for Vision-Language-Action Models”, NeurIPS 2025. <a href="https://arxiv.org/abs/2506.09937">arXiv:2506.09937</a></li>
</ol>
