---
description: Run processes on a schedule, process lists of items with FOREACH, and see why this beats a crontab.
---

# 04 · Scheduling & batch jobs

## Why not just cron?

Cron is excellent at starting a command at a time. Everything after that is your problem: did it succeed, should it retry, who gets told, what if yesterday's run is still going? When the same schedule triggers a **process**, every run gets a status, a history, retries and alerts.

{% hint style="success" %}
**Free tool:** paste any cron expression into the [Cron → Workflow converter](https://unmeshed-showcase.vercel.app/tools/cron-to-workflow) to see it in plain English, its next run times, and how it looks as an Unmeshed workflow.
{% endhint %}

## Schedules in Unmeshed

[Schedules](https://unmeshed.io/docs/concepts/schedules) trigger a process using standard Unix cron (5 fields):

```
┌──────── minute (0–59)
│ ┌────── hour (0–23)
│ │ ┌──── day of month (1–31)
│ │ │ ┌── month (1–12)
│ │ │ │ ┌ day of week (0–6, Sunday = 0)
0 22 * * 1-5      → 22:00, Monday to Friday
```

Two things cron doesn't give you:

* **No overlapping runs by default.** If last night's run is still going when the next one is due, Unmeshed doesn't start a second copy unless you allow it.
* **Calendars.** Beyond cron, you can schedule from a list of dates, from code ("first working day after a public holiday"), or by combining calendars, for example "all weekdays *minus* company holidays".

## FOREACH: process a list of items

Batch jobs usually mean "do the same thing to every item". The [`FOREACH`](https://unmeshed.io/docs/refs/step-type-foreach) step takes an `inputArray`, runs its **one** child step for each item, and lets you set `concurrency` so several items run at once. Inside the child, `{{ foreach.value }}` is the current item and `{{ foreach.index }}` its position.

## Hands-on: a nightly report across all users

We'll fetch every user, then fetch each user's todos in parallel batches, and produce a completion report.

```json
{
  "namespace": "default",
  "name": "tutorial_04_nightly_report",
  "version": 1,
  "steps": [
    {
      "namespace": "default",
      "name": "list_users",
      "type": "HTTP",
      "ref": "users",
      "optional": false,
      "input": {
        "method": "GET",
        "url": "https://jsonplaceholder.typicode.com/users"
      }
    },
    {
      "namespace": "default",
      "name": "todos_per_user",
      "type": "FOREACH",
      "ref": "perUser",
      "optional": false,
      "input": {
        "inputArray": "{{ steps.users.output.response }}",
        "concurrency": 5
      },
      "children": [
        {
          "namespace": "default",
          "name": "get_todos",
          "type": "HTTP",
          "ref": "todos",
          "optional": false,
          "input": {
            "method": "GET",
            "url": "https://jsonplaceholder.typicode.com/todos",
            "params": { "userId": "{{ foreach.value.id }}" }
          }
        }
      ]
    },
    {
      "namespace": "default",
      "name": "build_report",
      "type": "JAVASCRIPT",
      "ref": "report",
      "optional": false,
      "input": {
        "script": "(steps, context) => {\n  const users = steps.users.output.response;\n  const rows = users.map((u, i) => {\n    const todos = steps['todos[' + i + ']'].output.response;\n    const done = todos.filter(t => t.completed).length;\n    return { user: u.name, done, total: todos.length, pct: Math.round(100 * done / todos.length) };\n  });\n  rows.sort((a, b) => b.pct - a.pct);\n  return { generatedAt: new Date().toISOString(), rows };\n}"
      }
    }
  ]
}
```

**Then schedule it:** create a schedule for `tutorial_04_nightly_report` with the cron expression `0 2 * * *` (every day at 02:00).

How the pieces fit:

* `FOREACH` gets the 10 users as its `inputArray` and runs `get_todos` once per user, 5 at a time.
* Each iteration gets its own reference: `todos[0]`, `todos[1]`, and so on. `build_report` reads them by index.
* If one user's request fails, the other iterations still finish, then the `FOREACH` step is marked failed. Add the `retry_3_backoff` policy from [chapter 03](../03-retries-and-error-handling/README.md) to `get_todos` and the run heals itself.

{% hint style="info" %}
**More than one step per item?** `FOREACH` takes exactly one child. For a multi-step loop body, make that child a [Sub Process](https://unmeshed.io/docs/refs/step-type-sub-process) step that runs a whole process per item.
{% endhint %}

## Running jobs on your own servers

Shell scripts that live on a specific machine can run as steps too: install an [Unmeshed agent](https://unmeshed.io/docs/agents/adding-agents-for-command-executions) on Linux, macOS, Windows or even [IBM i / AS400](https://unmeshed.io/docs/agents/ibmi-as400-agent-installation), and the schedule triggers the command there, with the same run history and retries.

## Try it yourself

1. Change `concurrency` to 1 and to 10. Compare the run durations.
2. Make `build_report` return only users below 50% completion, and add a step that alerts if that list isn't empty.

**Next:** [05 · API orchestration](../05-api-orchestration/README.md)
