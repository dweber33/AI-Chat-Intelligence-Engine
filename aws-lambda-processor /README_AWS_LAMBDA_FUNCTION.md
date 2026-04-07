# AWS Lambda Bridge - Architecture and Implementation Guide

> `aws-lambda-processor/` · The secure translation layer between Power Automate and AWS Bedrock

---

## What This Component Does

The Lambda function is the connective tissue of the pipeline. Power Automate cannot call AWS Bedrock directly - it has no native AWS authentication, no way to manage context window limits, and no mechanism to inject a system prompt as a first-class parameter separate from the user message. The Lambda bridge solves all three.

Every time the main flow fires, it sends an authenticated HTTP POST to this function. The function validates the request, sanitises the payload, constructs the full prompt package - system instructions plus log data - and invokes the Bedrock Converse API. The AI-generated summary comes back through the same path and is returned to Power Automate as a clean JSON response ready for Teams delivery.

---

## Logic Flow

The function executes four stages in sequence. Any stage can terminate the request early with an appropriate error code. Only a request that clears all four stages produces a summary.

```
POST from Power Automate
        │
        ▼
┌─────────────────────────┐
│  Stage 1: Security      │  Validate x-api-key against AUTH_TOKEN
│                         │  → 403 Forbidden if invalid or missing
└────────────┬────────────┘
             │
        ▼
┌─────────────────────────┐
│  Stage 2: Sanitization  │  Parse JSON body, extract chat_logs
│                         │  Truncate if > 300k chars
│                         │  → 400 Bad Request if no logs present
└────────────┬────────────┘
             │
        ▼
┌─────────────────────────┐
│  Stage 3: AI Engine     │  Apply system prompt (persona + rules)
│                         │  Invoke Bedrock Converse API
│                         │  → 500 Internal Error if Bedrock fails
└────────────┬────────────┘
             │
        ▼
┌─────────────────────────┐
│  Stage 4: Delivery      │  Return 200 + summary JSON
│                         │  Power Automate posts to Teams
└─────────────────────────┘
```

---

## Stage 1 - Security

The first thing the function does is check the incoming request for a valid secret. Nothing else runs until this check passes.

```python
expected_secret = os.environ.get('AUTH_TOKEN')
provided_secret = event.get('headers', {}).get('x-api-key')

if not provided_secret or provided_secret != expected_secret:
    logger.warning("Unauthorized access attempt detected.")
    return {'statusCode': 403, 'body': json.dumps('Forbidden')}
```

The secret is stored as a Lambda environment variable (`AUTH_TOKEN`) and never appears in the function code itself. Power Automate sends it in the `x-api-key` request header on every invocation. If the header is absent, empty, or does not match the stored value, the function returns a `403` and exits. No log data is processed, no AI call is made.

**Configuring the secret in AWS**

1. Open your function in the Lambda Console
2. Select the **Configuration** tab → **Environment variables** → **Edit**
3. Add a variable with Key `AUTH_TOKEN` and your chosen secret string as the Value
4. Save

Rotate this value periodically. Update the matching value in the Power Automate main flow's HTTP action header at the same time.

---

## Stage 2 - Sanitization

Once the request is authenticated, the function extracts the payload and prepares it for the AI call.

```python
body = json.loads(event.get('body', '{}'))
chat_logs = body.get('chat_logs', '')
lookback_hours = body.get('lookback_hours', '12')

if not chat_logs:
    return {'statusCode': 400, 'body': json.dumps('Bad Request: No log data')}

if len(chat_logs) > 300000:
    chat_logs = chat_logs[-300000:]
```

Two things happen here. First, if the payload contains no log data at all, the function returns a `400` and exits - there is nothing to analyse. Second, if the log string exceeds 300,000 characters, it is truncated to the most recent 300k characters. This is the context window safety valve.

**Why truncation matters**

Claude Sonnet has a large but finite context window. Sending more text than the model can process causes the call to fail entirely. More subtly, sending more text than the model can *reason through effectively* degrades output quality - the model starts to lose coherence across very long inputs. The 300k character limit is a practical ceiling derived from testing. If you are consistently hitting it, the primary fix is tightening your collector flows to reduce noise before logs reach this function, not raising the limit.

The truncation keeps the most recent data rather than the oldest. For a shift handover brief, recency is what matters.

---

## Stage 3 - AI Engine

This stage constructs the full prompt package and calls Bedrock.

```python
system_prompt = f"""You are a Senior Executive Incident Manager..."""

messages = [{"role": "user", "content": [{"text": f"LOGS:\n{chat_logs}"}]}]

response = bedrock.converse(
    modelId='us.anthropic.claude-3-5-sonnet-20240620-v1:0',
    messages=messages,
    system=[{"text": system_prompt}],
    inferenceConfig={"maxTokens": 4096, "temperature": 0.1}
)
summary_text = response['output']['message']['content'][0]['text']
```

**The Converse API structure**

The Bedrock Converse API separates `system` from `messages` as distinct parameters. This is intentional and important. The `system` parameter contains the operational rules and persona instructions - these are treated by the model as high-authority directives that frame everything that follows. The `messages` array contains the user-turn input, which in this case is the raw log data. Keeping them separate ensures the model never conflates the instructions with the data it is analysing.

**Temperature**

`temperature: 0.1` keeps the model in analytical mode. A higher temperature introduces creative variability - useful for generative tasks, counterproductive for operational intelligence where you want the model to reason from evidence rather than interpolate. At 0.1, the output is grounded in the log content and consistent across runs against the same input.

**maxTokens**

`maxTokens: 4096` sets the ceiling on the response length. For a structured output with five sections - Executive Summary, Failed Changes, Watchlist, Lessons Learned, GAP Analysis - 4096 tokens is enough for thorough coverage without runaway verbosity. Adjust downward if your use case calls for tighter summaries.

**The `us.` prefix and cross-region inference**

The model ID begins with `us.` rather than the bare model identifier. This activates AWS cross-region inference routing. Instead of pinning the call to a single region, AWS evaluates available capacity across all supported US regions and routes to whichever has headroom. For a production intelligence pipeline running on a fixed schedule, this is the right default - it trades deterministic routing for resilience against regional throttling. If your organisation has data residency requirements that restrict cross-region processing, remove the `us.` prefix and specify a fixed region in the Bedrock client initialisation instead.

**Global client initialisation**

```python
bedrock = boto3.client('bedrock-runtime', region_name='us-east-1')
```

The Bedrock client is initialised outside the handler function, at module level. Lambda reuses the execution environment across warm invocations, which means this client is created once and reused on subsequent calls to the same container. This eliminates the connection setup overhead on every invocation and is standard practice for any Lambda function making repeated calls to a downstream service.

---

## Stage 4 - Delivery and Error Handling

```python
return {
    'statusCode': 200,
    'headers': {'Content-Type': 'application/json'},
    'body': json.dumps({'summary': summary_text})
}

except Exception as e:
    logger.error(f"Bedrock invocation failed: {str(e)}")
    return {
        'statusCode': 500,
        'body': json.dumps("Internal Server Error: AI processing failed.")
    }
```

A successful call returns a `200` with the summary text wrapped in a JSON object under the key `summary`. Power Automate's HTTP action reads `body('HTTP_AWS_Bedrock')?['summary']` to extract this and pass it to the Teams post action.

The `try/except` around the Bedrock call catches any failure - throttling, model unavailability, malformed response - and returns a `500` with a diagnostic message rather than letting the Lambda function crash silently. The error is also written to CloudWatch Logs via `logger.error()`, which means you have a searchable audit trail for every failure without any additional instrumentation.

---

## Full Function Reference

```python
import json
import os
import boto3
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

bedrock = boto3.client('bedrock-runtime', region_name='us-east-1')

def lambda_handler(event, context):

    # Stage 1: Security
    expected_secret = os.environ.get('AUTH_TOKEN')
    provided_secret = event.get('headers', {}).get('x-api-key')
    if not provided_secret or provided_secret != expected_secret:
        logger.warning("Unauthorized access attempt detected.")
        return {'statusCode': 403, 'body': json.dumps('Forbidden: Invalid or missing security token')}

    # Stage 2: Sanitization
    try:
        body = json.loads(event.get('body', '{}'))
        chat_logs = body.get('chat_logs', '')
        lookback_hours = body.get('lookback_hours', '12')
        if not chat_logs:
            return {'statusCode': 400, 'body': json.dumps('Bad Request: No log data')}
        if len(chat_logs) > 300000:
            chat_logs = chat_logs[-300000:]
    except Exception as e:
        return {'statusCode': 400, 'body': json.dumps(f'Bad Request: {str(e)}')}

    # Stage 3: AI Engine
    system_prompt = f"""You are a Senior Executive Incident Manager for the Enterprise IT Account.
    Your job is to filter noise, use deep operational intelligence, and present a highly polished
    executive summary. You must use deductive reasoning to elevate systemic threats.

    Reporting Window: Last {lookback_hours} Hours.

    [Insert your domain-specific exclusion rules, inclusion rules, and output format here.
    See aws-lambda-processor/README.md - Prompt Engineering section for templates.]"""

    messages = [{"role": "user", "content": [{"text": f"LOGS:\n{chat_logs}"}]}]

    try:
        response = bedrock.converse(
            modelId='us.anthropic.claude-3-5-sonnet-20240620-v1:0',
            messages=messages,
            system=[{"text": system_prompt}],
            inferenceConfig={"maxTokens": 4096, "temperature": 0.1}
        )
        summary_text = response['output']['message']['content'][0]['text']

        # Stage 4: Delivery
        return {
            'statusCode': 200,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps({'summary': summary_text})
        }

    except Exception as e:
        logger.error(f"Bedrock invocation failed: {str(e)}")
        return {'statusCode': 500, 'body': json.dumps("Internal Server Error: AI processing failed.")}
```

---

## Deployment Checklist

- [ ] Lambda function created in your target AWS region
- [ ] `AUTH_TOKEN` environment variable set
- [ ] IAM execution role includes `bedrock:InvokeModel` permission for the target model ARN
- [ ] Function URL or API Gateway endpoint configured and URL copied into the Power Automate main flow's HTTP action
- [ ] Matching secret value set in the Power Automate HTTP action's `x-api-key` header
- [ ] CloudWatch log group retention policy set (recommended: 30 days)
- [ ] End-to-end test run manually - trigger main flow, confirm summary posts to Teams, confirm CloudWatch shows a clean execution log

---

## Prompt Engineering

The system prompt inside Stage 3 is the only component of this function that is domain-specific. Everything else - authentication, sanitisation, the Bedrock call structure, error handling - is reusable across any deployment of this framework.

For detailed guidance on writing effective prompts for Claude Sonnet, including the Heuristic Prompting techniques developed during this project's field testing, see [`AI_Prompt_Engineering_Lessons_Learned.md`](../AI_Prompt_Engineering_Lessons_Learned.md).

---

## Relationship to the Rest of the System

This function sits between the Power Automate layer and AWS Bedrock. It has no awareness of Teams channels, SharePoint, or the collector flows. It receives logs and returns a summary - the pipeline on either side handles everything else.

For the full system overview, see the root `README.md`. For the flow that calls this function, see `flows/README_Main_Flow.md`.
