---
layout: page
title: Real-robot self-improvement on OpenMANIPULATOR-X
description: Fine-tuning π0.5 on its own rollouts with a low-cost arm (in progress)
img: # TODO: add a preview image, e.g. assets/img/omx_arm.jpg
importance: 1
related_publications: false
---

An ongoing study of what a vision-language-action model gains and loses when it keeps learning after deployment, run on a real OpenMANIPULATOR-X arm rather than in simulation. The policy is π0.5, fine-tuned on a case-to-mug pick-and-place task, then fine-tuned again on its own self-generated rollouts.

**Artifacts on Hugging Face**

- Model: [endeshen/pi05_omx_case_to_mug](https://huggingface.co/endeshen/pi05_omx_case_to_mug)
- Dataset: [endeshen/omx_case_to_mug_20260710_175217](https://huggingface.co/datasets/endeshen/omx_case_to_mug_20260710_175217)

**Videos**

<!-- TODO: drop rollout videos into assets/video/ and reference them here. Example:
<div class="row">
  <div class="col-sm mt-3 mt-md-0">
    {% include video.liquid path="assets/video/omx_rollout_1.mp4" class="img-fluid rounded z-depth-1" controls=true autoplay=false %}
  </div>
</div>
-->

_Videos coming soon._
