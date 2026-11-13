---
description: Put LLM calls inside a governed workflow, with structured output, deterministic routing, guardrails and human review.
---

# 07 · AI agent workflows

LLMs are great at reading messy input and making judgment calls. They're bad at being the only thing between a customer and your production systems. The reliable pattern is to let the model **decide or draft**, and let the workflow **control what happens next**: validation, routing, retries, approvals and an audit trail.

## The pattern

```
input → LLM (structured JSON) → validate → SWITCH on the decision
                                              ├─ low risk  → act automatically
                                              └─ high risk → WAIT for a human
```

Each box is a normal step, so every model call is logged, retried on failure and replayable like any other step.

## The LLM step

The [AI LLM step](https://unmeshed.io/docs/refs/llm) calls a model through an integration you configure once (for example OpenAI). Its input has a system prompt, a user prompt with `{{ }}` templates, and generation settings. Setting `responseType` to `json_object` gives you parsed JSON instead of text, which the next steps can branch on.

## Hands-on: support ticket triage

A ticket comes in. The model classifies it and drafts a reply. Routine tickets get the reply automatically; anything urgent or negative waits for an agent to review.

**Before you start:** create a Generative AI integration in Unmeshed and note its name. We'll call it `openai-main`.

```json
{
  "namespace": "default",
  "name": "tutorial_07_ticket_triage",
  "version": 1,
  "steps": [
    {
      "namespace": "default",
      "name": "classify_and_draft",
      "type": "INTEGRATION",
      "ref": "llm",
      "optional": false,
      "configuration": { "errorPolicyName": "retry_3_backoff" },
      "input": {
        "type": "llm",
        "name": "openai-main",
        "publishProperties": {
          "allowedTokens": 600,
          "temperature": 0.2,
          "responseType": "json_object"
        },
        "messageBody": {
          "systemPrompt": "You triage customer support tickets. Reply ONLY with JSON: {\"category\": \"billing|bug|how_to|other\", \"urgency\": \"low|high\", \"sentiment\": \"positive|neutral|negative\", \"draftReply\": \"...\"}. Mark urgency high for outages, data loss, security or legal threats.",
          "userPrompt": "Customer: {{ context.input.customerName }}\nSubject: {{ context.input.subject }}\nMessage: {{ context.input.body }}"
        }
      }
    },
    {
      "namespace": "default",
      "name": "validate_output",
      "type": "JAVASCRIPT",
      "ref": "check",
      "optional": false,
      "input": {
        "script": "(steps, context) => {\n  const r = steps.llm.output.result || {};\n  const ok = ['billing','bug','how_to','other'].includes(r.category)\n    && ['low','high'].includes(r.urgency)\n    && typeof r.draftReply === 'string' && r.draftReply.length > 20;\n  return { valid: ok, needsHuman: !ok || r.urgency === 'high' || r.sentiment === 'negative', ...r };\n}"
      }
    },
    {
      "namespace": "default",
      "name": "route",
      "type": "SWITCH",
      "ref": "route",
      "optional": false,
      "input": {
        "script": "(steps, context) => steps.check.output.result.needsHuman ? 'review' : 'auto'",
        "responseMapping": [
          { "value": "auto", "targetRef": "sendReply" },
          { "value": "review", "targetRef": "humanReview" }
        ]
      },
      "children": [
        {
          "namespace": "default",
          "name": "send_reply",
          "type": "HTTP",
          "ref": "sendReply",
          "optional": false,
          "input": {
            "method": "POST",
            "url": "https://webhook.site/your-unique-id",
            "headers": { "Content-Type": "application/json" },
            "body": {
              "ticketId": "{{ context.input.ticketId }}",
              "reply": "{{ steps.check.output.result.draftReply }}"
            }
          }
        },
        {
          "namespace": "default",
          "name": "human_review",
          "type": "WAIT",
          "ref": "humanReview",
          "optional": false,
          "input": {
            "script": "(steps, context) => ({\n  waitUntil: steps.__self.start + 24 * 60 * 60 * 1000,\n  completionToken: 'ticket-' + context.input.ticketId\n})"
          }
        }
      ]
    }
  ]
}
```

{% hint style="warning" %}
Add the LLM step from the step palette the first time, so the editor fills in the exact step type and integration name for your account, then compare it with the JSON above. Use the same output path that your run view shows for the `llm` and `check` steps.
{% endhint %}

### Run it with two tickets

```json
{ "ticketId": "T-1001", "customerName": "Priya", "subject": "Where do I find invoices?",
  "body": "Hi! Loving the product. Where can I download last month's invoice?" }
```

```json
{ "ticketId": "T-1002", "customerName": "Marco", "subject": "Production is DOWN",
  "body": "All our workflows stopped 20 minutes ago and customers are complaining. Fix this now." }
```

The first should go straight to `sendReply` (check your webhook.site page). The second should stop at `humanReview`, waiting for an agent. Approve it with an Update Step on token `ticket-T-1002`, as in [chapter 06](../06-human-in-the-loop/README.md).

## Guardrails that matter

* **Structured output plus validation.** Never branch on free text. Ask for JSON, then check it in code, and treat invalid output as "needs a human".
* **Low temperature for decisions.** Use 0–0.3 when the model is classifying, higher only for creative drafting.
* **Retries on the model call.** LLM APIs rate-limit and time out; an error policy handles that without custom code.
* **Humans for high-impact actions.** Let the model draft refunds and replies, but put a `WAIT` in front of anything irreversible.
* **Everything is logged.** Each run stores the prompt inputs, model output and the path taken, so you can audit and improve your prompts.

## Try it yourself

1. Add a `PARALLEL` step that also looks up the customer's plan with an HTTP call, and include the plan in the user prompt.
2. Route `billing` tickets to a different review queue (`billing-{{ticketId}}`) than everything else.

**Next:** [08 · Migrating from cron & Netflix Conductor](../08-migrating/README.md)
