---
description: Ready-made workflow recipes, videos and tools to copy from. New recipes added every week.
---

# 09 · Cookbook

You've learned the building blocks. The cookbook is where you see them combined into complete, real-world workflows.

## Steal This Workflow

**Steal This Workflow** is a weekly video series where each episode builds one production-style workflow on Unmeshed in under a minute. Every episode has a full write-up with the trigger, every step and the step types used in the [Unmeshed Workflow Showcase](https://unmeshed-showcase.vercel.app/).

### Ep. 1: Webhook Reconciliation

Catch every incoming webhook, check it against the source system and repair mismatches automatically.
Uses: HTTP, JavaScript, Switch, retries (chapters [01](../01-basics/README.md), [03](../03-retries-and-error-handling/README.md), [06](../06-human-in-the-loop/README.md)).

{% embed url="https://www.youtube.com/shorts/ldGS5CaVGAQ" %}

→ [Full recipe in the showcase](https://unmeshed-showcase.vercel.app/workflows/webhook-reconciliation)

### Ep. 2: Automated Lead Enrichment

Turn a bare signup email into an enriched, AI-scored lead routed to the right rep.
Uses: Parallel, LLM, Switch, HTTP (chapters [05](../05-api-orchestration/README.md), [07](../07-ai-agent-workflows/README.md)).

{% embed url="https://www.youtube.com/shorts/aNsmn7X-au4" %}

→ [Full recipe in the showcase](https://unmeshed-showcase.vercel.app/workflows/automated-lead-enrichment)

{% hint style="info" %}
New episodes every week. Subscribe on [YouTube](https://www.youtube.com/@unmeshedio) or browse the [showcase](https://unmeshed-showcase.vercel.app/) for the latest.
{% endhint %}

## Recipes from this tutorial

| Recipe | Chapter | Patterns |
| --- | --- | --- |
| User summary | [01](../01-basics/README.md) | HTTP chaining, templates, script step |
| Activity scoring worker | [02](../02-steps-and-workers/README.md) | SDK worker in Python, TypeScript, Go, Java |
| Flaky API with alerts | [03](../03-retries-and-error-handling/README.md) | Error policy, backoff, optional steps |
| Nightly report | [04](../04-scheduling-and-batch/README.md) | Schedule, FOREACH with concurrency |
| User dashboard endpoint | [05](../05-api-orchestration/README.md) | API orchestration, PARALLEL, caching |
| Expense approval | [06](../06-human-in-the-loop/README.md) | SWITCH, WAIT, completion tokens |
| Support ticket triage | [07](../07-ai-agent-workflows/README.md) | LLM with JSON output, validation, human review |

## Tools

* [Cron → Workflow converter](https://unmeshed-showcase.vercel.app/tools/cron-to-workflow): explain any cron expression, see its next runs, and map it to a workflow.

## More examples from Unmeshed

* [API data aggregation example](https://unmeshed.io/docs/examples/data-aggregation)
* [REST API calls example](https://unmeshed.io/docs/examples/rest-api-calls)
* [All how-to guides](https://unmeshed.io/docs/how-to-guide/all-guides)
