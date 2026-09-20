# @pipeline — VS Code Multi-Agent Orchestration Extension
### Complete Implementation Plan · Start to End

---

## What You Are Building

A VS Code Extension that registers as `@pipeline` inside GitHub Copilot Chat.

It:
- **Auto-discovers** all your `.agent.md` files from `.github/agents/`
- Lets you **chain** them into named pipelines in any order
- **Runs** them sequentially — each agent's output becomes the next agent's input
- **Pauses** after every stage in verify mode so you can approve, edit, retry, or stop
- Works for **any project** — browser agents, research agents, coding agents, reviewers — anything you build as a `.agent.md` file

---

## Prerequisites (Install These First)

| Tool | Why | Install |
|---|---|---|
| Node.js 18+ | Runs the extension build | [nodejs.org](https://nodejs.org) |
| VS Code | Where the extension runs | Already installed |
| GitHub Copilot | Provides the LM API | Already have it |
| `yo` + `generator-code` | Scaffolds VS Code extensions | `npm install -g yo generator-code` |
| `vsce` | Packages the extension | `npm install -g @vscode/vsce` |
| TypeScript | Language | Included in scaffold |

---

## Phase 1 — Project Scaffold

### Step 1.1 — Generate the Extension

Run this in your terminal:

```bash
yo code
```

Answer the prompts exactly like this:

```
? What type of extension?     → New Extension (TypeScript)
? Extension name?             → pipeline-orchestrator
? Identifier?                 → pipeline-orchestrator
? Description?                → Multi-agent pipeline orchestrator for Copilot Chat
? Initialize git repo?        → Yes
? Bundle with webpack?        → No
? Package manager?            → npm
```

This creates the folder `pipeline-orchestrator/`. All further work is inside it.

### Step 1.2 — Install Dependencies

```bash
cd pipeline-orchestrator
npm install gray-matter     # parses .agent.md frontmatter (name, description, tools)
npm install glob            # scans folders for .agent.md files
npm install @types/node --save-dev
```

---

## Phase 2 — File Structure

After setup, your final file structure will be:

```
pipeline-orchestrator/
├── package.json                  ← Extension manifest + chat participant registration
├── tsconfig.json                 ← TypeScript config
├── src/
│   ├── extension.ts              ← Entry point, activates everything
│   ├── agent-loader.ts           ← Scans + parses .agent.md files
│   ├── pipeline-store.ts         ← Saves + loads pipeline definitions
│   ├── engine.ts                 ← Core: runs agents in sequence
│   └── commands/
│       ├── agents.ts             ← /agents command → lists available agents
│       ├── create.ts             ← /create command → interactive pipeline builder
│       ├── run.ts                ← /run command → executes pipeline
│       └── list.ts               ← /list command → shows saved pipelines
└── .vscode/
    └── launch.json               ← Debug config (auto-generated)
```

---

## Phase 3 — `package.json` (Extension Manifest)

This is the most critical file. It registers `@pipeline` as a Copilot Chat participant and defines all slash commands.

```json
{
  "name": "pipeline-orchestrator",
  "displayName": "Pipeline Orchestrator",
  "description": "Chain .agent.md files into automated pipelines inside Copilot Chat",
  "version": "1.0.0",
  "engines": { "vscode": "^1.95.0" },
  "categories": ["AI", "Chat"],
  "activationEvents": [],
  "main": "./out/extension.js",
  "contributes": {
    "chatParticipants": [
      {
        "id": "pipeline-orchestrator.pipeline",
        "fullName": "Pipeline Orchestrator",
        "name": "pipeline",
        "description": "Chain your .agent.md files into automated pipelines",
        "isSticky": true,
        "commands": [
          {
            "name": "agents",
            "description": "List all discovered .agent.md agents"
          },
          {
            "name": "create",
            "description": "Create a new pipeline by chaining agents"
          },
          {
            "name": "list",
            "description": "List all saved pipelines"
          },
          {
            "name": "run",
            "description": "Run a pipeline. Usage: /run [pipeline-name] [--verify]"
          }
        ]
      }
    ]
  },
  "scripts": {
    "compile": "tsc -p ./",
    "watch": "tsc -watch -p ./",
    "package": "vsce package"
  },
  "dependencies": {
    "gray-matter": "^4.0.3",
    "glob": "^10.3.10"
  },
  "devDependencies": {
    "@types/node": "^20.0.0",
    "@types/vscode": "^1.95.0",
    "typescript": "^5.3.0",
    "@vscode/vsce": "^2.24.0"
  }
}
```

---

## Phase 4 — `src/agent-loader.ts`

**Job:** Scan `.github/agents/` in the open workspace, parse every `.agent.md` file, return a list of agents the engine can call.

```typescript
import * as vscode from 'vscode';
import * as fs from 'fs';
import * as path from 'path';
import matter from 'gray-matter';

export interface Agent {
  id: string;           // filename without extension e.g. "browser"
  name: string;         // from frontmatter: name
  description: string;  // from frontmatter: description
  model: string;        // from frontmatter: model (default: gpt-4o)
  tools: string[];      // from frontmatter: tools
  systemPrompt: string; // body of the .md file
  filePath: string;     // absolute path to the .agent.md file
}

export async function loadAgents(): Promise<Agent[]> {
  const workspaceFolders = vscode.workspace.workspaceFolders;
  if (!workspaceFolders) { return []; }

  const agents: Agent[] = [];

  for (const folder of workspaceFolders) {
    const agentsDir = path.join(folder.uri.fsPath, '.github', 'agents');
    if (!fs.existsSync(agentsDir)) { continue; }

    const files = fs.readdirSync(agentsDir)
      .filter(f => f.endsWith('.agent.md'));

    for (const file of files) {
      const filePath = path.join(agentsDir, file);
      const raw = fs.readFileSync(filePath, 'utf-8');
      const parsed = matter(raw);

      const id = file.replace('.agent.md', '');
      agents.push({
        id,
        name:         parsed.data.name        ?? id,
        description:  parsed.data.description ?? '',
        model:        parsed.data.model       ?? 'gpt-4o',
        tools:        parsed.data.tools       ?? [],
        systemPrompt: parsed.content.trim(),
        filePath,
      });
    }
  }

  return agents;
}

export async function getAgentById(id: string): Promise<Agent | undefined> {
  const agents = await loadAgents();
  return agents.find(a => a.id === id);
}
```

---

## Phase 5 — `src/pipeline-store.ts`

**Job:** Save and load pipeline definitions to `.vscode/pipelines.json` in the workspace.

```typescript
import * as vscode from 'vscode';
import * as fs from 'fs';
import * as path from 'path';

export interface Pipeline {
  id: string;         // unique slug e.g. "research-build"
  name: string;       // display name e.g. "Research → Build"
  steps: string[];    // ordered list of agent IDs e.g. ["browser","research","coder","reviewer"]
  createdAt: string;
}

function getPipelinesFilePath(): string | undefined {
  const ws = vscode.workspace.workspaceFolders?.[0];
  if (!ws) { return undefined; }
  const dir = path.join(ws.uri.fsPath, '.vscode');
  if (!fs.existsSync(dir)) { fs.mkdirSync(dir, { recursive: true }); }
  return path.join(dir, 'pipelines.json');
}

export function savePipeline(pipeline: Pipeline): void {
  const filePath = getPipelinesFilePath();
  if (!filePath) { return; }

  const all = loadAllPipelines();
  const idx = all.findIndex(p => p.id === pipeline.id);
  if (idx >= 0) { all[idx] = pipeline; } else { all.push(pipeline); }

  fs.writeFileSync(filePath, JSON.stringify(all, null, 2));
}

export function loadAllPipelines(): Pipeline[] {
  const filePath = getPipelinesFilePath();
  if (!filePath || !fs.existsSync(filePath)) { return []; }
  try {
    return JSON.parse(fs.readFileSync(filePath, 'utf-8'));
  } catch { return []; }
}

export function getPipelineById(id: string): Pipeline | undefined {
  return loadAllPipelines().find(p => p.id === id);
}

export function deletePipeline(id: string): void {
  const filePath = getPipelinesFilePath();
  if (!filePath) { return; }
  const updated = loadAllPipelines().filter(p => p.id !== id);
  fs.writeFileSync(filePath, JSON.stringify(updated, null, 2));
}
```

---

## Phase 6 — `src/engine.ts`

**Job:** The core. Runs agents in sequence. Passes output from one to the next. Handles verify mode (pauses and waits for user response).

```typescript
import * as vscode from 'vscode';
import { Agent, getAgentById } from './agent-loader';
import { Pipeline } from './pipeline-store';

export type VerifyDecision = 'approve' | 'edit' | 'retry' | 'stop';

export interface StageResult {
  agentId: string;
  input: string;
  output: string;
  approved: boolean;
}

export interface RunOptions {
  verify: boolean;          // true = pause at each stage
  verifyStages?: string[];  // if set, only pause at these agent IDs
}

export async function runPipeline(
  pipeline: Pipeline,
  initialInput: string,
  options: RunOptions,
  stream: vscode.ChatResponseStream,
  token: vscode.CancellationToken,
  // Called in verify mode — returns user's decision and optional edited text
  onVerify: (agentId: string, output: string) => Promise<{ decision: VerifyDecision; editedOutput?: string }>
): Promise<StageResult[]> {

  const results: StageResult[] = [];
  let currentInput = initialInput;

  for (let i = 0; i < pipeline.steps.length; i++) {
    const agentId = pipeline.steps[i];
    const agent = await getAgentById(agentId);

    if (!agent) {
      stream.markdown(`\n> ⚠️ Agent \`${agentId}\` not found. Skipping.\n`);
      continue;
    }

    // Show progress
    stream.progress(`Running Stage ${i + 1}/${pipeline.steps.length}: ${agent.name}...`);
    stream.markdown(`\n---\n### 🤖 Stage ${i + 1}: ${agent.name}\n`);

    // Call the LM with this agent's system prompt
    let output = '';
    try {
      output = await callAgent(agent, currentInput, stream, token);
    } catch (err: any) {
      stream.markdown(`\n> ❌ Error in ${agent.name}: ${err.message}\n`);
      break;
    }

    const shouldVerify = options.verify &&
      (!options.verifyStages || options.verifyStages.includes(agentId));

    if (shouldVerify) {
      // Show verify prompt
      stream.markdown(
        `\n**Stage complete.** What do you want to do?\n\n` +
        `- Type \`approve\` → pass this output to next stage\n` +
        `- Type \`edit [your correction]\` → use your version instead\n` +
        `- Type \`retry\` → run this stage again\n` +
        `- Type \`stop\` → end the pipeline here\n`
      );

      let decision: VerifyDecision = 'approve';
      let finalOutput = output;

      // Wait for user decision
      const result = await onVerify(agentId, output);
      decision = result.decision;
      if (result.editedOutput) { finalOutput = result.editedOutput; }

      if (decision === 'stop') {
        stream.markdown(`\n> 🛑 Pipeline stopped at **${agent.name}**.\n`);
        break;
      }

      if (decision === 'retry') {
        i--; // re-run same step
        stream.markdown(`\n> 🔁 Retrying **${agent.name}**...\n`);
        continue;
      }

      // approve or edit — continue with finalOutput
      currentInput = finalOutput;
      results.push({ agentId, input: currentInput, output: finalOutput, approved: true });

    } else {
      // Auto mode — pass output directly
      currentInput = output;
      results.push({ agentId, input: currentInput, output, approved: true });
    }
  }

  stream.markdown(`\n---\n✅ **Pipeline complete.** ${results.length} stages ran successfully.\n`);
  return results;
}

async function callAgent(
  agent: Agent,
  input: string,
  stream: vscode.ChatResponseStream,
  token: vscode.CancellationToken
): Promise<string> {

  const models = await vscode.lm.selectChatModels({ family: 'gpt-4o' });
  if (models.length === 0) {
    throw new Error('No Copilot language model available. Make sure GitHub Copilot is active.');
  }

  const model = models[0];
  const messages = [
    vscode.LanguageModelChatMessage.Assistant(agent.systemPrompt),
    vscode.LanguageModelChatMessage.User(input),
  ];

  const response = await model.sendRequest(messages, {}, token);

  let fullOutput = '';
  for await (const chunk of response.text) {
    stream.markdown(chunk);
    fullOutput += chunk;
  }

  return fullOutput;
}
```

---

## Phase 7 — Commands

### `src/commands/agents.ts` — List all agents

```typescript
import * as vscode from 'vscode';
import { loadAgents } from '../agent-loader';

export async function agentsCommand(
  stream: vscode.ChatResponseStream
): Promise<void> {
  const agents = await loadAgents();

  if (agents.length === 0) {
    stream.markdown(
      `> No agents found.\n\n` +
      `Create \`.agent.md\` files in \`.github/agents/\` to get started.`
    );
    return;
  }

  stream.markdown(`## 🤖 Available Agents (${agents.length})\n\n`);
  stream.markdown(`*Discovered from \`.github/agents/\`*\n\n`);

  const rows = agents.map(a =>
    `| \`${a.id}\` | ${a.name} | ${a.description} |`
  ).join('\n');

  stream.markdown(
    `| ID | Name | Description |\n|---|---|---|\n${rows}`
  );
}
```

### `src/commands/list.ts` — List saved pipelines

```typescript
import * as vscode from 'vscode';
import { loadAllPipelines } from '../pipeline-store';

export async function listCommand(
  stream: vscode.ChatResponseStream
): Promise<void> {
  const pipelines = loadAllPipelines();

  if (pipelines.length === 0) {
    stream.markdown(
      `> No pipelines saved yet.\n\n` +
      `Use \`@pipeline /create\` to build your first pipeline.`
    );
    return;
  }

  stream.markdown(`## 📋 Saved Pipelines\n\n`);

  for (const p of pipelines) {
    stream.markdown(
      `### ${p.name} (\`${p.id}\`)\n` +
      `**Steps:** ${p.steps.join(' → ')}\n\n` +
      `Run it: \`@pipeline /run ${p.id}\` or \`@pipeline /run ${p.id} verify\`\n\n`
    );
  }
}
```

### `src/commands/create.ts` — Create a pipeline

```typescript
import * as vscode from 'vscode';
import { loadAgents } from '../agent-loader';
import { savePipeline, Pipeline } from '../pipeline-store';

export async function createCommand(
  request: vscode.ChatRequest,
  stream: vscode.ChatResponseStream
): Promise<void> {
  const agents = await loadAgents();

  if (agents.length === 0) {
    stream.markdown(`> No agents found in \`.github/agents/\`. Create some first.`);
    return;
  }

  // Parse from prompt: /create name:"Research Pipeline" steps:browser,research,coder,reviewer
  const prompt = request.prompt.trim();
  const nameMatch  = prompt.match(/name:"([^"]+)"/);
  const stepsMatch = prompt.match(/steps:([\w,]+)/);

  if (!nameMatch || !stepsMatch) {
    // Show help
    const agentIds = agents.map(a => `\`${a.id}\``).join(', ');
    stream.markdown(
      `## Create a Pipeline\n\n` +
      `**Usage:**\n` +
      `\`\`\`\n@pipeline /create name:"My Pipeline" steps:agent1,agent2,agent3\n\`\`\`\n\n` +
      `**Available agent IDs:** ${agentIds}\n\n` +
      `**Example:**\n` +
      `\`\`\`\n@pipeline /create name:"Research & Build" steps:browser,research,planner,coder,reviewer\n\`\`\`\n`
    );
    return;
  }

  const name  = nameMatch[1];
  const steps = stepsMatch[1].split(',').map(s => s.trim());
  const id    = name.toLowerCase().replace(/[^a-z0-9]+/g, '-');

  // Validate all step IDs exist
  const availableIds = agents.map(a => a.id);
  const invalid = steps.filter(s => !availableIds.includes(s));
  if (invalid.length > 0) {
    stream.markdown(`> ❌ Unknown agent IDs: ${invalid.join(', ')}\n\nRun \`@pipeline /agents\` to see valid IDs.`);
    return;
  }

  const pipeline: Pipeline = { id, name, steps, createdAt: new Date().toISOString() };
  savePipeline(pipeline);

  stream.markdown(
    `## ✅ Pipeline Created: **${name}**\n\n` +
    `**ID:** \`${id}\`\n\n` +
    `**Flow:** ${steps.join(' → ')}\n\n` +
    `**Run it:**\n` +
    `\`\`\`\n` +
    `@pipeline /run ${id}          ← auto mode\n` +
    `@pipeline /run ${id} verify   ← verify mode (pauses at each stage)\n` +
    `\`\`\``
  );
}
```

### `src/commands/run.ts` — Run a pipeline

```typescript
import * as vscode from 'vscode';
import { getPipelineById } from '../pipeline-store';
import { runPipeline, VerifyDecision } from '../engine';

export async function runCommand(
  request: vscode.ChatRequest,
  context: vscode.ChatContext,
  stream: vscode.ChatResponseStream,
  token: vscode.CancellationToken
): Promise<void> {

  const parts   = request.prompt.trim().split(/\s+/);
  const pipelineId = parts[0];
  const verify     = parts.includes('verify');

  if (!pipelineId) {
    stream.markdown(`> Usage: \`@pipeline /run [pipeline-id]\` or \`@pipeline /run [pipeline-id] verify\``);
    return;
  }

  const pipeline = getPipelineById(pipelineId);
  if (!pipeline) {
    stream.markdown(`> ❌ Pipeline \`${pipelineId}\` not found. Run \`@pipeline /list\` to see saved pipelines.`);
    return;
  }

  // Get initial input — everything after the pipeline id and flags
  // or fall back to conversation history
  const inputParts = parts.slice(verify ? 2 : 1);
  let initialInput = inputParts.join(' ').trim();

  if (!initialInput) {
    // Try to pull from last user message in history
    const history = context.history;
    const lastUser = [...history].reverse().find(h => h instanceof vscode.ChatRequestTurn);
    initialInput = lastUser ? (lastUser as vscode.ChatRequestTurn).prompt : '';
  }

  if (!initialInput) {
    stream.markdown(
      `> Please provide input after the pipeline name.\n\n` +
      `Example: \`@pipeline /run ${pipelineId} verify I need a user authentication system with Google OAuth\``
    );
    return;
  }

  stream.markdown(
    `## 🚀 Running Pipeline: **${pipeline.name}**\n\n` +
    `**Mode:** ${verify ? '🔍 Verify (pauses at each stage)' : '⚡ Auto'}\n\n` +
    `**Stages:** ${pipeline.steps.join(' → ')}\n\n` +
    `**Input:** ${initialInput.slice(0, 100)}${initialInput.length > 100 ? '...' : ''}\n\n---\n`
  );

  // Verify handler — listens for next user message
  const onVerify = async (agentId: string, output: string): Promise<{ decision: VerifyDecision; editedOutput?: string }> => {
    // The user's next message in chat will contain their decision
    // We read it from the follow-up prompt via a simple token approach
    // In practice the extension waits for the next turn naturally
    return { decision: 'approve' };
    // Note: Full async verify requires VS Code's followUp API or
    // reading the next request turn — see Phase 8 for the full pattern
  };

  await runPipeline(pipeline, initialInput, { verify }, stream, token, onVerify);
}
```

---

## Phase 8 — `src/extension.ts` (Entry Point)

**Job:** Register the `@pipeline` chat participant and route commands to the right handler.

```typescript
import * as vscode from 'vscode';
import { agentsCommand }  from './commands/agents';
import { createCommand }  from './commands/create';
import { listCommand }    from './commands/list';
import { runCommand }     from './commands/run';

export function activate(context: vscode.ExtensionContext) {

  const participant = vscode.chat.createChatParticipant(
    'pipeline-orchestrator.pipeline',
    async (
      request: vscode.ChatRequest,
      chatContext: vscode.ChatContext,
      stream: vscode.ChatResponseStream,
      token: vscode.CancellationToken
    ) => {
      const command = request.command; // e.g. "agents", "create", "run", "list"

      switch (command) {
        case 'agents':
          await agentsCommand(stream);
          break;

        case 'create':
          await createCommand(request, stream);
          break;

        case 'list':
          await listCommand(stream);
          break;

        case 'run':
          await runCommand(request, chatContext, stream, token);
          break;

        default:
          // No command → show help
          stream.markdown(
            `## @pipeline — Multi-Agent Orchestrator\n\n` +
            `| Command | What it does |\n|---|---|\n` +
            `| \`/agents\` | List all discovered \`.agent.md\` agents |\n` +
            `| \`/create\` | Build a new pipeline by chaining agents |\n` +
            `| \`/list\` | Show all saved pipelines |\n` +
            `| \`/run [id]\` | Run a pipeline in auto mode |\n` +
            `| \`/run [id] verify\` | Run with stage-by-stage checkpoints |\n`
          );
      }
    }
  );

  participant.iconPath = new vscode.ThemeIcon('circuit-board');
  context.subscriptions.push(participant);
}

export function deactivate() {}
```

---

## Phase 9 — Build & Test

### Step 9.1 — Compile

```bash
npm run compile
```

Fix any TypeScript errors before continuing.

### Step 9.2 — Test in VS Code (Debug Mode)

Press **F5** in VS Code. This opens a new VS Code window (Extension Development Host) with your extension loaded.

In the new window, open any project that has a `.github/agents/` folder.

Test each command:
```
@pipeline                              → shows help menu
@pipeline /agents                      → lists your .agent.md files
@pipeline /create name:"Test" steps:browser,coder    → creates pipeline
@pipeline /list                        → shows saved pipelines
@pipeline /run test Here is my input   → runs the pipeline
```

### Step 9.3 — Test Verify Mode

```
@pipeline /run test verify I need a login page
```

Confirm it:
1. Runs first agent and shows output
2. Shows approve/edit/retry/stop options
3. Waits for your response
4. Passes output to next agent on approve

---

## Phase 10 — Package & Install Locally

When everything works:

```bash
vsce package
```

This creates `pipeline-orchestrator-1.0.0.vsix`.

Install it into your VS Code:

```bash
code --install-extension pipeline-orchestrator-1.0.0.vsix
```

Or in VS Code: **Extensions panel → `...` menu → Install from VSIX**

---

## Phase 11 — Add Your First Agents

Create your agents in any project:

```
your-project/
└── .github/
    └── agents/
        ├── browser.agent.md
        ├── research.agent.md
        ├── planner.agent.md
        ├── coder.agent.md
        └── reviewer.agent.md
```

Each file follows this format:

```markdown
---
name: Browser Agent
description: Browses the web and extracts relevant information
model: gpt-4o
tools: [web_search, read_url]
---

You are a web research agent. When given a topic or question:
1. Identify the key information needed
2. Search for and extract the most relevant content
3. Return a structured summary with sources

Always format output as bullet points with clear headings.
```

---

## Complete Usage Flow (End to End)

```
1. Open VS Code in your project
2. Create your .agent.md files in .github/agents/

3. In Copilot Chat:
   @pipeline /agents
   → sees: browser, research, planner, coder, reviewer

4. @pipeline /create name:"Full Build" steps:browser,research,planner,coder,reviewer
   → pipeline saved ✅

5. @pipeline /run full-build verify I want to build a user authentication system

6. Stage 1 (Browser Agent) runs → shows output
   → you type: approve

7. Stage 2 (Research Agent) runs → shows output
   → you type: edit The output should also include OAuth2 flow details

8. Stage 3 (Planner) runs with your corrected input → shows plan
   → you type: approve

9. Stage 4 (Coder) generates code → shows output
   → you type: approve

10. Stage 5 (Reviewer) reviews code → gives feedback
    → Pipeline complete ✅
```

---

## What Gets Built — File Count

| File | Lines (approx) |
|---|---|
| `package.json` | 60 |
| `src/extension.ts` | 45 |
| `src/agent-loader.ts` | 60 |
| `src/pipeline-store.ts` | 55 |
| `src/engine.ts` | 100 |
| `src/commands/agents.ts` | 30 |
| `src/commands/create.ts` | 60 |
| `src/commands/list.ts` | 30 |
| `src/commands/run.ts` | 65 |
| **Total** | **~505 lines** |

---

## Future Enhancements (After V1 Works)

| Feature | Effort |
|---|---|
| Parallel stages (run Gather1 + Gather2 simultaneously) | Medium |
| Pipeline templates (share via JSON) | Easy |
| Stage output saved to files automatically | Easy |
| Web UI dashboard showing run history | Hard |
| Schedule pipelines to run automatically | Medium |
| Share pipelines across team via Git | Easy (already in `.vscode/`) |

---

> [!IMPORTANT]
> **Start by getting `/agents` working first.** Once it correctly reads and lists your `.agent.md` files, everything else is just wiring. Never skip straight to `/run` — validate each phase before moving to the next.
