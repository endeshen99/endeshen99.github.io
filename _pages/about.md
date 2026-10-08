---
layout: about
title: about
permalink: /
subtitle: Independent researcher · continual learning & safety for robots

profile:
  align: right
  image: prof_pic.jpg
  alt: Ende Shen # alt text for the profile photo
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

I'm an independent researcher in San Francisco working on continual learning and safety for robots. I study what happens to vision-language-action (VLA) models when they keep learning after deployment: which behaviors survive fine-tuning, which are lost, and what that means for safety.

Before robotics, I worked on zero-knowledge proofs for machine learning, as a cryptographer at Modulus Labs on the Remainder proving system and then at Tools for Humanity. I also founded Tegore (YC X25), a voice-based math tutor for K-12 students.

I did my B.S. and M.S. in Computer Science at Stanford, where I worked on text generation with [Tianyi Zhang](https://tiiiger.github.io/) in [Tatsu Hashimoto](https://thashim.github.io/)'s group.

#### On my mind

- Can we tell from a VLA's internals what fine-tuning will erase before it happens?
- Do the safety behaviors trained into robot foundation models survive the routine fine-tuning that happens after deployment?
- Should self-improvement methods be judged by a frozen benchmark score, or by how fast they improve under a fixed budget?
