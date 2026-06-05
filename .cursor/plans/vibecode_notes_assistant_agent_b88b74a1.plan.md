---
name: Vibecode Notes Assistant Agent
overview: Author the "First Agent Notes Assistant" Agentforce employee agent as an Agent Script authoring bundle, add a Find_Account_By_Name resolution Flow, then validate and publish it to the org.
todos:
  - id: preflight
    content: "VERIFY: Confirm existing backing logic is active in org (Flow + Prompt Templates)"
    status: pending
  - id: standard-actions
    content: "DISCOVER: Query org for exact standardInvocableAction identifiers (Search Web, SF Docs)"
    status: pending
  - id: flow
    content: "BUILD: Create Find_Account_By_Name.flow-meta.xml + flowDefinition-meta.xml"
    status: pending
  - id: deploy-flow
    content: "DEPLOY+TEST: Deploy Find_Account_By_Name flow and verify it activates in org"
    status: pending
  - id: scaffold
    content: "SCAFFOLD: sf agent generate authoring-bundle --no-spec for First_Agent_Notes_Assistant"
    status: pending
  - id: agentscript
    content: "BUILD: Write the complete .agent Agent Script"
    status: pending
  - id: validate
    content: "TEST: sf agent validate authoring-bundle — must pass with zero errors"
    status: pending
  - id: deploy-bundle
    content: "DEPLOY: sf project deploy start --metadata AiAuthoringBundle:First_Agent_Notes_Assistant"
    status: pending
  - id: publish
    content: "PUBLISH+ACTIVATE: sf agent publish + sf agent activate"
    status: pending
  - id: smoke-test
    content: "TEST: sf agent preview --use-live-actions — test name-based path via CLI"
    status: pending
  - id: retrieve
    content: "RETRIEVE: sf project retrieve start --metadata Agent:First_Agent_Notes_Assistant"
    status: pending
  - id: commit
    content: "COMMIT: git commit flow + authoring bundle + retrieved agent metadata"
    status: pending
isProject: false
---

# Vibecode the First Agent Notes Assistant

Build the agent as an **Agent Script authoring bundle** (`aiAuthoringBundle`), wiring three existing actions plus a new account-resolution Flow. Each step below has a verification checkpoint — do not proceed past a step if its check fails.

---

## Reuse (do not rebuild)

| Component | Type | Key I/O |
|---|---|---|
| `First_Agent_Log_Account_Note` | Flow | in: `varAccountID`, `varNoteBody`, `varNoteTitle` / out: `varIsSuccess` |
| `Agent_Summarize_Notes` | Prompt Template | in: `NotesText`, `AccountName` |
| `First_Agent_Follow_Up_Email` | Prompt Template | in: `NotesText`, `Account_Name` |

---

## Step 1 — Pre-flight: verify existing backing logic is active

```bash
sf data query --json -q "SELECT DeveloperName, Status FROM Flow WHERE DeveloperName = 'First_Agent_Log_Account_Note'" -o belgium-hackathon-demo-org
sf data query --json -q "SELECT DeveloperName FROM GenAiPromptTemplate WHERE DeveloperName IN ('Agent_Summarize_Notes','First_Agent_Follow_Up_Email')" -o belgium-hackathon-demo-org
```

**Checkpoint:** Both queries return records. Flow `Status = Active`. If not, deploy the missing components before continuing.

---

## Step 2 — Discover standardInvocableAction identifiers

```bash
sf data query --json -q "SELECT Id, ActionType, DeveloperName, MasterLabel FROM GenAiFunction WHERE MasterLabel LIKE '%Search%' OR MasterLabel LIKE '%Documentation%'" -o belgium-hackathon-demo-org
```

**Checkpoint:** Note the exact `DeveloperName` values — these become the `standardInvocableAction://` targets in the agent script. If not found, these actions will be omitted from the initial build (add later).

---

## Step 3 — Build `Find_Account_By_Name` Flow

Create two files:

**`force-app/main/default/flows/Find_Account_By_Name.flow-meta.xml`** — AutoLaunched Flow:
- Inputs: `accountId` (String, optional), `accountName` (String, optional)
- Decision: if `accountId` is not blank → Get Record by Id; else → Get Records where `Name LIKE %{accountName}%` (limit 5, order by Name)
- Outputs: `matchCount` (Number), `accountId` (String), `accountName` (String), `candidateList` (String — newline-separated "Name – BillingCity" for disambiguation)

**`force-app/main/default/flowDefinitions/Find_Account_By_Name.flowDefinition-meta.xml`** — points to Active version.

**Checkpoint:** File exists at correct path. XML validates with `prettier --check`.

---

## Step 4 — Deploy the Flow and verify activation

```bash
sf project deploy start --json --metadata Flow:Find_Account_By_Name FlowDefinition:Find_Account_By_Name -o belgium-hackathon-demo-org
```

Then confirm:
```bash
sf data query --json -q "SELECT DeveloperName, Status FROM Flow WHERE DeveloperName = 'Find_Account_By_Name'" -o belgium-hackathon-demo-org
```

**Checkpoint:** Deploy exits with `status: 0`. SOQL returns `Status = Active`.

---

## Step 5 — Scaffold the authoring bundle

```bash
sf agent generate authoring-bundle --json --no-spec --name "First Agent Notes Assistant" --api-name First_Agent_Notes_Assistant -o belgium-hackathon-demo-org
```

**Checkpoint:** Directory `force-app/main/default/aiAuthoringBundles/First_Agent_Notes_Assistant/` exists with `First_Agent_Notes_Assistant.agent` and `First_Agent_Notes_Assistant.bundle-meta.xml`.

---

## Step 6 — Write the Agent Script

Edit `force-app/main/default/aiAuthoringBundles/First_Agent_Notes_Assistant/First_Agent_Notes_Assistant.agent`:

**Key design decisions:**

- `config.agent_type`: `"AgentforceEmployeeAgent"` — NO `default_agent_user` (causes publish failure)
- `currentRecordId: mutable id = ""` + `visibility: "External"` — platform auto-injects Account ID when agent opens from an Account record page (name must be exactly `currentRecordId`)
- Actions use `outputs:` blocks — required or publish fails with "Internal Error" (known platform issue)
- `require_user_confirmation` is NOT used — known platform bug where it does not trigger UI (use `available when` guards instead)

**Variables:**
```
currentRecordId: mutable id = ""   visibility: "External"
selectedAccountId: mutable id = ""
selectedAccountName: mutable string = ""
notesText: mutable string = ""
accountResolved: mutable boolean = False
```

**Actions in `notes_assistant` subagent:**
| Agent action | Target | Notes |
|---|---|---|
| `find_account` | `flow://Find_Account_By_Name` | Both Id and name branches |
| `log_account_note` | `flow://First_Agent_Log_Account_Note` | LLM slot-fills title as "Meeting Notes – {name} – {date}" |
| `summarize_notes` | `prompt://Agent_Summarize_Notes` | |
| `draft_follow_up_email` | `prompt://First_Agent_Follow_Up_Email` | |
| `search_web` | `standardInvocableAction://...` | Identifier from Step 2 |
| `answer_with_sf_docs` | `standardInvocableAction://...` | Identifier from Step 2 |

**Reasoning logic (two-phase):**
- Phase 1 deterministic: if `currentRecordId != ""` → `run find_account` with `accountId = @variables.currentRecordId` → set `selectedAccountId`, `selectedAccountName`, `accountResolved = True`
- Phase 2 LLM: if `accountResolved == True` → tell the LLM which account it's on; else → ask user for account name → LLM calls `find_account` by name → if `matchCount > 1` → show `candidateList` and ask user to pick

**Checkpoint:** File is valid YAML-like Agent Script. No tabs (spaces only, 4-space indent). All action definitions have both `inputs:` and `outputs:` blocks.

---

## Step 7 — Validate (local, no org call)

```bash
sf agent validate authoring-bundle --json --api-name First_Agent_Notes_Assistant
```

**Checkpoint:** Command exits with `status: 0` and zero errors. Fix all errors before proceeding — do not deploy a failing bundle.

---

## Step 8 — Deploy the authoring bundle

```bash
sf project deploy start --json --metadata AiAuthoringBundle:First_Agent_Notes_Assistant -o belgium-hackathon-demo-org
```

**Checkpoint:** Deploy exits with `status: 0`.

---

## Step 9 — Publish and activate

```bash
sf agent publish authoring-bundle --json --api-name First_Agent_Notes_Assistant -o belgium-hackathon-demo-org
sf agent activate --json --api-name First_Agent_Notes_Assistant -o belgium-hackathon-demo-org
```

**Checkpoint:** Both commands exit with `status: 0`. Confirm via:
```bash
sf data query --json -q "SELECT DeveloperName, Status FROM BotVersion WHERE BotDefinition.DeveloperName = 'First_Agent_Notes_Assistant'" -o belgium-hackathon-demo-org
```
Expect `Status = Active`.

---

## Step 10 — Smoke test via CLI preview (name-based path)

```bash
sf agent preview start --json --use-live-actions --authoring-bundle First_Agent_Notes_Assistant -o belgium-hackathon-demo-org
# Note the sessionId from output, then:
sf agent preview send --json --authoring-bundle First_Agent_Notes_Assistant --session-id <SESSION_ID> --utterance "I need to log meeting notes for Acme Corp" -o belgium-hackathon-demo-org
```

**Checkpoint:** Agent routes to `notes_assistant`, calls `find_account` with the name, returns account context or disambiguation list. Check trace at `.sfdx/agents/First_Agent_Notes_Assistant/sessions/<id>/traces/`.

> Note: The record-context path (`currentRecordId` injected from Account page) cannot be tested via CLI — validate that path in-org via Agentforce Studio or the native Lightning panel.

---

## Step 11 — Retrieve runtime metadata

```bash
sf project retrieve start --json --metadata Agent:First_Agent_Notes_Assistant -o belgium-hackathon-demo-org
```

**Checkpoint:** `Bot`, `BotVersion`, `GenAiPlannerBundle`, `GenAiFunction`, `GenAiPlugin` files appear under `force-app/`.

---

## Step 12 — Commit

```bash
git add force-app/main/default/flows/Find_Account_By_Name.flow-meta.xml \
        force-app/main/default/flowDefinitions/Find_Account_By_Name.flowDefinition-meta.xml \
        force-app/main/default/aiAuthoringBundles/ \
        force-app/main/default/bots/ \
        force-app/main/default/genAiFunctions/ \
        force-app/main/default/genAiPlugins/ \
        force-app/main/default/genAiPlannerBundles/
git commit -m "feat: add First Agent Notes Assistant authoring bundle and Find_Account_By_Name flow"
```

**Checkpoint:** `git status` shows clean working tree.

---

## Confirmed findings

- `currentRecordId` with `visibility: "External"` is automatically populated by the platform when the agent is opened from a Lightning record page. Name must be exactly `currentRecordId`. Confirmed by Salesforce SE.
- Employee agents must NOT have `default_agent_user` — causes "Internal Error, try again later" on publish.
- All action definitions must include an `outputs:` block — actions with only `inputs:` cause "Internal Error" on publish (platform known issue).
- `sf agent preview` cannot inject `currentRecordId` — test that path in-org only.

## Note on the existing draft

The current builder draft ("Notes & Use Cases Assistant" subagent) won't be edited. Publishing this bundle creates a clean source-controlled version. Retire the draft afterward.