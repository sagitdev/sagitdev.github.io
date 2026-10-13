---
title: "Why I Started This Blog"
date: 2026-10-12
draft: false
tags: ["blogging", "Hugo", "learning"]
summary: "An engineering notebook kept in public: the decisions, dead ends, and lessons behind the systems I build."
---

## Why I'm writing

This blog is my engineering notebook, kept in public.

A repository shows *what* got built. It rarely shows *why*: the trade-offs I weighed, the bugs that cost me an afternoon, the design I threw away. That reasoning is usually the most valuable part of the work, and it's also the first thing I forget. Writing it down makes me understand problems more deeply, and it leaves a trail I can come back to the next time something similar breaks.

It's also a way to show how I work: how I approach unfamiliar problems, how I debug, and how a project changes as it grows.

## What you'll find here

- **Build logs**: milestones and design decisions from the projects I'm working on
- **Debugging stories**: the problems that stopped me, and how I tracked them down
- **Things worth remembering**: tools, techniques, and concepts I've picked up along the way
- **Retrospectives**: what worked, what didn't, and what I'd do differently

## A first lesson: setting up this blog

Even getting this site online came with a few surprises, which feels like a fitting start.

The stack is simple: **Hugo** with the **Blowfish** theme, hosted on **GitHub Pages**, with **GitHub Actions** handling deployment. Two things didn't go to plan:

1. **Homebrew couldn't install Hugo.** My Intel Mac is no longer a supported Homebrew configuration, so it tried to build Go from source and failed. Rather than fight the toolchain, I queried the GitHub Releases API for the exact asset names and installed the official `.pkg` directly.
2. **My home directory was secretly a Git repository.** `git submodule add` succeeded even though I hadn't run `git init`, and that was the red flag. `git rev-parse --show-toplevel` confirmed it: the theme had been registered in a repo rooted at `/Users/tank`. I removed it and set up a proper standalone repository for the blog.

The takeaway: **when something works but shouldn't, stop and find out why.** A surprising success is a bug that hasn't shown itself yet.

## Let's talk

If you have a suggestion, a question, or a better way to solve something I wrote about, I'd like to hear it. You can find me on [GitHub](https://github.com/sagitdev).

Thanks for reading.
