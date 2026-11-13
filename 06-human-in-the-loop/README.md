---
description: Pause a workflow until a person approves, resume it from an external event, and branch with SWITCH.
---

# 06 · Human-in-the-loop

Some decisions shouldn't be automated: large refunds, contract changes, an AI's draft reply to an angry customer. A workflow engine is the right place for these approvals because the process can **wait for hours or days without holding any resources**, then pick up exactly where it stopped.

## The building blocks

* [`SWITCH`](https://unmeshed.io/docs/refs/step-type-switch-case) decides which path to take. A script returns a case name, and `responseMapping` maps each case to a child step's `ref`.
* [`WAIT`](https://unmeshed.io/docs/refs/step-type-wait) pauses the run. Its script returns `waitUntil` (an epoch timestamp) and, optionally, a `completionToken`.
* **`UPDATE_STEP`** finishes a waiting step early by matching its completion token. It can run from any other process, for example one triggered by an "Approve" button in your app. See [Update Wait Step Using Completion Token](https://unmeshed.io/docs/how-to-guide/update-wait-step-using-completion-token).

## Hands-on: expense approval

Expenses under $100 are approved automatically. Anything larger waits for a manager, for up to 7 days.

### 1. The main process

```json
{
  "namespace": "default",
  "name": "tutorial_06_expense_approval",
  "version": 1,
  "steps": [
    {
      "namespace": "default",
      "name": "route_by_amount",
      "type": "SWITCH",
      "ref": "route",
      "optional": false,
      "input": {
        "script": "(steps, context) => context.input.amount < 100 ? 'auto' : 'manual'",
        "responseMapping": [
          { "value": "auto", "targetRef": "autoApprove" },
          { "value": "manual", "targetRef": "waitForManager" }
        ]
      },
      "children": [
        {
          "namespace": "default",
          "name": "auto_approve",
          "type": "NOOP",
          "ref": "autoApprove",
          "optional": false,
          "input": {}
        },
        {
          "namespace": "default",
          "name": "wait_for_manager",
          "type": "WAIT",
          "ref": "waitForManager",
          "optional": false,
          "input": {
            "script": "(steps, context) => {\n  const sevenDays = 7 * 24 * 60 * 60 * 1000;\n  return {\n    waitUntil: steps.__self.start + sevenDays,\n    completionToken: 'expense-' + context.input.expenseId\n  };\n}"
          }
        }
      ]
    },
    {
      "namespace": "default",
      "name": "record_decision",
      "type": "JAVASCRIPT",
      "ref": "decision",
      "optional": false,
      "input": {
        "script": "(steps, context) => ({\n  expenseId: context.input.expenseId,\n  amount: context.input.amount,\n  status: 'approved',\n  approvedAt: new Date().toISOString()\n})"
      }
    }
  ]
}
```

### 2. The approval process

Create a second process, `tutorial_06_approve`, and add an **Update Step** from the step palette:

| Setting | Value |
| --- | --- |
| Update Step By | Completion Token |
| Completion Token | `expense-{{ context.input.expenseId }}` |
| Update Step Status To | `COMPLETED` |

Your app calls this process when a manager clicks **Approve**.

### 3. Try both paths

1. Run `tutorial_06_expense_approval` with `{ "expenseId": "A1", "amount": 40 }`. It takes the `auto` branch and finishes immediately.
2. Run it with `{ "expenseId": "B2", "amount": 850 }`. It stops at `waitForManager`. You can see it under **Admin Tools → Steps Waiting for Completion Token**.
3. Run `tutorial_06_approve` with `{ "expenseId": "B2" }`. The first run wakes up and finishes `record_decision`.

{% hint style="info" %}
The waiting run uses no worker or thread while it waits. You could have thousands of open approvals and the cost is the same as having none.
{% endhint %}

## Making it production-ready

* **Rejections:** have a "Reject" button run a variant of the approval process that sets the status to failed, and attach an [error policy](../03-retries-and-error-handling/README.md) with a `runRequest` that notifies the submitter.
* **Reminders and escalation:** put a shorter `WAIT` (say 24 hours) and a reminder step in a `PARALLEL` branch next to the approval wait.
* **Audit trail:** every run records who triggered what and when, which is exactly what auditors ask for on approvals.

## Try it yourself

1. Add a third case: amounts over $5,000 go to a different `WAIT` with token `finance-{{expenseId}}`, so the finance team approves them.
2. Replace `NOOP` with an HTTP step that posts the auto-approval to a webhook such as [webhook.site](https://webhook.site/).

**Next:** [07 · AI agent workflows](../07-ai-agent-workflows/README.md)
