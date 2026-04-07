# Main Flow - Analysis and Delivery Engine

> `code_main_flow.json` · Phase 2 of the Chat Intelligence Engine

---

## What This Flow Does

This is the intelligence layer of the pipeline. It runs on a schedule, reads everything the collector flows have written to the SharePoint buffer, and sends it to AWS Bedrock for AI analysis. The resulting summary is delivered to the right Teams audience and the buffer is purged, ready for the next cycle.

Where the collector flow is always-on and passive, this flow is deliberate and structured. It decides how far back to look, how to prepare the data for the LLM, what to ask the model to find, and where to send the output. Every configuration decision in this flow is an intelligence decision.

---

## How It Works

The flow executes in four stages after the scheduled trigger fires.

**Stage 1 - Lookback window evaluation**

Before touching the buffer, the flow evaluates the current day and time and sets the lookback window accordingly. The default is 12 hours. Two conditional overrides expand it automatically when the context demands a broader view. See the Intelligent Lookback Logic section below.

**Stage 2 - Data extraction and optimisation**

The flow queries the SharePoint buffer for all records within the lookback window, then passes the result through a `Select` action configured in Text Mode. This flattens the JSON array into a plain-text string, one log entry per line, stripping all JSON key overhead before the payload reaches the LLM. The format is `[ChannelName] [Timestamp] Department: MessageBody`. This step exists entirely to reduce token consumption and improve model performance.

**Stage 3 - AI analysis**

The optimised payload is sent to AWS Bedrock via an authenticated HTTP POST to the Lambda bridge. Claude Sonnet processes the logs using a Chain-of-Thought prompt structured to distinguish routine conversation from high-value signals: tribal knowledge, process gaps, undocumented fixes, and complaints that carry escalation weight. The model is prompted to reason through the logs step by step before producing output, rather than summarising them directly.

**Stage 4 - Delivery and cleanup**

The AI-generated summary is posted to the configured Teams recipient, a channel, a direct message, or a bot conversation. Processed records are then deleted from the SharePoint buffer in a concurrency-50 loop, clearing the slate for the next cycle without hitting SharePoint throttling limits.

---

## Architecture

```
┌─────────────────────────────────────────────────────────┐
│  TRIGGER                                                │
│  Recurrence · 5 AM and 5 PM EST · every day            │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│  LOOKBACK EVALUATION                                    │
│  Set varLookbackHours: -12 / -60 / -168                 │
└───────────────────────┬─────────────────────────────────┘
                        │
┌───────────────────────▼─────────────────────────────────┐
│  DATA EXTRACTION                                        │
│  Query SharePoint buffer → token-optimised flat string  │
└───────────────────────┬─────────────────────────────────┘
                        │ HTTP POST
┌───────────────────────▼─────────────────────────────────┐
│  AI ANALYSIS                                            │
│  AWS Lambda → AWS Bedrock · Claude Sonnet · CoT prompt  │
└───────────────────────┬─────────────────────────────────┘
                        │ summary output
┌───────────────────────▼─────────────────────────────────┐
│  DELIVERY + CLEANUP                                     │
│  Post to Teams → delete processed buffer rows           │
└─────────────────────────────────────────────────────────┘
```

---

## Intelligent Lookback Logic

The lookback window is the most consequential configuration decision in this flow. Getting it wrong means either missing data or reprocessing data that has already been analysed.

The flow initialises `varLookbackHours` to `-12` and then evaluates two conditions before querying the buffer.

**Monday 5 AM → 60 hours**

If the flow fires on Monday morning, the default 12-hour window would capture only the last 12 hours of Sunday night, missing everything that happened over the weekend. A 60-hour window reaches back to Friday evening, ensuring all weekend activity is captured in the Monday handover brief.

**Friday 5 PM → 168 hours**

If the flow fires on Friday evening, the lookback expands to a full 168 hours, one complete week. This generates a strategic week-in-review rather than a shift handover, appropriate for leadership audiences who want to see patterns across the full week rather than a single shift's events.

All other triggers use the 12-hour default.

These values are set in a single integer variable and are straightforward to adjust for any domain's reporting cadence. A daily ops digest might use a 24-hour default. A weekly BI summary might trigger once on Friday with a fixed 168-hour window and no conditional logic at all.

---

## AI Output Structure

The Chain-of-Thought prompt sent to Claude Sonnet is structured around explicit extraction targets rather than a general summarisation instruction. The model is directed to reason through the logs and identify:

- **Incident and activity recap** - a concise account of what was worked on during the period
- **Tribal knowledge** - fixes, commands, configuration changes, or diagnostic techniques mentioned in conversation that are not in formal documentation
- **Gap and risk signals** - recurring complaints, manual workarounds, or tool failures that indicate systemic problems in architecture or process
- **Escalation candidates** - concerns that were expressed but not formally raised, surfaced for leadership visibility

The prompt template lives in the Lambda bridge layer. See `aws-lambda-processor/README.md` for the full prompt structure and guidance on adapting it for different domains and audiences.

---

## Performance Notes

**Token optimisation**

The `Select` action's Text Mode mapping is not cosmetic. At high log volumes, the difference between a JSON array and a flat plain-text string is significant in token terms. Unnecessary JSON keys, brackets, and formatting characters compound across hundreds of log entries. Stripping them before the payload leaves Power Automate keeps costs predictable and model context focused on signal rather than structure.

**Cleanup concurrency**

The delete loop runs 50 concurrent operations. This is the practical ceiling for SharePoint Online before throttling responses begin appearing. If you observe `429` responses in the flow run history, reduce this value incrementally rather than eliminating concurrency entirely, sequential deletion at high buffer volumes will cause the flow to time out.

---

## Deployment

**Prerequisites**

- SharePoint buffer list deployed and populated by at least one active collector flow
- AWS Lambda bridge deployed and configured with your Bedrock prompt
- Teams recipient configured, channel ID, user UPN, or bot conversation reference

**Steps**

1. Download the `.zip` solution package from the `/deploy` folder
2. Go to [make.powerautomate.com](https://make.powerautomate.com) → **Solutions** → **Import solution**
3. Upload the package and authenticate your **SharePoint Online** and **Microsoft Teams** connection references when prompted
4. Open the imported flow in Edit Mode
5. Update the **Get items** action to point to your SharePoint site URL and buffer list name
6. Update the **HTTP** action with your Lambda function URL and `x-mim-secret` header value
7. Update the **Post message** action with your target Teams recipient
8. Update the **Delete** action inside the cleanup loop to point to the same SharePoint site and list
9. Review the lookback conditions and adjust the hours and trigger times if your domain uses a different cadence
10. Save and enable the flow

---

## Relationship to the Rest of the System

This flow has no awareness of individual Teams channels and no direct connection to Teams as a data source. It reads only from the SharePoint buffer that the collector flows populate. The separation is intentional, the collector layer and the analysis layer are independently deployable and independently scalable.

For the full system overview, see the root `README.md`. For the ingest layer, see `flows/README_Collector.md`. For the Lambda bridge and prompt engineering, see `aws-lambda-processor/README.md`.
