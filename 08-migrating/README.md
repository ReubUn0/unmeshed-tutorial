---
description: A practical checklist for moving cron jobs and Netflix Conductor workflows onto Unmeshed.
---

# 08 · Migrating from cron & Netflix Conductor

## Part 1: from crontab to scheduled processes

### Step 1: Inventory what you have

Run this on each server to list every cron job:

```bash
for u in $(cut -f1 -d: /etc/passwd); do crontab -l -u "$u" 2>/dev/null | grep -v '^#' | sed "s/^/$u: /"; done
ls /etc/cron.d /etc/cron.daily /etc/cron.hourly 2>/dev/null
```

For each job, write down: the schedule, what it does, which server it must run on, who cares if it fails, and whether it depends on another job finishing first.

{% hint style="success" %}
Paste each schedule into the [Cron → Workflow converter](https://unmeshed-showcase.vercel.app/tools/cron-to-workflow) to check what it actually does. Teams regularly find jobs that run at a different time than everyone thought.
{% endhint %}

### Step 2: Pick a step type per job

| The job… | Becomes |
| --- | --- |
| Calls an API or webhook (`curl …`) | An [`HTTP`](https://unmeshed.io/docs/refs/step-type-http) step |
| Runs a small script | A [`PYTHON`](https://unmeshed.io/docs/refs/step-type-python) or [`JAVASCRIPT`](https://unmeshed.io/docs/refs/step-type-javascript) step |
| Runs a command that must stay on a specific server | A command run through an [Unmeshed agent](https://unmeshed.io/docs/agents/adding-agents-for-command-executions) on that server |
| Is part of your application | A [`WORKER`](https://unmeshed.io/docs/refs/step-type-worker) step in your service |

### Step 3: Add what cron never had

* An [error policy](../03-retries-and-error-handling/README.md) with retries and an alert process.
* Dependencies: if job B used to run "an hour after job A, hopefully", make B a step after A in one process, or use [cross-process dependencies](https://unmeshed.io/docs/how-to-guide/cross-process-dependencies).
* A [schedule](https://unmeshed.io/docs/concepts/schedules) with the same cron expression. Overlapping runs are blocked by default.

### Step 4: Run both, then switch off cron

Run the Unmeshed schedule alongside cron for a few cycles (with the job made safe to run twice, or with the Unmeshed version in a dry-run mode), compare results, then comment out the crontab line. Keep the old line for a week before deleting it.

## Part 2: from Netflix Conductor

Unmeshed was built by the team that created Conductor, so the concepts map closely. The official [Conductor migration guide](https://unmeshed.io/docs/authoring/conductor-migration-guide) has the details; here's the mental model.

| Conductor | Unmeshed |
| --- | --- |
| Workflow definition | Process definition |
| Task | Step |
| `taskReferenceName` | `ref` |
| `SIMPLE` task + worker | `WORKER` step + SDK worker |
| `HTTP` | `HTTP` |
| `SWITCH` / `DECISION` | `SWITCH` |
| `FORK_JOIN` / `JOIN` | `PARALLEL` (no separate join needed) |
| `DO_WHILE` | `WHILE` |
| `SUB_WORKFLOW` | Sub Process step |
| `INLINE` | `JAVASCRIPT` (or `PYTHON`) |
| `JSON_JQ_TRANSFORM` | `JQ` |
| `WAIT` / `HUMAN` | `WAIT` with a completion token |
| `${workflow.input.x}` | `{{ context.input.x }}` |
| `${someTask.output.x}` | `{{ steps.someRef.output.x }}` |

Things that get easier after the move:

* **No nesting limits.** Container steps can be nested to any depth.
* **Parallel without a join task.** `PARALLEL` waits for its children itself, and doesn't abandon branches when one fails.
* **Managed or self-hosted.** Run it in Unmeshed's cloud or your own infrastructure instead of operating Conductor's stack yourself.

### A migration order that works

1. Start with one workflow that's well understood and not business-critical.
2. Port its workers first: mostly a change of SDK import and registration code.
3. Recreate the definition, test it with the same inputs, compare outputs run by run.
4. Move traffic gradually, then repeat for the next workflow.

Read more: [Migrating from Conductor use case](https://unmeshed.io/docs/use-cases/migrating-from-conductor) · [Scheduled jobs use case](https://unmeshed.io/docs/use-cases/scheduled-jobs)

**Next:** [09 · Cookbook](../09-cookbook/README.md)
