---
title: "My Rollback Drill Found a Timeout That Never Worked"
date: 2026-11-17
draft: true
description: "I shipped a deliberately broken release to practise rolling back. The rollback worked; the drill found a streaming LLM timeout that had never limited anything."
tags: ["kubernetes", "k3s", "sre", "monitoring", "prometheus", "llmops", "aws"]
ShowToc: true
cover:
  image: "/images/llmops/drill-b2.png"
  alt: "Drill B: a bad image passes every probe; only monitoring catches it"
  hiddenInSingle: true
---

I shipped a deliberately broken release to my own system, to practise rolling back. I expected it to fail loudly. Instead, most requests succeeded and my alert stayed completely silent. Something was wrong and nothing came up.

The rollback itself worked fine. The real problem was a timeout I had added earlier in the project, which, it turns out, had never limited anything. In this post I explain how a rollback drill found it, and why I now think a drill tests your assumptions more than your tools.

## The project

The app is a small support API with two jobs. `POST /predict` classifies a ticket's priority using TF-IDF and logistic regression. `POST /ask` answers IT questions using retrieval-augmented generation: ChromaDB finds the right documents, and tinyllama, running on Ollama, writes the answer. It runs on k3s on one AWS EC2 t3.large, built by GitHub Actions and created with Terraform.

I kept the models small on purpose. What I wanted to learn was everything around them, the questions an owner of a production system gets asked: How good is it? What exactly is running right now? Is it healthy? How much load can it take? And what happens when a bad release ships? This post is about the last question. All numbers are from 15 November 2026, and the method behind each one is in the repo's results file.

## The drill

I planned two kinds of bad release, because they fail in different ways and need different protection. First, a bad config, where the pod itself can tell something is wrong. Second, a bad image, where the pod looks perfectly healthy and only the users would notice.

| | Drill A: bad config | Drill B: bad image |
| --- | --- | --- |
| What I broke | Pointed generation at `llama3`, a model the node doesn't have | Built an image with a 5-second LLM deadline, when answers take about 13 s on this hardware |
| What should catch it | The readiness probe | Monitoring: an error-ratio alert |
| Why | The pod knows it is broken | The pod looks healthy |

The safety net was the same for both:

- **Immutable images:** every image is tagged with its git SHA, and the tag is chosen at deploy time, never written in the manifest.
- **Careful rollouts:** an old pod is removed only after a new one is Ready (`maxUnavailable: 0`, `maxSurge: 1`). If a rollout makes no progress for 240 s, Kubernetes marks it failed.
- **Readiness that checks only the pod's own faults:** `/ready` fails if the pod's index is broken or the node doesn't have this release's models. I think this is the most important design choice here. It does *not* fail when the shared Ollama server is down, because then every pod would leave service at once, including `/predict`, which doesn't even need the LLM.
- **An alert per endpoint:** more than 5% of `/predict` or `/ask` requests returning 5xx for 1 minute.

During the whole drill, background traffic kept running: 2 requests per second to `/predict` and one `/ask` every 20 seconds.

## Drill A: the readiness probe does its job

Drill A went exactly as planned: zero user errors. I released the bad config at 04:32:09 UTC. The new pod started, and its `/ready` endpoint answered 503 with a clear reason: `ollama is missing models: llama3`. The error said exactly what was wrong, so the fix was obvious.

Because the rollout never removes an old pod before a new one is Ready, the two old pods simply kept serving. At 04:36:11 Kubernetes marked the rollout `ProgressDeadlineExceeded`. During the whole window, all 481 `/predict` and 12 `/ask` requests succeeded. Rolling back with `kubectl rollout undo` took 2 seconds, since the old pods were still running.

This is the easy case: the pod knew it was broken, and said so.

![Drill A: the new pod never becomes Ready, and user errors stay at zero](/images/llmops/drill-a.png)

## Drill B: the alert that never fired

The bad image passed every probe, as expected. What I did not expect: most of its requests succeeded. With a 5-second deadline and 13-second answers, nearly every `/ask` should have returned 503. But the alert never fired, and after watching for a while I rolled back by hand, which took 88 seconds.

What was going on? My first check was the obvious one: were the bad pods really running with the 5-second setting? They were. The config inside the image said 5 s. The setting was there, it just wasn't doing what I thought.

The cause was in what the timeout actually measured. I had passed it to the HTTP client (httpx) that talks to Ollama. An httpx timeout limits each *read*, meaning how long to wait for the next piece of data. The LLM sends its answer back token by token, so every new token arrived well within 5 seconds and reset the clock. A request could run for a whole minute without ever hitting it.

The few errors I did see in that first run were `ReadTimeout`s, from the rare moments when the gap between two tokens was longer than 5 s. And that meant something worse: my real production setting of 60 s had never limited anything either. If a 5-second gap between tokens was already rare, a 60-second gap would almost never happen. So as long as the model kept producing tokens, a request could stay open for minutes with nothing to stop it. That is not at all acceptable for a timeout.

## The fix, and the re-run

The fix was simple once I understood the problem: put the deadline on the whole answer, not on each read. The code checks the clock as tokens arrive. When the deadline passes, it raises an error and closes the stream, so Ollama stops generating too:

```python
deadline = time.monotonic() + config.LLM_TIMEOUT_S
parts = []
stream = get_llm().stream(prompt)
try:
    for chunk in stream:
        parts.append(chunk.content)
        if time.monotonic() > deadline:
            raise LLMDeadlineExceeded(f"no complete answer within {config.LLM_TIMEOUT_S} s")
finally:
    stream.close()
```

I also added a unit test that feeds a deliberately slow stream and checks for a 503 and a closed stream. Locally, with a 2-second deadline, the answer was cut off at 2.01 s. Before the fix, the same request had "succeeded" after 5.7 s.

Then I ran drill B again with a new bad image. This time:

| Time (UTC) | Event |
| --- | --- |
| 04:53:58 | First bad pod Ready and taking traffic |
| 04:54:48 | Rollout complete: all pods on the bad image |
| 04:58:27 | Alert `EndpointErrorRatioHigh` firing for `/ask` |
| 04:58:29 | I ran `kubectl rollout undo` |
| 04:59:52 | Rollback complete: 83 s |
| 04:59:58 | Errors back to zero |

The alert fired 3 min 39 s after the rollout finished. `/predict` returned no 5xx at any point, because only `/ask` uses the LLM. And this time the error told me exactly what was wrong: `stage="generation", error="LLMDeadlineExceeded"`. No guessing needed. An error should say what is wrong and where to look, whether the person reading it is a customer or the engineer on call.

![Drill B, second run: pods by commit, /ask completions, and the alert going pending then firing](/images/llmops/drill-b2.png)

## Why 3 min 39 s, not 1 minute

You may wonder why the alert took 3 min 39 s when it only needs 1 minute. The answer is that ratio alerts behave badly at low traffic. `/ask` received only about 3 requests a minute, and the alert needed the error ratio above 5% for a full minute without a break.

Two things kept resetting that timer. First, two `/ask` requests happened to finish within the deadline, and in a quiet minute one success was enough to drop the ratio below the threshold. Second, right after the rollout, some 1-minute windows had no completed requests at all, so the ratio couldn't be calculated. The alert kept switching between pending and inactive until enough failures lined up.

At this traffic level, a percentage over one minute is simply the wrong tool. What I'd do next is a count-based alert for `/ask` (for example, "3 or more 5xx in 5 minutes") and a synthetic probe that asks a known question every minute, so there is always something to measure.

## What I took away

- **A drill tests your assumptions, not just your rollback button.** The rollback worked both times. What failed was my belief that the timeout capped how long a request could run. It never did.
- **Know which faults each check can see.** Readiness catches a pod that knows it is broken. Only monitoring catches a pod that looks healthy while users get errors.
- **Read what a timeout actually measures.** Per-read, per-connection and whole-request timeouts are different things, and streaming hides the difference: my 60 s per-read timeout let an answer run as long as tokens kept coming.
- **Ratio alerts need traffic.** An error percentage only works when there are enough requests to measure. With a few a minute, a single success can reset the alert, so alert on error counts instead and add a synthetic probe that sends a request every minute.

The code, the drill evidence (timeline logs, Prometheus exports, rollout history) and the full write-up are in the repo: [helipatel11/llmops-support-assistant](https://github.com/helipatel11/llmops-support-assistant). The project overview is on the [LLMOps Support Assistant](/projects/llmops-support-assistant/) page.
