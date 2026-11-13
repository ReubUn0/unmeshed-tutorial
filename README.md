---
description: A free, hands-on tutorial on workflow orchestration and durable execution, from first workflow to AI agents. Every chapter has a runnable example.
---

# Workflow Orchestration Open Tutorial

**Learn workflow orchestration by building real workflows.** This free tutorial starts from "what is an orchestrator?" and ends with production patterns for retries, scheduling, API orchestration, human approvals and AI agents.

Every chapter has a **hands-on example you can run** on [Unmeshed](https://unmeshed.io/)'s free tier, using free public test APIs, so there's nothing to set up except an account.

{% hint style="info" %}
**Why Unmeshed?** It's a durable workflow orchestration platform built by the creators of **Netflix Conductor**. Workflows are plain JSON, workers can be written in Java, Python, Go or TypeScript, and there's a [free tier](https://unmeshed.io/pricing) with no credit card. The concepts in this tutorial apply to any orchestrator.
{% endhint %}

## Roadmap

| # | Chapter | You'll build |
| --- | --- | --- |
| 01 | [Basics](01-basics/README.md) | Your first process: chained API calls and a script step |
| 02 | [Steps & workers](02-steps-and-workers/README.md) | A worker in Python, TypeScript, Go or Java |
| 03 | [Retries & error handling](03-retries-and-error-handling/README.md) | A process that survives a flaky API, with alerts |
| 04 | [Scheduling & batch jobs](04-scheduling-and-batch/README.md) | A nightly report that loops over every user |
| 05 | [API orchestration](05-api-orchestration/README.md) | A fast "dashboard" endpoint with parallel calls and caching |
| 06 | [Human-in-the-loop](06-human-in-the-loop/README.md) | An expense approval that waits for a manager |
| 07 | [AI agent workflows](07-ai-agent-workflows/README.md) | Support ticket triage with an LLM and human review |
| 08 | [Migrating from cron & Conductor](08-migrating/README.md) | A checklist for moving existing jobs |
| 09 | [Cookbook](09-cookbook/README.md) | Real-world recipes and weekly videos |

## Who this is for

* **Backend developers** tired of writing retry loops, cron wrappers and glue services.
* **Platform and DevOps engineers** looking for one place to run and observe jobs across servers.
* **AI engineers** who need LLM calls to be reliable, auditable and safe in production.

You should be comfortable reading JSON and a little JavaScript or Python. No orchestration experience needed.

## How to use this tutorial

1. **Create a free account** at [unmeshed.io](https://unmeshed.io/).
2. **Go in order** if you're new; each chapter builds on the last. Experienced? Jump to any chapter.
3. **Paste the JSON** from each hands-on section into a new process, run it, and open the run view to see every step's input and output.
4. **Do the "Try it yourself" exercises.** Breaking things on purpose is the fastest way to learn how an orchestrator behaves.

## Contributing

Found a mistake or have an idea for a chapter? Open an issue or pull request on the GitHub repository linked at the top of this page.

## Resources

* [Unmeshed documentation](https://unmeshed.io/docs/start-here/getting-started)
* [Unmeshed Workflow Showcase](https://unmeshed-showcase.vercel.app/): weekly workflow recipes
* [Steal This Workflow on YouTube](https://www.youtube.com/@unmeshedio)
* [Unmeshed on GitHub](https://github.com/unmeshed) · [LinkedIn](https://www.linkedin.com/company/unmeshed-inc/) · [X](https://x.com/unmeshedio)

---

*Maintained by the team at [Unmeshed](https://unmeshed.io/). Examples use [JSONPlaceholder](https://jsonplaceholder.typicode.com/) and [httpbin](https://httpbin.org/) as free test APIs.*
