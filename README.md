# AI Lead-Sifting & Segmentation Engine

An automated data-cleaning, classification, and routing pipeline for inbound leads. This workflow ingests raw contacts from multiple sources (forms, webhooks, or cold email platforms), standardizes the payloads using JavaScript, runs an AI-driven triage using **Google Gemini** to score value intent, and splits the data into segmented Google Sheets for targeted bulk email campaigns and warmup sequences.

---

## Workflow Architecture

```
[Form / DataCollect] ──┐
                       ├──► [DataCleanup (JS)] ➔ [Classifier (Gemini)] ➔ [Merge Data (JS)] ➔ [Classification Rules] ──► [HighValue Sheet]
[Webhook Trigger]   ──┘                                                                                               ──► [Standard Sheet]
                                                                                                                      ──► [Unqualified Sheet]

```

### Execution Steps

1. **Ingestion Phase:** Accepts lead data via a custom data collection form (`DataCollect`) or through real-time webhooks hooked into outbound systems like Instantly or Smartlead.
2. **Sanitization (`DataCleanup`):** A custom JavaScript node maps uneven inbound webhook schemas into a single standardized object, strips bad spacing, and fixes case formatting.
3. **Intent Profiling (`Classifier`):** Passes the normalized data to the AI Agent. The **Google Gemini** model evaluates company metrics, offer descriptions, or custom text fields to score lead quality.
4. **Context Merging (`Code in JavaScript`):** A secondary script injects the AI-generated tier classification tag directly into the baseline customer object, ensuring no data loss occurs during routing.
5. **Dynamic Routing (`Classification`):** A native n8n Rules node switches paths based on the evaluation value, separating outputs into three buckets: `HighValue`, `Standard`, or `Unqualified`.
6. **Targeted Database Entry:** Appends data to separate Google Sheets tables tailored for different outbound treatment tracks.

---

## Node Configurations

### 1. Inbound Triggers

* **DataCollect:** Set up as an n8n Form or internal payload listener.
* **Webhook:** Configured as a `GET` or `POST` listener to catch live data events from your CRM or lead providers.

### 2. Cognitive Layer (AI Agent)

* **Model:** Google Gemini.
* **Triage Constraints:** The agent evaluates the lead's parameters against your Ideal Customer Profile (ICP). Configure the agent to output *only* one of the three core taxonomy tags:
* `HighValue` (e.g., Enterprise targets, high budget, high urgency)
* `Standard` (e.g., Good fit, mid-market, standard request)
* `Unqualified` (e.g., Spam submissions, wrong geography, out-of-scope requests)



### 3. Data Merging Script (`Code in JavaScript`)

This node bridges the structural gap between the AI output and the initial data collection payload. It merges the incoming classification tag back with the original fields (`email`, `company`, `name`, etc.) so that the final row appends completely into the destination sheet.

```javascript
// Example implementation pattern
return [{
    json: {
        ...$.DataCleanup.item.json,
        classification: $.Classifier.item.json.output.trim()
    }
}];

```

### 4. Classification Router & Storage

* **Router Mode:** Rules
* **Conditions:** * Path 0: `{{ $json.classification }} = HighValue` ➔ Appends to High-Value Master List.
* Path 1: `{{ $json.classification }} = Standard` ➔ Appends to Nurture Sequence List.
* Path 2: `{{ $json.classification }} = Unqualified` ➔ Appends to Blacklist / Low-Priority Log.



> **Note on Error Handling:** The error branches routed from the Javascript and Google Sheets nodes are designed to prevent complete workflow execution failure in the event of missing structural fields or Google API timeout limits.

---

## Downstream Campaign Execution

By splitting leads into three explicit Google Sheets, you can connect these sheets natively back to your bulk email platform:

* **HighValue Sync:** Wired directly to personalized, low-volume, high-priority manual outreach sequences.
* **Standard Sync:** Automated into automated bulk warmup sequences, drip marketing campaigns, or generic sales flows.
* **Unqualified Sync:** Kept out of active marketing pipelines to protect domain sender reputation and minimize bounce rates.
