---
title: "LLMOps Support Assistant"
date: 2026-11-18
draft: false
description: "Taking a working ML + RAG app and making it measurable, versioned, monitored, load tested and safe to deploy, on k3s on AWS."
tags: ["kubernetes", "k3s", "aws", "terraform", "llmops", "mlops", "prometheus", "grafana", "github-actions"]
weight: 1
ShowToc: true
cover:
  image: "/images/llmops/drill-b2.png"
  alt: "Rollback drill: a bad image passes every probe; only monitoring catches it"
  hiddenInSingle: true
# Demo video: set ONE of these and the "Demo" section appears automatically.
# demoYouTube: ""                        # YouTube video ID (unlisted is fine)
# demoVideo: "/videos/llmops-demo.mp4"   # or an mp4 in static/videos/
---

## Introduction

I had a working app: a FastAPI service that classifies support tickets and answers IT questions with retrieval-augmented generation on a local LLM, deployed on Kubernetes on AWS. It "worked", but I couldn't answer the questions someone who owns a production system gets asked. How good is it? What exactly is running right now? Is it healthy, and where does the time go? How much load can it take? And what happens when a bad release ships?

The goal of this project was to answer each of those with evidence, on a hobby budget. I kept the models small on purpose: the point is everything around them.

{{< demo-video title="Demo" >}}

## What it does

- **`POST /predict`** classifies a ticket's priority (TF-IDF + logistic regression).
- **`POST /ask`** answers IT questions from a knowledge base: ChromaDB retrieves the right documents and tinyllama, running on Ollama, writes the answer.
- **`GET /version`** says exactly which model, prompt, retrieval config and git commit are running.

It runs on k3s on one AWS EC2 t3.large, created with Terraform and built by GitHub Actions.

## Architecture

![Architecture: Terraform and GitHub Actions on the left; on one EC2 node, k3s runs Traefik, the FastAPI pods, Prometheus and Grafana, while Ollama runs outside Kubernetes](/images/llmops/architecture.png)

- **Infrastructure:** Terraform creates the VPC, a security group that admits only my IP, the EC2 instance (IMDSv2, instance role), the ECR repository and the IAM roles. A cloud-init bootstrap installs k3s and Ollama and pulls the models.
- **Kubernetes:** the API runs as a Deployment with an HPA (2 to 5 pods) behind Traefik. Each pod builds its own vector index in an init container, and `/ready` refuses traffic if the index doesn't match the retrieval config.
- **Monitoring:** Prometheus scrapes every pod and holds the alert rules; the Grafana dashboard is committed as code.
- **CI/CD:** tests gate the build, and images are pushed to ECR with immutable git-SHA tags. GitHub logs in to AWS through OIDC, so there are no stored AWS keys anywhere.

![Grafana dashboard: rate, errors and duration per endpoint](/images/llmops/grafana-dashboard1.png)

## Key results

Every number here was measured, and the repo's results file records how, where and when.

| Question | Answer (measured) |
| --- | --- |
| How good is the classifier? | Macro F1 **0.50** on 150 held-out synthetic tickets. A char-n-gram v2 caught more outages but misrouted more medium tickets, so v1 stays the default |
| Does retrieval find the right doc? | hit@2 **0.96**, MRR **0.97** on 25 golden questions |
| Are the answers right? | tinyllama fully correct on **28%**; llama3 on **64%**, at about 2× the latency (LLM judge, calibrated and checked against my own scoring) |
| Where does the time go? | Generation is **~96%** of `/ask` time |
| How much load can it take? | `/predict` p95 **8 ms at 100 req/s**. `/ask` is capped at **~0.07 answers/s** by the LLM on one CPU core; the autoscaler added pods but no capacity |
| What happens when a bad release ships? | A bad config was stopped by the readiness probe with **0 user errors** and a **2 s** rollback. A bad image passed every probe; the alert fired after **3 min 39 s** and the rollback took **83 s** |

![MLflow: classifier v1 vs v2, macro F1](/images/llmops/mlflow2.png)

## What broke (the most useful part)

- **The rollback drill found a bug in my own fix.** My LLM timeout limited each *read*, and the answer streams token by token, so the clock reset on every token. It had never limited anything. I wrote about it in [My Rollback Drill Found a Timeout That Never Worked](/posts/rollback-drill-timeout/).
- **The autoscaler worked perfectly and achieved nothing.** Under load it scaled from 2 to 5 pods with no throughput gain. The CPU was being used by Ollama, which runs on the node *outside* Kubernetes, so pod CPU was the wrong signal.
- **A metric improved while the product got worse.** Retrieving 3 chunks instead of 2 fixed the only retrieval miss, but the small model then wrote longer, invented answers. I rejected that version.
- **My first evaluation scored a perfect 1.0.** The synthetic test set shared phrases with training. I think a perfect score on a hard problem is a bug until proven otherwise.

## Decisions I'd defend

- **Readiness checks only the pod's own faults.** If the shared LLM server is down, failing readiness would remove every pod, including `/predict`, which doesn't need the LLM.
- **The image tag is chosen at deploy time, never in the manifest,** so a `kubectl apply` can't silently roll everything back.
- **A small CPU model, on purpose.** The honest `/ask` SLO on this hardware is p95 ≤ 30 s, which is too slow for chat. Fixing that is a cost decision (GPU or hosted model), not a tuning fix.

## Cost

One t3.large, created per working session and destroyed afterwards. A clean rebuild takes 39 s of Terraform plus 2 min 12 s of bootstrap, and each session cost about $0.11–0.16.

## Limitations and what's next

The tickets and knowledge base are synthetic, the evaluation sets are small, and it runs on one node. Next, I would add a count-based alert and a synthetic probe for low traffic, automated rollback with canary analysis, GitOps with Argo CD, and a concurrency cap on `/ask`.

**Code, results and runbook:** [github.com/helipatel11/llmops-support-assistant](https://github.com/helipatel11/llmops-support-assistant)
