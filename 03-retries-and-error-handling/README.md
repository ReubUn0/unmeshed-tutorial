---
description: Make workflows survive flaky APIs, slow services and outright failures with error policies, retries and optional steps.
---

# 03 · Retries & error handling

In production, the question isn't *whether* a step fails, it's *what happens next*. Good orchestration makes that answer a configuration choice instead of custom code in every service.

## The four ways a step can go wrong

Unmeshed's [error policies](https://unmeshed.io/docs/concepts/error-policies) cover four separate situations:

| Scenario | Triggers when | Example |
| --- | --- | --- |
| **Start timeout** | A step isn't picked up in time | All workers for that step are down |
| **Response timeout** | A step was picked up but hasn't reported back | A worker hangs mid-task |
| **Completion timeout** | A step hasn't finished within a time window from when it was scheduled | An SLA of "done within 5 minutes" |
| **Fail** | A step reports failure | An API returns 500 |

For each one you choose an **action**: `RETRY` (with count, interval and optional exponential backoff), `FAIL`, or `NONE`. You can also run a separate process when it happens, for example to send an alert or undo earlier work.

## A policy, in JSON

This policy retries a failed step 3 times with exponential backoff, and runs an alert process only after all retries are used up:

```json
{
  "startTimeoutConfig": null,
  "responseTimeoutConfig": null,
  "completionTimeoutConfig": {
    "duration": 300000,
    "action": "FAIL",
    "runRequest": null
  },
  "failConfig": {
    "action": "RETRY",
    "retryCount": 3,
    "retryInterval": 2000,
    "retryBackoffExponential": true,
    "runRequestAfterRetries": true,
    "runRequest": {
      "name": "tutorial_alert",
      "namespace": "default",
      "version": null,
      "correlationId": "retry_exhausted",
      "input": {},
      "testRun": false
    }
  }
}
```

Save it as an error policy named `retry_3_backoff`, then attach it to any step:

```json
"configuration": { "errorPolicyName": "retry_3_backoff" }
```

## Hands-on: tame a flaky API

[httpbin](https://httpbin.org/) has an endpoint that randomly returns either 200 or 500. It's a perfect stand-in for a flaky dependency.

**1. Create the alert process** that runs when retries are exhausted. A single script step is enough for the tutorial; in real life this would post to Slack or Teams.

```json
{
  "namespace": "default",
  "name": "tutorial_alert",
  "version": 1,
  "steps": [
    {
      "namespace": "default",
      "name": "log_alert",
      "type": "JAVASCRIPT",
      "ref": "alert",
      "optional": false,
      "input": {
        "script": "(steps, context) => ({ alert: 'Retries exhausted', at: new Date().toISOString() })"
      }
    }
  ]
}
```

**2. Create the `retry_3_backoff` error policy** from the JSON above.

**3. Create the main process:**

```json
{
  "namespace": "default",
  "name": "tutorial_03_flaky_api",
  "version": 1,
  "steps": [
    {
      "namespace": "default",
      "name": "call_flaky_service",
      "type": "HTTP",
      "ref": "flaky",
      "optional": false,
      "configuration": { "errorPolicyName": "retry_3_backoff" },
      "input": {
        "method": "GET",
        "url": "https://httpbin.org/status/200,500,500"
      }
    },
    {
      "namespace": "default",
      "name": "nice_to_have_enrichment",
      "type": "HTTP",
      "ref": "enrichment",
      "optional": true,
      "input": {
        "method": "GET",
        "url": "https://httpbin.org/status/500"
      }
    },
    {
      "namespace": "default",
      "name": "finish",
      "type": "JAVASCRIPT",
      "ref": "done",
      "optional": false,
      "input": {
        "script": "(steps, context) => ({ ok: true })"
      }
    }
  ]
}
```

**4. Run it a few times** and open each run:

* `flaky` fails about two-thirds of the time, but the policy retries it, so most runs still succeed. The run view shows every attempt.
* `enrichment` **always** fails, but it's marked `"optional": true`, so the process continues to `done` anyway.
* Occasionally all 4 attempts fail. Then the process fails and `tutorial_alert` runs.

## Patterns worth knowing

* **Optional steps** for anything that's nice to have (analytics, enrichment, notifications).
* **Exponential backoff** for rate-limited APIs, so retries don't make the overload worse.
* **Completion timeouts as SLAs**: "if this isn't done in 5 minutes, page someone."
* **Compensation**: use `runRequest` to start a process that undoes earlier steps, such as refunding a charge when shipping fails. This is the *saga* pattern.
* **Parallel and foreach don't abandon work:** if one branch fails, Unmeshed lets the others finish and then fails the container step, so you're never left with half-finished branches.

## Try it yourself

1. Change `retryCount` to 0 and run again. How often does the process fail now?
2. Add a `FAIL` step behind a `SWITCH` that deliberately fails when the input contains `"simulateError": true`. Use it to test your alert process.

**Next:** [04 · Scheduling & batch jobs](../04-scheduling-and-batch/README.md)
