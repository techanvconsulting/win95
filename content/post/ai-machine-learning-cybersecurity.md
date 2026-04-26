+++
author = "Anubhav Gain"
title = "AI in Cybersecurity: What Actually Works vs What Is Marketing"
date = "2026-03-25"
description = "Every security vendor claims AI. Most mean regex with a gradient boost wrapper. Here is an honest breakdown of where ML actually outperforms rules — and where it does not."
tags = ["ai", "machine-learning", "security", "detection", "engineering"]
categories = ["ai", "security"]
+++

Walk into any security vendor booth and you will hear one word repeated until it loses meaning: AI. In most cases, the reality is a gradient-boosted decision tree on the same log data your SIEM has ingested for a decade, wrapped in a dashboard that generates a confidence score nobody knows how to interpret.

<!--more-->

ML genuinely solves problems rules cannot. But the conflation of "we use AI" with "we solve your security problem" drives buying decisions based on marketing language instead of engineering merit.

---

## The AI Security Hype Problem

When a vendor says "AI-powered," they usually mean one of three things: (1) rule-based logic with an ML label — static signatures and regex marketed as intelligent detection; (2) a single classifier bolted onto an otherwise traditional product; or (3) actual ML pipelines with real feedback loops, which is rare.

The marketing language is identical across all three. Ask: What is the model architecture? What features does it consume? How is it retrained? What is the false positive rate at your target recall? Vendors who answer clearly are worth evaluating. Vendors who pivot to case studies are not.

---

## Where ML Genuinely Wins

**UEBA and anomaly detection.** Normal behavior has a statistical signature — login times, data access patterns, lateral movement velocity. Rules cannot enumerate every variant of anomalous; a behavioral model scores deviations continuously without needing a rule for each one.

**Alert correlation.** An attacker's intrusion chain might touch twenty log sources across six hours. Rules produce forty separate low-confidence alerts. An ML correlation model assembles them into one incident with a reconstructed timeline — analyst workload reduction that is real and measurable.

**NLP-based phishing detection.** Attackers rotate domains and reword lures faster than signatures can follow. Language models trained on email semantics and brand impersonation patterns catch novel campaigns that have never appeared in any threat feed.

---

## Where Rules Still Win

**Compliance checks.** Whether MFA is enforced or a service account has admin rights is a binary fact. A rule checks it in microseconds with zero false positives — no distribution to model.

**Known-bad signature matching.** IOC lookups — file hashes, domains, IPs — are deterministic and auditable. ML adds no value and introduces interpretability problems you don't want in a forensic context.

**Audit logging.** Compliance frameworks do not accept "the model assessed this as likely benign." They require a reproducible determination. Rules produce that. Models do not.

---

## Supervised vs Unsupervised Learning

**Supervised learning** requires labeled attack samples and generalizes well when the training distribution matches production. The problem: attack distributions shift constantly. A malware classifier trained eighteen months ago has degraded coverage on current families, and retraining pipelines are often skipped — so the "AI" you bought becomes a historical artifact.

**Unsupervised learning** — isolation forests, autoencoders — builds a model of normal and scores deviations without labels. More robust to distribution shift, but harder to interpret: "anomaly score 0.87" gives an analyst nowhere to start.

The strongest pipelines combine both: unsupervised to surface anomalies and generate weak labels, supervised to produce actionable outputs from those labels.

---

## The False Positive Problem

A model with 95% recall and 5% false positive rate sounds excellent in a paper. At 100,000 events per day, that is 5,000 false positives daily. No SOC triages that volume. The model gets tuned to near-uselessness or switched off.

Demand one specific metric from vendors: false positive rate at your target recall, measured in your environment, not on a benchmark dataset. Any vendor that cannot provide this number is not ready for production.

---

## LLMs in Security: Help vs Hype

LLMs are genuinely useful for a narrow set of tasks: summarizing threat actor reports into operational briefs, flagging insecure code patterns in pull requests, and drafting initial incident narratives from raw log data.

Where they fail: autonomous detection and response. LLMs hallucinate and have no concept of your environment's behavioral baseline. Giving an LLM authority to block traffic or close incidents without human review is how you create availability incidents in the name of security. Anyone selling autonomous LLM response is selling the demo, not the product.

---

## Building Your Own ML Security Tooling

Start with structured, normalized logs — raw syslog is not features. Normalize authentication events, DNS queries, and network flows into consistent schemas using Logstash or Vector.

Start simple: scikit-learn's Isolation Forest on authentication anomalies is a reasonable first project. Avoid neural architectures until you have a baseline — complexity rarely earns its cost early on.

Build a retraining pipeline before you ship the model. Collect analyst feedback on alerts and retrain weekly. Training once and calling it done is the most common failure mode.

---

## TechAnv's AI Platform: Honest About What It Does

TechAnv's AI-First platform uses ML for behavioral anomaly scoring, alert correlation, and NLP-assisted phishing detection — with documented false-positive profiles and retraining pipelines on your telemetry. It does not make autonomous containment decisions without analyst confirmation, does not promise zero false positives, and does not replace detection engineering.

OpenArmor, TechAnv's open-source security platform, exposes the detection logic directly. Read the models, inspect the feature pipelines, validate behavior against your own data. Transparency is not a limitation — it is the only honest foundation for tooling that earns trust.

**Explore OpenArmor on GitHub, or start a conversation at [techanv.com](https://techanv.com) about AI-First security that does what it claims.**
