# Collector Flow - Real-Time Ingest Pipeline

> `code_collector.json` · Phase 1 of the Chat Intelligence Engine

---

## What This Flow Does

This is the always-on listener that starts the pipeline. The moment a message is posted to any monitored Teams channel, this flow fires, extracts the relevant content, and writes an anonymised record to the SharePoint buffer that the analysis layer reads from.

Nothing else in the system runs until this flow has done its job. The quality of every AI summary downstream depends on the completeness and cleanliness of what this flow writes to the buffer.

---

## How It Works

The flow is event-driven - it does not poll or run on a schedule. It registers as a webhook with the Teams API and receives a push notification in real time when a new message arrives.

Three actions execute in sequence after the trigger fires.

**Action 1 - Get message details**

Fetches the full metadata of the triggering message: the plain text body, the subject line, the timestamp, and the sender's department. This is the primary data extraction step.

**Action 2 - Get channel details**

Uses the `channelId` from the trigger to resolve the human-readable display name of the channel. This becomes the `ChatID` field in the buffer - it is how the analysis layer knows which operational stream a message came from.

**Action 3 - Create item**

Writes the extracted, anonymised record to the SharePoint buffer list. The message subject and body are concatenated into a single `MessageBody` field, formatted for downstream token efficiency.

---

## Architecture

```
┌─────────────────────────────────────────────────────┐
│  SOURCE                                             │
│  Teams channel - any monitored group or channel     │
└───────────────────────┬─────────────────────────────┘
                        │ webhook push (real-time)
┌───────────────────────▼─────────────────────────────┐
│  COLLECTOR FLOW                                     │
│  Get message details → Get channel details          │
└───────────────────────┬─────────────────────────────┘
                        │ anonymised record
┌───────────────────────▼─────────────────────────────┐
│  SHAREPOINT BUFFER                                  │
│  Transient log store · shared across all collectors │
└─────────────────────────────────────────────────────┘
```

---

## Privacy by Design

The Microsoft Teams connector exposes the sender's display name directly. This flow deliberately does not use it.

Capturing display names would introduce PII into the buffer, the LLM prompt, and the AI-generated summary - classifying the entire pipeline as a PII-handling system and triggering obligations under GDPR, CCPA, and ISO 27001 data minimisation standards. By extracting the sender's **department** instead, the flow retains meaningful operational context (which team sent this message) without identifying any individual.

Do not modify this field mapping without a formal PII assessment for your jurisdiction.

---

## SharePoint Buffer Schema

The buffer list schema is fixed across all collector deployments, regardless of domain. Every collector - whether monitoring MIM channels, ops channels, or BI channels - writes to the same four columns.

| Column name | Type | Source |
|---|---|---|
| `Title` | Single line of text | Sender's department |
| `MessageBody` | Multiple lines of text | Subject + plain text body (concatenated) |
| `OriginalTimestamp` | Date and Time | Message created datetime |
| `ChatID` | Single line of text | Channel display name |

Create this list before importing the flow. The flow will fail silently if the column names do not match exactly.

---

## Scaling: One Collector Per Channel Group

A single collector flow can be configured to monitor multiple channels simultaneously. At low message volumes this works without issue. At high volumes, busy incident channels, large team channels with frequent activity, a single flow will begin hitting Power Automate concurrency limits. The most common symptom is duplicate entries in the SharePoint buffer.

Duplicate buffer entries are not just a data quality problem. Every duplicate is passed to the LLM during the analysis run, inflating the token count and introducing redundant noise into the prompt. This increases AWS Bedrock costs and degrades the quality of the AI output.

The safe pattern is one collector flow per logical channel group, for example, one flow for network-related channels, another for database channels, another for MIM war rooms. All flows write to the same shared buffer. The analysis layer reads from a single list regardless of how many collectors are feeding it.

---

## Deployment

**Prerequisites**

- Microsoft 365 tenant with Power Automate access
- SharePoint Online list created with the schema above
- Teams Group ID and Channel IDs for the channels you want to monitor

**Steps**

1. Download the `.zip` solution package from the `/deploy` folder
2. Go to [make.powerautomate.com](https://make.powerautomate.com) → **Solutions** → **Import solution**
3. Upload the package and authenticate your **Microsoft Teams** and **SharePoint Online** connection references when prompted
4. Open the imported flow in Edit Mode
5. Update the trigger with your organisation's Teams **Group ID** and target **Channel IDs**
6. Update the **Create item** action to point to your SharePoint site URL and buffer list name
7. Save and enable the flow

Repeat steps 1–7 for each logical channel group you want to monitor. All instances should point to the same SharePoint buffer list.

---

## Relationship to the Rest of the System

This flow is Phase 1 of the pipeline. It has no dependency on the analysis layer - it simply writes to the buffer and exits. The analysis layer (`code_main_flow.json`) is entirely responsible for reading from the buffer, deciding what time window to cover, sending the payload to Bedrock, delivering the summary, and cleaning up processed records.

For the full system overview, see the root `README.md`. For the analysis and delivery layer, see `flows/README_Main_Flow.md`.
