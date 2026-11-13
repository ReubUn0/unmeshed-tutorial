---
description: The main step types, when to use a script step vs a worker, and how to run your own code.
---

# 02 · Steps & workers

## Pick the right step type

Unmeshed has many [step types](https://unmeshed.io/docs/refs/step-definition). You'll use these most:

| Need | Step type | Notes |
| --- | --- | --- |
| Call a REST API | [`HTTP`](https://unmeshed.io/docs/refs/step-type-http) | Templated URL, headers, params and body. |
| Small data transform | [`JAVASCRIPT`](https://unmeshed.io/docs/refs/step-type-javascript), [`PYTHON`](https://unmeshed.io/docs/refs/step-type-python), [`JQ`](https://unmeshed.io/docs/refs/step-type-jq) | Runs inline, no deployment. |
| Your own business logic | [`WORKER`](https://unmeshed.io/docs/refs/step-type-worker) | Runs in your service via an SDK. |
| Branch | [`SWITCH`](https://unmeshed.io/docs/refs/step-type-switch-case) | Picks a child step with a script. |
| Run at the same time | [`PARALLEL`](https://unmeshed.io/docs/refs/step-type-parallel) | Children run concurrently. |
| Loop over items | [`FOREACH`](https://unmeshed.io/docs/refs/step-type-foreach) | One child, optional concurrency. |
| Pause | [`WAIT`](https://unmeshed.io/docs/refs/step-type-wait) | Until a time, or until an external event. |
| Reuse a workflow | [`Sub Process`](https://unmeshed.io/docs/refs/step-type-sub-process) | Call another process as a step. |

## Script step or worker?

Use a **script step** (`JAVASCRIPT` / `PYTHON`) for short, stateless logic: reshaping JSON, computing a flag, building a message.

Use a **worker** when the logic needs your libraries, database connections or private network, or when it's big enough that you want it in version control and tests. A worker is a normal function in your own service. Unmeshed sends it work; it never needs an inbound port, because the SDK polls for tasks.

## Hands-on: write a Python worker

We'll extend the process from [chapter 01](../01-basics/README.md) with a worker that scores how "active" a user is.

**1. Create credentials.** In Unmeshed, [generate a client ID and auth token](https://unmeshed.io/docs/sdks/creating-client-credentials).

**2. Install the SDK.**

```bash
pip install unmeshed-sdk
```

**3. Write the worker.**

```python
# activity_worker.py
from unmeshed.sdk.configs.client_config import ClientConfig
from unmeshed.sdk.unmeshed_client import UnmeshedClient
from unmeshed.sdk.apis.workers.worker import Worker


def score_activity(input_dict: dict) -> dict:
    post_count = input_dict.get("postCount", 0)
    if post_count >= 10:
        level = "very active"
    elif post_count >= 3:
        level = "active"
    else:
        level = "quiet"
    return {"activityLevel": level, "score": min(post_count * 10, 100)}


config = ClientConfig()
config.set_client_id("your-client-id")
config.set_auth_token("your-auth-token")
config.set_base_url("https://your-unmeshed-host")   # from your Unmeshed account
config.set_port(443)

client = UnmeshedClient(config)
client.register_worker(
    Worker(execution_method=score_activity, name="score_activity",
           namespace="default", max_in_progress=20)
)
client.start()
```

**4. Run it** with `python activity_worker.py`. It starts polling for `score_activity` tasks.

**5. Add a `WORKER` step** at the end of `tutorial_01_user_summary` (save it as version 2):

```json
{
  "namespace": "default",
  "name": "score_activity",
  "type": "WORKER",
  "ref": "activity",
  "optional": false,
  "input": {
    "postCount": "{{ steps.summary.output.result.postCount }}"
  }
}
```

The step's `name` must match the name the worker registered. When a run reaches this step, your Python function receives the step's input and its return value becomes the step output.

{% hint style="warning" %}
**Check the output path in your run view.** Script steps put their return value under the step's output; open a finished `summary` step and copy the exact path it shows into the template above.
{% endhint %}

**6. Run the process again** and open the `activity` step to see your worker's result.

## The same worker in other languages

{% tabs %}
{% tab title="TypeScript" %}
```javascript
const { UnmeshedClient } = require('@unmeshed/sdk');

const client = new UnmeshedClient({
  baseUrl: 'https://your-unmeshed-host', port: 443,
  clientId: 'your-client-id', authToken: 'your-auth-token',
});

const scoreActivity = async (input) => {
  const n = input.postCount || 0;
  return { activityLevel: n >= 10 ? 'very active' : n >= 3 ? 'active' : 'quiet' };
};

client.startPolling([{ worker: scoreActivity, namespace: 'default', name: 'score_activity', maxInProgress: 20 }]);
```
{% endtab %}

{% tab title="Go" %}
```go
func ScoreActivity(data map[string]interface{}) map[string]interface{} {
    n, _ := data["postCount"].(float64)
    level := "quiet"
    if n >= 10 { level = "very active" } else if n >= 3 { level = "active" }
    return map[string]interface{}{"activityLevel": level}
}

// in main(): client.RegisterWorker(apis2.NewWorker(ScoreActivity, "score_activity")); client.Start()
```
{% endtab %}

{% tab title="Java" %}
```java
@WorkerFunction(name = "score_activity")
public ActivityOutput scoreActivity(ActivityInput input) {
    int n = input.getPostCount();
    String level = n >= 10 ? "very active" : n >= 3 ? "active" : "quiet";
    return new ActivityOutput(level);
}
```
{% endtab %}
{% endtabs %}

Full setup for each language: [Python](https://unmeshed.io/docs/sdks/python-sdk) · [TypeScript](https://unmeshed.io/docs/sdks/javascript-typescript-sdk) · [Go](https://unmeshed.io/docs/sdks/go-sdk) · [Java](https://unmeshed.io/docs/sdks/java-sdk)

## Try it yourself

1. Stop the worker and run the process. What state does the `activity` step sit in? Start the worker again and watch the run continue. That's durable execution in action.
2. Replace the worker with an inline `PYTHON` step that does the same scoring. Which version would you rather maintain?

**Next:** [03 · Retries & error handling](../03-retries-and-error-handling/README.md)
