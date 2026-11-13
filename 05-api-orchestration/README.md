---
description: Combine several backend calls into one fast endpoint, run calls in parallel, and return a single response.
---

# 05 · API orchestration

## The problem

A single screen in your app often needs data from several services: profile, orders, recommendations, notifications. The usual answers are a "backend for frontend" (BFF) service full of glue code, or a frontend that makes five calls and handles five failure modes.

**API orchestration** moves that glue into a workflow: one request comes in, the orchestrator fans out to the services (in parallel where it can), merges the results and returns one response, with timeouts, retries and caching handled by configuration.

## How Unmeshed does it

* A process with `"type": "API_ORCHESTRATION"` is built to run synchronously and return its result to the caller.
* [`PARALLEL`](https://unmeshed.io/docs/refs/step-type-parallel) runs independent calls at the same time, so total latency is roughly the slowest call, not the sum.
* [API Mappings](https://unmeshed.io/docs/concepts/api-mappings) expose a process as an HTTP endpoint, or you can call it from code with the SDK's `runProcessSync`.
* Step-level caching (`useCache`, `cacheKey`, `cacheTimeoutSeconds`) avoids re-fetching data that rarely changes.

## Hands-on: a "user dashboard" endpoint

One call returns a user's profile plus their post, todo and album counts, fetched in parallel.

```json
{
  "namespace": "default",
  "name": "tutorial_05_user_dashboard",
  "version": 1,
  "type": "API_ORCHESTRATION",
  "steps": [
    {
      "namespace": "default",
      "name": "get_profile",
      "type": "HTTP",
      "ref": "profile",
      "optional": false,
      "configuration": {
        "useCache": true,
        "cacheKey": "profile_{{ context.input.userId }}",
        "cacheTimeoutSeconds": 3600
      },
      "input": {
        "method": "GET",
        "url": "https://jsonplaceholder.typicode.com/users/{{ context.input.userId }}"
      }
    },
    {
      "namespace": "default",
      "name": "fan_out",
      "type": "PARALLEL",
      "ref": "fanOut",
      "optional": false,
      "children": [
        {
          "namespace": "default",
          "name": "get_posts",
          "type": "HTTP",
          "ref": "posts",
          "optional": false,
          "input": {
            "method": "GET",
            "url": "https://jsonplaceholder.typicode.com/posts",
            "params": { "userId": "{{ context.input.userId }}" }
          }
        },
        {
          "namespace": "default",
          "name": "get_todos",
          "type": "HTTP",
          "ref": "todos",
          "optional": false,
          "input": {
            "method": "GET",
            "url": "https://jsonplaceholder.typicode.com/todos",
            "params": { "userId": "{{ context.input.userId }}" }
          }
        },
        {
          "namespace": "default",
          "name": "get_albums",
          "type": "HTTP",
          "ref": "albums",
          "optional": true,
          "input": {
            "method": "GET",
            "url": "https://jsonplaceholder.typicode.com/albums",
            "params": { "userId": "{{ context.input.userId }}" }
          }
        }
      ]
    },
    {
      "namespace": "default",
      "name": "shape_response",
      "type": "JAVASCRIPT",
      "ref": "response",
      "optional": false,
      "input": {
        "script": "(steps, context) => {\n  const p = steps.profile.output.response;\n  const todos = steps.todos.output.response;\n  const albums = steps.albums && steps.albums.output ? steps.albums.output.response : null;\n  return {\n    id: p.id,\n    name: p.name,\n    email: p.email,\n    company: p.company.name,\n    stats: {\n      posts: steps.posts.output.response.length,\n      todosOpen: todos.filter(t => !t.completed).length,\n      albums: albums ? albums.length : null\n    }\n  };\n}"
      }
    }
  ]
}
```

### Call it from code

{% tabs %}
{% tab title="TypeScript" %}
```javascript
const result = await unmeshedClient.runProcessSync({
  name: 'tutorial_05_user_dashboard',
  namespace: 'default',
  version: null,               // latest
  input: { userId: 2 },
});
console.log(result);
```
{% endtab %}

{% tab title="Python" %}
```python
from unmeshed.sdk.common.process_request_data import ProcessRequestData

request = ProcessRequestData(
    namespace="default",
    name="tutorial_05_user_dashboard",
    version=1,
    input={"userId": 2},
)
result = client.run_process_sync(request)
print(result)
```
{% endtab %}
{% endtabs %}

### What you get

* The three list calls run **at the same time** inside `fanOut`. Check the run view's timeline: they overlap.
* `profile` is **cached** for an hour per user, so repeat calls skip it entirely.
* `albums` is **optional**: if that service is down, the endpoint still answers, with `albums: null`.

## Design tips for orchestrated APIs

* Keep synchronous processes short. Anything slow (emails, reports) should start a separate async process.
* Make every non-critical call optional and decide what the response looks like without it.
* Put timeouts on every external call with a [completion-timeout policy](../03-retries-and-error-handling/README.md), so one slow service can't hang your endpoint.

## Try it yourself

1. Add a fourth parallel call to `/comments?postId=1` and include its count in the response.
2. Remove the `PARALLEL` container so the calls run one after another. How much slower is the run?

Read more: [API orchestration use case](https://unmeshed.io/docs/use-cases/api-orchestration) · [API Orchestration product page](https://unmeshed.io/products/api-orchestration)

**Next:** [06 · Human-in-the-loop](../06-human-in-the-loop/README.md)
