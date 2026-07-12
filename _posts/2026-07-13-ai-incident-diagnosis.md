---
title: What if your logs could diagnose themselves at 3am?
author: Bow
tags: devops
---

It's 3:04 AM. Your payment service just started throwing errors. Somewhere, an on-call engineer's phone is buzzing.

Here's what happens next in most companies: they open a laptop, squint at a dashboard, open another dashboard, grep through logs across four or five services, try to remember if this happened before, and — 30 to 45 minutes later — finally understand *what* broke. Only then do they start actually fixing it.

That gap — between "something is wrong" and "I understand why" — is where I spent the last few weeks building **Sentinel**, a small AI-powered incident triage platform, to see how much of it could be automated.

## The idea, in one sentence

Most observability tools are excellent at telling you *that* something broke. Almost none of them tell you *why*, in plain English, in the first 60 seconds. Sentinel tries to close that gap.

## A concrete walkthrough

I built a mock e-commerce backend — `checkout`, `payment`, `inventory`, and `notification` services — and ran it through a realistic failure scenario: a bad deploy that shrinks a database connection pool, which then causes payment errors under load.

Here's what the pipeline does, end to end:

**1. Detection.** Every service streams structured logs into Kafka. A stream processor computes a rolling error rate per service. When `payment-service`'s error rate jumps from 0.2% to 14% in three minutes, a rule-based anomaly detector fires.

**2. Cooldown check.** Before anything expensive happens, the system checks Redis: has this exact anomaly already triggered an investigation in the last 5 minutes? If so, skip — no point re-analyzing the same fire twice.

**3. Investigation.** An AI agent wakes up. It pulls the last 500 log lines from the affected window, and — this is the part I think is actually interesting — searches a vector store of *past* incidents for anything similar. It finds two near-identical incidents from the last 90 days.

**4. Diagnosis.** All of that context goes to Claude, with instructions to return a strict, structured JSON response — no rambling, no markdown, just fields a UI can render directly:

- **Probable cause**: DB connection pool reduced from 50 to 20 in the latest deploy
- **Affected services**: payment-service, and checkout-service downstream
- **Recommended action**: revert the deploy, or raise the pool size back to 50
- **Confidence**: 72%

**5. Delivery.** The on-call engineer opens one page. Instead of a wall of dashboards, they read a diagnosis that's already correlated the deploy timing, the error pattern, and institutional memory of past incidents — before they've had their coffee.

Total time from anomaly to readable diagnosis: under 60 seconds in testing.

## The part that matters more than the AI call

Anyone can wire an API to an LLM. The part I spent the most time on was making sure the AI layer never becomes a liability:

- **A circuit breaker sits around every call to Claude.** If the API times out or fails, the incident is still created — with the raw logs attached — just without an AI summary. The pipeline degrades gracefully instead of breaking.
- **The AI never gets to ramble.** It's constrained to a strict JSON schema. If the response doesn't parse, that's treated identically to a circuit-breaker failure — one retry, then fall back. No infinite retry loops, no surprises.
- **A dedup/cooldown layer means the AI isn't called on repeat.** During a noisy incident where the same anomaly might fire repeatedly, this alone keeps both cost and noise under control.

None of this is exotic engineering, but it's the difference between an AI feature that's a fun demo and one that's actually trustworthy enough to page a human based on.

## Making it reproducible, not a lucky capture

One problem with demoing an anomaly-detection system: real bugs don't show up on command. So the mock e-commerce backend behind these examples has a deliberate fault-injection layer — each service exposes an endpoint to toggle a specific failure mode for a set duration, then auto-revert:

- **DB pool exhaustion** (payment-service) — the exact scenario walked through above
- **Latency spike** (inventory-service) — artificial delay injected into a percentage of requests, to test p95-based detection instead of pure error-rate detection
- **Error burst** (any service) — straightforward elevated 5xx rate, the simplest threshold case
- **Downstream timeout** (notification-service) — simulates a hanging dependency, useful for testing whether the AI diagnosis correctly identifies *which* service is the actual source versus which ones are just showing symptoms

This means every incident in this post — and every one in the dashboard screenshots — is something I can trigger on demand and re-run, not something I got lucky catching once.

## What's next

Right now this is rule-based anomaly detection only — static thresholds and rolling z-scores, deliberately not ML-based yet. The roadmap includes real cloud deployment (Terraform + Kubernetes on AWS), autoscaling tied to Kafka consumer lag, and eventually a tool-using agent that can query live metrics instead of working from a static log pull.

If you're curious about the architecture or want to poke at the code, [it's open source] — I'd rather get real feedback on whether AI-generated incident diagnoses are actually useful than guess.

---

*Sentinel is a portfolio project exploring distributed systems, DevOps/cloud engineering, and applied AI integration. Feedback, especially skeptical feedback, is welcome.*
