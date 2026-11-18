---
title: "Hello, World: Why I Built This Site"
date: 2026-11-11
tags: ["intro", "portfolio", "hugo"]
description: "What this site is for, what you’ll find here, and how it’s built."
ShowToc: false
---

Welcome! I’m Heli. I work on cloud infrastructure, Kubernetes, and MLOps, and this is where I write about it.

## Why a site?

Most of what I learn comes from things breaking: an IAM policy that’s one permission short, a CORS error that only shows up in production, a node group that won’t scale. Writing those fixes down helps me remember them, and I hope it saves someone else an afternoon.

## What you’ll find here

- **[Projects](/projects/):** deep dives into things I’ve built end to end, including the architecture, the implementation, and the parts that didn’t go to plan. Good places to start are the [LLMOps Support Assistant](/projects/llmops-support-assistant/), the [Cloud Resume Challenge](/projects/cloud-resume-challenge/) and [Plant Pal](/projects/plant-pal/).
- **Posts:** shorter notes, lessons learned, and how-tos.

## How it’s built

The site is a static [Hugo](https://gohugo.io/) site with the PaperMod theme. It’s hosted on GitHub Pages and deployed by a GitHub Actions workflow on every push, with DNS pointing my own domain at it. Infrastructure people can’t resist automating their own homepage.

If something here helps you, or you spot a mistake, I’d love to hear about it on [LinkedIn](https://www.linkedin.com/in/helipatelhp/).
