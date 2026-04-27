---
title: 'Automated Red Teaming for LLMs'
description: 'How our automated risk assessment pipeline generates adversarial prompts, applies escalating attack strategies, and produces actionable safety reports for your AI systems.'
pubDate: 'Apr 27 2026'
heroImage: '/blog-placeholder-2.jpg'
track: 'engineering'
authors: ['trustyai-team']
tags: ['red-teaming', 'safety', 'garak', 'guardrails', 'evaluation']
---

> We built an automated red teaming pipeline that generates adversarial prompts from a policy document, throws them at your model using increasingly complex attack strategies (from simple baseline to LLM-driven jailbreaks), and produces a report showing exactly where your model's safety controls fail. The same pipeline can then be re-run after deploying guardrails, giving you a measurable before-and-after comparison.

## The problem

You've deployed an LLM. You've told it to be helpful and harmless. But how do you *know* it won't produce content that violates your organization's policies? Manual red teaming is slow, expensive, and hard to repeat. Automated benchmarks exist, but most test a fixed set of prompts that don't reflect your specific policies or the diversity of real-world misuse attempts.

We wanted a pipeline that could take any organization's safety policy, automatically generate realistic attack prompts, and probe a model's defenses using increasingly complex techniques, all in a single run.

## How it works

The automated risk assessment has two phases: **prompt generation** and **security testing**. The whole flow is driven by a policy document.

### Phase 1: Policy-driven prompt generation

Everything starts with a **policy document**, a list of harm categories that describe what your AI system should not produce. If you don't have one, the system provides a default taxonomy covering categories like illegal activity, hate speech, security and malware, violence, fraud, sexually explicit content, misinformation, and self-harm.

You can also define **custom harm categories** specific to your domain. For example, a financial services organization might add categories like "executive compensation disclosure," "debt repayment negotiation," or "investment advice", things the default taxonomy wouldn't cover.

From these categories, the pipeline uses a **challenger LLM** (an uncensored model without safety constraints) to generate a diverse set of adversarial test prompts. The generation pipeline produces prompts that vary across multiple dimensions (demographic, region, writing style, and others) so that the resulting prompts simulate how real users with different backgrounds might attempt to misuse the model.

### Phase 2: Escalating attack strategies

With the generated prompts, the pipeline sends them through [Garak](https://github.com/NVIDIA/garak) (developed in collaboration with NVIDIA) using a custom harness that applies increasingly aggressive attack strategies. At each stage, only the prompts that the model *refused* carry forward to the next strategy. This means simple-to-jailbreak prompts are caught early, and expensive techniques are reserved for the hard cases.

The attack strategies escalate in the following order:

![Attack strategies in increasing order of complexity: Baseline, SPO, Translation, TAP](/ART-probes.svg)

1. **Baseline.** Sends each prompt unmodified. This establishes the model's default behavior: what it complies with and what it rejects out of the box.

2. **System Prompt Override (SPO).** Replaces the system prompt with adversarial instructions that try to override the model's safety controls. Multiple override templates are tried, including well-known jailbreak prompts like "DAN." SPO variants also apply statistical text manipulation to the user prompt, the system prompt, or both.

3. **Translation.** Translates prompts into another language (Mandarin Chinese by default) and translates responses back for classification. This tests whether the model's safety controls are language-dependent, which is a common blind spot.

4. **Tree of Attacks with Pruning (TAP).** The most complex strategy. A separate attacker LLM dynamically generates new prompts based on the target model's previous rejection responses. The attacker iteratively crafts prompts to bypass the specific safety controls it observes, producing attacks with completely different wording from the original while preserving the harmful intent.

### Judging the results

A **judge model** classifies every response from the target model into one of four categories:

- **Complied.** The model provided the harmful content. Safety controls failed.
- **Rejected.** The model refused, citing safety or policy reasons. Safety controls worked.
- **Alternative.** The model didn't directly comply but offered a redirect or partial answer.
- **Other.** The response doesn't fit the above categories.

A prompt is marked as unsafe if it received a "complied" classification under *any* strategy. The primary metric is the **Attack Success Rate (ASR)**: the percentage of test prompts that bypassed the model's safety controls. Lower is better.

## Real-world example: Qwen3 before and after guardrails

We ran the pipeline against a Qwen3 model served without any guardrails. The attack success rate was **100%**: every adversarial prompt got the model to comply. Whilst all the prompts were rejected in the baseline step, more than 50% of the harmful requests got accepted by just using a simple System Prompt Override. Nearly all remaining prompts were broken with just simple variations of SPO.

![ART report showing 100% success rate and chart for success rate by probe](/ART-report-unsafe.png)


We then deployed NVIDIA NeMo Guardrails in front of the same model endpoint with three detectors:

- A IBM Granite based HAP detector for harmful content
- A prompt injection detector to block system prompt overrides
- A language detector to block non-English input (countering the translation attack)

We re-ran the same pipeline, same prompts, now pointing at the guarded endpoint. The attack success rate dropped to **10%**. Only 4 prompts got through. The pipeline tried many more attack variations this time around, but the guardrails blocked the vast majority.

![ART report showing 10% success rate and chart for success rate by probe](/ART-report-safe.png)

The next step is to review those 4 successful attacks, figure out why they bypassed the guardrails, and tune the configuration. Then run the assessment again. This is the core loop: assess, deploy mitigations, re-assess, repeat.

## Running it

The pipeline runs on OpenShift AI and can be triggered with a single API call to EvalHub, or submitted directly with the Kubeflow Pipelines Python SDK. You can follow the pipeline execution (including logs and input/output parameters) in the Data Science Pipelines UI. Results can optionally be exported to MLflow for tracking across multiple assessment runs.

For the full setup and configuration details, see the [Evaluating AI systems](https://docs.redhat.com/en/documentation/red_hat_openshift_ai_self-managed/3.4/html/evaluating_ai_systems/index) documentation.