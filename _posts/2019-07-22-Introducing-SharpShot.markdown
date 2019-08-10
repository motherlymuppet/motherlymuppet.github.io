---
layout: post
title: Introducing SharpShot
date: 2019-07-22 18:00:00 +0100
description: I'm making a zach-like puzzle game. In this post I explain how it's going and the future plans.
img: posts/20190722/sharpshot.png # Add image post (optional)
tags: [SharpShot] # add tag
youtubeId: rmkditieXoE
youtubeDesc: A simple board which computes (A+B)*C
---

# Origin

In October 2018, I participated in Durhack, a hackathon run by the university's computing society.
As a group of three, we wrote v0.1 of SharpShot in 24 hours[^1].
At first, the idea was simple: A 2-dimensional visual programming language.
The first commit was 2 hours after the hackathon started, and by hour 5 it was fully usable[^2].

The remaining 17 hours contained more playing with the program than developing it.
It is incredibly addictive - when taking the Github screenshot I accidentally spent 5 hours playing around with it instead.
SharpShot won the "Github prize for best developer tool", and placed second overall in the hackathon.

[^1]:Which is still available for [download via github](https://github.com/motherlymuppet/SharpShot/releases/tag/0.1)

[^2]: I know this thanks to a commit optimistically titled [Hackathon done!](https://github.com/motherlymuppet/SharpShot/commit/158d76c4ee01c7ccc2c50a855bb216d669a1361e) at 4:45pm

# What is Sharpshot?

Sharpshot is a game where players attempt to solve a series of challenges by combining simple components.
They place **nodes** on a rectangular grid known as a **board**.
Its size is configurable by the user.
Each node performs a pre-determined function on **bullets**, which carry data around the board.
When a bullet leaves the board, it is written to the output.
When two bullets collide, they annhilate each other[^3].

[^3]: This is actually the main way to perform boolean logic. I wasn't kidding when I said esoteric!

{% include ytSmall.html id=page.youtubeId desc=page.youtubeDesc %}

A solution is scored on three metrics:

* **Speed** - The total number of ticks taken for the test suite
* **Cost** - Each node placed increases the cost, with more complex nodes costing more
* **Size** - The total number of squares on the board

Faster, cheaper, and smaller solutions are better.
The solution will be given a percentile grading for each of these metrics, relative to all other players.
It is almost impossible to optimise all three metrics at once, so most solutions will optimise for just one.
To pass the level, a solution does not need to be optimal - it just has to pass the test suite.

# Current State & Roadmap

The current version of the game does not have set levels and does not provide a score.
It is still a sandbox while development continues.
The core gameplay is quite stable and fun to play around with, which you can do by downloading it [via github](https://github.com/motherlymuppet/SharpShot/releases).

The game is currently in version 0.4.1, and there's a lot left to do before v1.0. In it's current state, the roadmap is as follows:

* **v0.5** - 
Move, rotate, and delete large sections of a board.
Save these extracts as components to allow for reuse.
* **v0.6** - 
Create infrastructure for challenges, allowing boards to be ran against a test suite.
Populate the game with levels in many categories of various difficulties.
* **v0.7** -
Add online component and accompanying server code to score solutions on the 3 metrics.
Add leaderboards and ability to browse top solutions to each level.
* **v1.0** -
Graphical and sound overhaul.
Increase general polish of game and make it look less like enterprise software.
* **v1.1** -
Support user-made levels.

I'll let you know how that goes!

If you have any suggestions or comments, definitely get in touch. PRs are always welcome, it's open-source for a reason!

---