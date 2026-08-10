---
layout: default
title: Tools & Software
permalink: /software/
description: "Open-source research tools and software by Sakshi Bhalla, including breakingnews, a story segmentation tool for broadcast news transcripts."
---

## Tools & software

<div class="pub entry">
  <span class="date">2026</span>
  <p class="title">breakingnews: Parameter-efficient LLM adaptation for segmenting broadcast transcripts</p>
  <p class="byline">Python package · MIT license</p>
  <p class="links">
    <a href="https://pypi.org/project/breakingnews/">PyPI</a>&nbsp;
    <a href="https://doi.org/10.5281/zenodo.21863477">DOI</a>
  </p>
  <p class="software-desc">A broadcast news transcript arrives as an undifferentiated block of 2,000 to 17,000 words covering several unrelated stories. Content analysis needs a comparable unit, and a television transcript is not one article — treating it as one, or splitting it on speaker turns, gives the wrong denominator. This package finds the word offsets where one story ends and the next begins, emits one row per story, and reconstructs the source byte for byte.</p>
  <p class="software-desc">Boundaries are identified by a LoRA adapter on Llama-3.1-8B, fine-tuned on 998 annotated transcripts containing 2,829 boundaries drawn from US television news between 1992 and 2020. Segmentation is treated as a partition: every character lands in exactly one story.</p>
</div>
