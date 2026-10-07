---
layout: about
title: about
permalink: /
subtitle: Independent researcher · robot learning

profile:
  align: right
  image: prof_pic.jpg
  image_circular: false # crops the image to make it circular

selected_papers: true # includes a list of papers marked as "selected={true}"
social: true # includes social icons at the bottom of the page

announcements:
  enabled: false # includes a list of news items
  scrollable: true # adds a vertical scroll bar if there are more than 3 news items
  limit: 5 # leave blank to include all the news in the `_news` folder

latest_posts:
  enabled: false
  scrollable: true # adds a vertical scroll bar if there are more than 3 new posts items
  limit: 3 # leave blank to include all the blog posts
---

I'm an independent researcher in San Francisco working on robot learning. I study what happens to vision-language-action (VLA) models when they keep learning after deployment: which behaviors survive fine-tuning, which are silently lost, and what that means for safety. I run experiments on π0.5, both in simulation and on a low-cost real arm.

My background is in verifiable ML. As Staff Cryptographer at Modulus Labs, I built Remainder, a zero-knowledge proving system for ML inference, and at Tools for Humanity I implemented MPC proofs for a system serving ~22M users. I also founded Tegore (YC S25), a real-time voice tutor for K-12 math. The thread through all of it: how do we know what a system will do before it acts?

I did my B.S. and M.S. in Computer Science at Stanford, where I worked on faithful text generation in Tatsu Hashimoto's group.

#### On my mind

<!-- TODO: replace these placeholder questions with your own. -->

- Which safety-relevant behaviors in a VLA are most fragile under continued fine-tuning, and can we predict that before training?
- When a robot learns from its own rollouts, what does it quietly unlearn?
- What would a useful certificate of "this policy will not do X" even look like?
