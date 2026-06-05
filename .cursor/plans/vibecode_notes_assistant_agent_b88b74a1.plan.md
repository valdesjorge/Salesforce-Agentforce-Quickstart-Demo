---
name: Vibecode Notes Assistant Agent
overview: Author the "First Agent Notes Assistant" Agentforce employee agent as an Agent Script authoring bundle, add a Find_Account_By_Name resolution Flow, then validate and publish it to the org.
todos:
  - id: flow
    content: Create Find_Account_By_Name AutoLaunched Flow (resolve by Id or by name; return matchCount, accountId, accountName, candidateList) + flowDefinition
    status: pending
  - id: scaffold
    content: Scaffold authoring bundle via 'sf agent generate authoring-bundle --no-spec' for First_Agent_Notes_Assistant
    status: pending
  - id: agentscript
    content: "Write the .agent Agent Script: config (Employee agent), variables, system, start_agent, and notes_assistant subagent with all actions + account-context reasoning"
    status: pending
  - id: standard-actions
    content: Wire and confirm standardInvocableAction identifiers for Search The Web and Answer Questions with SF Documentation
    status: pending
  - id: deploy-flow
    content: Deploy Find_Account_By_Name flow to belgium-hackathon-demo-org
    status: pending
  - id: validate
    content: Validate the authoring bundle (sf agent validate authoring-bundle)
    status: pending
  - id: publish
    content: Publish the authoring bundle, activate, and smoke-test with sf agent preview
    status: pending
  - id: commit
    content: Commit the new flow + authoring bundle + retrieved agent metadata
    status: pending
isProject: false
---

# Vibecode the First Agent Notes Assistant

Build the agent as an **Agent Script authoring bundle** (`aiAuthoringBundle`), wiring the three existing actions plus a new account-resolution Flow and the two standard research actions. Then validate, publish, and smoke-test.

## What already exists (reuse, do not rebuild)
- Flow `flow://First_Agent_Log_Account_Note` - inputs `varAccountID`, `varNoteBody`, `varNoteTitle`; output `varIsSuccess`
- Prompt `Agent_Summarize_Notes` - inputs `NotesText`, `AccountName`
- Prompt `First_Agent_Follow_Up_Email` - inputs `NotesText`, `Account_Name`

## What's new
- **Flow `Find_Account_By_Name`** (AutoLaunched) - resolves an Account by Id (from record context) OR by name (from the user), returning enough to disambiguate.
- **Authoring bundle** `First_Agent_Notes_Assistant` containing the `.agent` Agent Script.

## 1. Find_Account_By_Name Flow
New AutoLaunched flow at `force-app/main/default/flows/Find_Account_By_Name.flow-meta.xml` (+ matching `flowDefinitions` entry).
- Inputs: `accountId` (string, optional), `accountName` (string, optional)
- Logic: if `accountId` is set, Get the Account by Id; else Get Accounts where `Name LIKE %accountName%` (limit ~5).
- Outputs: `matchCount` (number), `accountId` (string), `accountName` (string), `candidateList` (string - newline list of "Name - City" for disambiguation)

This single flow serves both branches of the account-context rule.

## 2. Authoring bundle + Agent Script
Scaffold with `sf agent generate authoring-bundle --no-spec` (creates `force-app/main/default/aiAuthoringBundles/First_Agent_Notes_Assistant/` with a `.bundle-meta.xml` + `.agent`). Then hand-write the `.agent` per `docs/Agent Script Rules & Guide.md`:

- `config`: `developer_name: "First_Agent_Notes_Assistant"`, `agent_type: "AgentforceEmployeeAgent"`, label/description.
- `variables`:
  - `current_record_id: linked id` (source = current record context - confirm exact source token at build time)
  - `selected_account_id: mutable id = ""`, `selected_account_name: mutable string = ""`
  - `notes_text: mutable string = ""`, `account_resolved: mutable boolean = False`
- `system`: welcome/error messages + global instructions (note-taking assistant persona).
- `start_agent agent_router`: transition to `notes_assistant`.
- `subagent notes_assistant` with `actions` (targets) and `reasoning`:
  - `find_account_by_name` -> `flow://Find_Account_By_Name`
  - `log_account_note` -> `flow://First_Agent_Log_Account_Note` (slot-fill `varNoteTitle` as "Meeting Notes - {name} - {date}")
  - `summarize_notes` -> `prompt://Agent_Summarize_Notes`
  - `draft_follow_up_email` -> `prompt://First_Agent_Follow_Up_Email`
  - `search_web` and `answer_with_sf_docs` -> `standardInvocableAction` (confirm exact identifiers at build time)
  - `reasoning.instructions` implement the account-context logic (id present -> resolve by id; else ask name -> resolve -> disambiguate on `matchCount > 1`), then collect notes and expose the action set.

## 3. Deploy, validate, publish
- Deploy the new Flow first: `sf project deploy start -d force-app/main/default/flows/Find_Account_By_Name.flow-meta.xml -o belgium-hackathon-demo-org`
- Validate: `sf agent validate authoring-bundle --api-name First_Agent_Notes_Assistant`
- Publish: `sf agent publish authoring-bundle --api-name First_Agent_Notes_Assistant -o belgium-hackathon-demo-org` (compiles, creates Bot/BotVersion/GenAiX, retrieves back)
- Activate + smoke-test: `sf agent activate` then `sf agent preview`.

## Open items to confirm during build (validate will surface these)
- Exact `source:` token for the `current_record_id` linked variable.
- Exact `standardInvocableAction` identifiers for "Search The Web" and "Answer Questions with Salesforce Documentation".
- Whether to target the prompt templates directly (`prompt://`) vs the existing GenAiFunctions - plan assumes direct prompt targets.

## Note on the existing draft
The current builder draft ("Notes & Use Cases Assistant" subagent) won't be edited; publishing this bundle creates a clean, source-controlled version of the agent. We can retire the draft afterward.