---
layout: post
title: SharpShot v0.5 DevLog
date: 2019-07-25 18:00:00 +0100
description: The latest version of sharpshot adds editing functionality, significantly increasing usability.
img: posts/20190722/sharpshot.png # Add image post (optional)
tags: [SharpShot] # add tag
youtubeId: rmkditieXoE
youtubeDesc: A simple board which computes (A+B)*C
---

# Reworking input handling
The old method was a mess
Many nested layers were manually ordered by how they nested.
The simple solution would be to one input layer which handles all mouse input, but we would end up reinventing a lot of what javafx gives us automatically.
This has been simplified by creating one layers class which automatically handles the nesting and allows the input priority to be controlled by the ordering of one list.

---