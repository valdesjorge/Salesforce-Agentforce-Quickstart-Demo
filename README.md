# Salesforce-Agentforce-Quickstart-Demo

A minimal, deployable Agentforce demo built on a Salesforce SDO (Sales Demo Org). This project showcases a **First Agent Notes Assistant** — an AI agent that helps sales reps capture, summarize, and follow up on meeting notes directly from Salesforce.

## What the Agent Does

After a customer call, the agent lets a sales rep:

1. **Log notes** — saves raw meeting notes as a `ContentNote` linked to an Account record
2. **Summarize notes** — produces a bulleted executive summary of key decisions, pain points, and next steps
3. **Draft a follow-up email** — generates a warm, professional follow-up email based on the notes

## Components

| Type | Name | Purpose |
|---|---|---|
| Flow | `First_Agent_Log_Account_Note` | Creates a ContentNote and links it to the Account |
| GenAiFunction | `First_Agent_Log_Account_Note` | Agent action wrapping the flow above |
| GenAiFunction | `First_Agent_Summarize_Notes` | Agent action calling the summarize prompt template |
| GenAiFunction | `First_Agent_Follow_Up_Email` | Agent action calling the follow-up email prompt template |
| GenAiPromptTemplate | `Agent_Summarize_Notes` | Prompt that summarizes notes into a bulleted digest |
| GenAiPromptTemplate | `First_Agent_Follow_Up_Email` | Prompt that drafts a follow-up email from notes |

## Prerequisites

- Salesforce CLI (`sf`) installed
- A Salesforce org with Agentforce enabled (Sales/Service Cloud with Einstein add-on, or an SDO)
- API version 66.0+

## Deploy to Your Org

```bash
# Authenticate to your org
sf org login web --alias my-org

# Deploy all components
sf project deploy start --manifest manifest/package.xml --target-org my-org
```

## Project Structure

```
force-app/main/default/
├── flows/                    # AutoLaunched flows used as agent actions
├── genAiFunctions/           # Agentforce agent actions (wrappers around flows & prompts)
├── genAiPromptTemplates/     # Einstein prompt templates
└── classes/                  # Supporting Apex (SDO tooling)
```

## Resources

- [Agentforce Developer Guide](https://developer.salesforce.com/docs/einstein/genai/guide/agent-get-started.html)
- [Salesforce CLI Reference](https://developer.salesforce.com/docs/atlas.en-us.sfdx_cli_reference.meta/sfdx_cli_reference/cli_reference.htm)
- [Salesforce DX Developer Guide](https://developer.salesforce.com/docs/atlas.en-us.sfdx_dev.meta/sfdx_dev/sfdx_dev_intro.htm)
