---
description: What workflow orchestration is, why durable execution matters, and your first Unmeshed process.
---

# 01 · Basics

## What is workflow orchestration?

Almost every real system is a chain of steps: call an API, transform the result, call another service, wait for something, notify someone. You can wire those steps together in application code, but then your code also has to handle everything around them:

* What happens if step 3 fails?
* What if the server restarts halfway through?
* Which run is stuck, and where?
* How do you retry just the failed part?

A **workflow orchestrator** takes that job away from your application. You describe the steps and how they connect, and the orchestrator runs them, remembers where each run is, retries failures and records everything.

## Durable execution in one paragraph

A workflow is **durable** when its progress survives failures. If the machine running a workflow crashes after step 3 of 6, a durable orchestrator picks it up again at step 4, with the outputs of steps 1 to 3 still available, instead of starting over or losing the run. That's the core promise of tools like [Unmeshed](https://unmeshed.io/), Netflix Conductor and Temporal.

{% hint style="info" %}
**Why this tutorial uses Unmeshed:** it was built by the team behind Netflix Conductor, it lets you define workflows as plain JSON, and it has a [free tier](https://unmeshed.io/pricing) you can follow along on. The concepts apply to any orchestrator.
{% endhint %}

## The vocabulary

| Term | Meaning |
| --- | --- |
| **Process** | A workflow definition: a named, versioned list of steps. |
| **Process run** | One execution of a process, with its own input, state and history. |
| **Step** | One unit of work: an HTTP call, a script, an LLM call, a wait, and so on. |
| **ref** | A step's unique ID inside a process. Other steps use it to read that step's output. |
| **Worker** | Your own code (Java, Python, Go, TypeScript) that Unmeshed calls to run a step. |
| **Container step** | A step that holds other steps: `PARALLEL`, `SWITCH`, `FOREACH`, `LIST`. |

## How data flows between steps

Steps pass data with `{{ }}` templates:

* `{{ context.input.userId }}` reads the process input.
* `{{ steps.<ref>.output.response }}` reads an earlier HTTP step's response body.
* `{{ secrets.myApiKey }}` reads a stored [secret](https://unmeshed.io/docs/concepts/secrets), so keys never sit in the definition.

Script steps (JavaScript or Python) receive the same data as arguments: `(steps, context) => { ... }`.

## Hands-on: your first process

We'll build a process that takes a `userId`, fetches that user and their posts from a free test API, and returns a short summary.

**1. Sign up** at [unmeshed.io](https://unmeshed.io/) (free tier, no credit card).

**2. Create a process** and paste this definition:

```json
{
  "namespace": "default",
  "name": "tutorial_01_user_summary",
  "version": 1,
  "description": "Fetch a user and their posts, then summarize",
  "steps": [
    {
      "namespace": "default",
      "name": "get_user",
      "type": "HTTP",
      "ref": "user",
      "optional": false,
      "input": {
        "method": "GET",
        "url": "https://jsonplaceholder.typicode.com/users/{{ context.input.userId }}",
        "headers": { "Accept": "application/json" }
      }
    },
    {
      "namespace": "default",
      "name": "get_posts",
      "type": "HTTP",
      "ref": "posts",
      "optional": false,
      "input": {
        "method": "GET",
        "url": "https://jsonplaceholder.typicode.com/posts",
        "headers": { "Accept": "application/json" },
        "params": { "userId": "{{ context.input.userId }}" }
      }
    },
    {
      "namespace": "default",
      "name": "summarize",
      "type": "JAVASCRIPT",
      "ref": "summary",
      "optional": false,
      "input": {
        "script": "(steps, context) => {\n  const user = steps.user.output.response;\n  const posts = steps.posts.output.response;\n  return {\n    name: user.name,\n    company: user.company.name,\n    postCount: posts.length,\n    latestTitle: posts.length ? posts[posts.length - 1].title : null\n  };\n}"
      }
    }
  ],
  "defaultInput": { "userId": 1 }
}
```

**3. Run it** with input `{ "userId": 3 }`.

**4. Open the run.** You'll see each step with its input, output and timing. Click the `summary` step to see the result:

```json
{
  "name": "Clementine Bauch",
  "company": "Romaguera-Jacobson",
  "postCount": 10,
  "latestTitle": "..."
}
```

That's the whole loop: define steps, connect them with `ref`s and templates, run, inspect.

## Try it yourself

1. Change `get_posts` to call `/todos` instead, and make `summarize` return how many todos are completed.
2. Run the process with `userId` 999 (a user that doesn't exist). Which step fails, and what does the run view show? We'll fix that properly in [03 · Retries & error handling](../03-retries-and-error-handling/README.md).

## Further reading

* [Process Definition reference](https://unmeshed.io/docs/refs/process-definition)
* [Step Definitions reference](https://unmeshed.io/docs/refs/step-definition)
* [Concepts overview](https://unmeshed.io/docs/concepts/overview)

**Next:** [02 · Steps & workers](../02-steps-and-workers/README.md)
