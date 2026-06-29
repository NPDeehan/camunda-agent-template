# Camunda Agent — Starter Template

![Process diagram](img/processimage.png)

This is a **ready-to-deploy starter template** for building a conversational AI agent on Camunda 8. If you have never built a Camunda agent before and want a working starting point — this is it.

The template ships with:

- A **Camunda Tasklist** interface for submitting requests and viewing responses.
- Built-in BPMN-native abilities to **wait for a user reply** or **pause for a period of time** before continuing — no custom code required.
- An **AI agent** backed by Claude Sonnet 4.6 on AWS Bedrock, wired up inside an ad-hoc sub-process.

You replace the sample domain logic with your own, keep what you need, and delete the rest.

---

## How it works

A user submits a request via Camunda Tasklist. The AI agent runs inside a BPMN ad-hoc sub-process and can call a set of **tools** — BPMN elements (service tasks, script tasks, timer events) that the agent chooses at runtime. The agent sends its answer back through the process, then waits for a follow-up reply before looping or finishing.

```
User submits request via Tasklist
        │
        ▼
Start event fires process instance
        │
        ▼
┌───────────────────────────────────────────┐
│  AI Agent (ad-hoc sub-process)            │
│  Model: Claude Sonnet 4.6 on Bedrock      │
│                                           │
│  Available tools:                         │
│  ├─ Create and send a message             │
│  ├─ Get current time / date               │
│  └─ Wait for a set duration              │
└───────────────────────────────────────────┘
        │
        ▼
Agent sends reply → process waits for follow-up response
        │
        ├─ User responds → agent loops with follow-up
        └─ Timeout → process ends
```

---

## Prerequisites

Before starting you will need:

- A **Camunda 8 SaaS** account with a cluster on **version 8.9 or later**.
  Sign up at [camunda.com](https://camunda.com) if you do not have one. (If you're a Camundi, you don't need to do this — we already have a cluster for you and you can ask to be invited.)
- An **AWS account** with Amazon Bedrock enabled and access to the model `us.anthropic.claude-sonnet-4-6` in your chosen region.
  *(May not be needed if you are using a Camunda-provided development cluster — see Step 5.)*

---

## Step 1 — Add the BPMN to your Camunda project

1. Log in to [Camunda Web Modeler](https://modeler.camunda.io).
2. Click **Create new project** and give it a name.
3. Inside the project, click **Add file** → **Browse blueprints**. Search for **Camunda Agent Starter** and select it. The BPMN will be added to your project automatically.

---

## Step 2 — Configure the model

Once the BPMN is loaded, make these three changes in Web Modeler before doing anything else. They must be unique to your deployment, especially if you are sharing a cluster with others.

### 2a. Set a unique Process Name and ID

1. Click on an empty area in the canvas (not on any element) to select the process itself. Make sure you're on the **Implement** tab at the top — there are three tabs: Design, Implement, and Play.
2. In the properties panel (named **Details**), update **Name** to something descriptive and unique, e.g. `Alice's Support Agent`.
3. Update **ID** to a unique identifier using only letters, numbers, and hyphens, e.g. `alice-support-agent`.

If two people deploy this template to the same cluster with the default ID `Camunda-Agent-Template`, their deployments will overwrite each other.

### 2b. Update the Version Tag

The **Version Tag** identifies what this agent does. It must start with `AGENT` followed by a short sentence explaining the agent's goal.

1. With the process still selected (empty canvas area), find the **Version Tag** field in the properties panel.
2. Replace the default value with your own description in this format:
   ```
   AGENT - A short sentence explaining what your agent does
   ```
   For example: `AGENT - Helps customers track and manage their support tickets`

This tag is used to identify the agent's purpose at a glance and is required for the agent to be recognised correctly by the runtime.

---

## Step 3 — Connect your cluster to the project

Before you can deploy, you need to link your Camunda cluster to the Web Modeler project.

1. Click the **back arrow** to return to the project view (the list of files in your project).
2. In the left sidebar, click **Connected clusters**.
3. In the dropdown, select the cluster you want to deploy to.

The cluster is now set as the deployment target for every BPMN in this project.

---

## Step 4 — Deploy the process

1. Open `Camunda Agent Builder Template V2.bpmn` in Web Modeler.
2. Click **Deploy** in the top-right corner and confirm.

> **Expected warnings:** You may see warnings about secrets that do not exist yet (`AWS_…`). This is normal — the secrets will be added in the next step. The deployment will still succeed.

---

## Step 5 — Configure AWS credentials *(skip if you have been given access to a demo cluster)*

> **Using a Camunda-provided development cluster?** Dev clusters often come with Bedrock model access pre-configured at the cluster level. Try running the agent first — if it fails to invoke the model, come back here and add the AWS secrets manually.

🤖 Not needed if you're using the Camunda Bot — it's already set up.

If you are using your own cluster or the agent cannot reach Bedrock, you need an IAM user or role with `bedrock:InvokeModel` permission on `us.anthropic.claude-sonnet-4-6`. See the AWS guide [Creating IAM users](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_users_create.html) if you need to create one (choose **Application running outside AWS** as the use case).

> **Add to your Camunda cluster** — Log in to [Camunda Console](https://console.camunda.io) → your cluster → **Secrets** tab → **Create new secret**. Create all three:
> - **Name:** `AWS_REGION` — **Value:** your Bedrock-enabled region, e.g. `us-east-1`
> - **Name:** `AWS_ACCESS_KEY` — **Value:** the IAM access key id
> - **Name:** `AWS_SECRET_KEY` — **Value:** the IAM secret access key
>
> 🤖 Camunda employees: ask the Camunda Slack bot to create these secrets in the cluster for you.

---

## Step 6 — Test it

Before testing, re-deploy the process to pick up all the configuration changes made since Step 4. Open the BPMN in Web Modeler and click **Deploy** again.

1. Go to [Camunda Tasklist](https://tasklist.camunda.io) and log in to the same cluster.
2. Start a new process instance using your process name and submit a request. A good first test exercises both built-in capabilities at once:
   ```
   In 30 seconds can you respond to me with the current time in Jakarta?
   ```
   This tests the timer wait and the current time tool together.
3. You should see a process instance appear in Camunda **Operate** and a response arrive via the agent.
4. Reply to continue the conversation — the process is waiting for it.

---

## Built-in BPMN capabilities

The template includes two built-in agent tools that are implemented entirely in BPMN — no external service required.

### Get the current time and date

The agent has access to a **Get the Current Time and date** tool, which is a BPMN script task that evaluates `now()` and returns the current timestamp. The agent uses this whenever it needs to reason about time, schedule actions, or tell the user what time it is.

### Wait for a duration

The agent has access to a **Wait for Next Communication** tool, which is a BPMN timer intermediate event. When the agent calls this tool it passes an [ISO 8601 duration](https://en.wikipedia.org/wiki/ISO_8601#Durations) string (e.g. `PT30M` for 30 minutes, `P1D` for one day). The process pauses for exactly that duration using Camunda's native timer, then automatically resumes.

This is useful for agents that check back later, send reminders, or implement scheduled follow-ups — with no external scheduler or custom code.

---

## Customising the template

### Swap in your own agent logic

The agent's system prompt is on the **AI Agent (ad-hoc sub-process)** element. Click it in Web Modeler and update the `System Prompt` input to describe what your agent should do.

### Add or remove tools

Any BPMN element inside the ad-hoc sub-process is a potential tool. Add a new service task (HTTP connector, script task, etc.) and give it a clear **Documentation** description — that description becomes the tool description the LLM sees. Remove sample tools you do not need by selecting them and pressing Delete.

### Change the model

The AI Agent task currently points at `us.anthropic.claude-sonnet-4-6` on AWS Bedrock. You can change this to any other Bedrock model, or switch the provider entirely to OpenAI, Anthropic direct, or Azure OpenAI by changing the **Provider type** input on the same task.

---

## File reference

| File | Purpose |
|------|---------|
| `Camunda Agent Builder Template V2.bpmn` | The main process definition — AI agent, timer events, message handling, and conversation flow. |
