# How to Build Your Own Agent Harness on the Claude Agent SDK

A step-by-step plan for building a reusable AI coding agent for embedded projects
(LPC55S69-EVK, LPC55S16-EVK, and future boards). You build the harness once.
Each project then plugs in through a small profile folder.

## Contents

1. [Background: configure vs. build](#1-background-configure-vs-build)
2. [Architecture](#2-architecture)
3. [Step-by-step plan](#3-step-by-step-plan)
4. [Recommended order](#4-recommended-order)
5. [Code sketches](#5-code-sketches)
6. [References](#6-references)

---

## 1. Background: configure vs. build

There are two ways to get a Claude coding agent working on a repository:

| Option | What you do | When to choose it |
|---|---|---|
| **1. Configure Claude Code** | Add `CLAUDE.md`, `.claude/settings.json` (permissions, hooks), `.claude/skills/`, `.claude/agents/`, `.mcp.json` to the repo | You want results quickly, using the existing Claude Code CLI / VS Code extension |
| **2. Build your own harness** | Write your own agent application with the **Claude Agent SDK** (TypeScript or Python) | You want your own tools, your own UI, and full control over safety rules, reused across many projects |

This document covers **option 2**.

The Agent SDK provides the agent loop, tool calling, permission handling,
hooks, subagents and sessions. You add:

- the **tools** (build, sign, flash, read UART, …)
- the **safety rules** (what may touch hardware, and when)
- the **front end** (CLI first, VS Code extension later)

---

## 2. Architecture

```
embedded-agent/                 <- the harness (built once)
├─ core/        agent loop, profile loader, safety layer, logging
├─ tools/       generic tools: build, sign, flash, serial, map-size
├─ cli/         terminal front end (phases 1-6)
└─ vscode/      VS Code extension front end (phase 7)

<any project>/.agent/           <- per-project profile (added to each repo)
├─ profile.json   board, tool paths, COM port, memory map, allowed actions
└─ prompt.md      project-specific rules, e.g. "edit .cproject only with the IDE closed"
```

Key idea: **the harness has no board-specific knowledge.** Everything that
differs between projects lives in `.agent/` inside the project repo.

Example `profile.json`:

```json
{
  "name": "lpcxpresso55s69_led_blinky",
  "board": "LPC55S69-EVK",
  "core": "cm33_core0",
  "rom": "K3.0.0",
  "isp": { "tool": "blhost", "port": "COM4", "baud": 57600 },
  "build": { "type": "mcuxpresso", "config": "Release" },
  "memory": {
    "flash": { "start": "0x00000000", "size": "0x00098000" },
    "sram":  { "start": "0x20000000", "size": "0x00044000" }
  },
  "scripts": {
    "sign":  "C:/path/to/tools/sign.ps1",
    "flash": "C:/path/to/tools/flash.ps1"
  },
  "forbidden": ["seal_cmpa", "write_cfpa_lock"]
}
```

---

## 3. Step-by-step plan

### Phase 0: Decisions and setup

1. **Choose a language.** **TypeScript** is recommended: VS Code extensions are
   written in TypeScript, so the CLI and the extension share one codebase.
   Python works if you only need a CLI.
2. **Install prerequisites.**
   - Node.js 20 or newer
   - an API key from <https://console.anthropic.com>. Agent SDK usage is
     billed at API rates, separately from a Claude subscription.
3. **Create the repository** for the harness, e.g. `embedded-agent`.

✅ **Done when:** `npm install @anthropic-ai/claude-agent-sdk` succeeds and
`ANTHROPIC_API_KEY` is set as an environment variable (never commit it).

### Phase 1: Minimal agent loop (CLI)

4. Write a `query()` call that streams the agent's messages to the terminal.
5. Allow only **read-only** built-in tools (`Read`, `Grep`, `Glob`) and set
   `cwd` to a project folder.

✅ **Done when:** asking *"Summarize this project's memory map"* makes the agent
read the linker script / `.cproject` and answer correctly.

### Phase 2: Project profile system

6. Define the `profile.json` schema (name, board, ROM version, tool paths,
   COM port, baud rate, flash/RAM regions, forbidden actions). Validate it
   with a schema library such as `zod`.
7. Write a loader that finds `.agent/` in the target project and appends
   `prompt.md` to the system prompt.

✅ **Done when:** the same CLI, pointed at an LPC55S16 project and then at an
LPC55S69 project, knows which board it is working on.

### Phase 3: Embedded tools

This phase is the main value of the harness.

8. Wrap your existing scripts as **custom tools** using the SDK's in-process
   MCP server:

   | Tool | What it does |
   |---|---|
   | `build` | Headless MCUXpresso / make build. Returns errors and the size summary |
   | `sign` | Calls the signing script for the image |
   | `check_connection` | Runs `blhost get-property 1` to confirm the ISP link |
   | `flash` | blhost erase / write of the signed image |
   | `serial_read` | Captures UART output for N seconds (boot log) |
   | `map_size` | Parses the `.map` file and checks usage against linker regions |

9. Make every tool return **structured JSON** (status, values, error text),
   not raw console output. The agent reasons far better over structured
   results.

✅ **Done when:** the agent can build → sign → flash a sample (e.g. led_blinky)
and read its boot log over UART.

### Phase 4: Safety layer

**Build this before any Phase 3 tool touches real hardware.**

10. **Permission callback (`canUseTool`):** read-only tools run automatically.
    Flash, erase and provisioning tools always ask the user for confirmation.
11. **Hooks** (`PreToolUse`) that enforce hard rules:
    - no erase/write unless `check_connection` succeeded earlier in the session
    - make sure the COM port is free (serial terminal closed) before blhost runs
    - block irreversible commands completely (e.g. sealing CMPA, locking CFPA)
    - block `.cproject` edits while MCUXpresso IDE is running (the IDE can
      silently overwrite them)
12. **Dry-run mode** (`--dry-run`): replace blhost with a fake that logs the
    command and returns success. Use it for all harness testing.

✅ **Done when:** the prompt *"Erase the whole flash"* is either blocked or stops
for your confirmation, and never runs silently.

> Why so strict? On LPC55 devices some mistakes are permanent: SWD debug can be
> locked after a signed boot, and a sealed CMPA cannot be changed again.

### Phase 5: Subagents

13. Add specialist subagents, each with its own prompt and limited tool set:
    - `map-auditor`: reviews flash/RAM/stack usage against linker regions
    - `code-reviewer`: reviews changes before you commit
    - `doc-writer`: keeps the project's design/bring-up notes up to date

### Phase 6: Sessions and logging

14. Save transcripts per project, support **resuming** a session, and log the
    token cost of each run.
15. Write an **audit log** of every hardware action: time, command, port and
    result.

### Phase 7: VS Code extension

16. Scaffold the extension with `npx yo code` (TypeScript). Add:
    - a **chat webview panel**
    - an **output channel** for tool logs
    - a **status bar item** showing the active project profile
17. Run the Phase 1–6 core inside the extension host. Show permission requests
    as VS Code dialogs (`window.showWarningMessage` with Allow / Deny).

✅ **Done when:** you open a board repo in VS Code and run *"build and flash"*
from the panel, with confirmations shown as dialogs.

### Phase 8: Roll out to projects

18. Create a **profile template** and add `.agent/` to each board repository
    (`lpc55s69-evk`, `lpc55s16-evk`, …).
19. Keep a list of **test prompts** in the harness repo and re-run them
    (in dry-run mode) after every harness change.

---

## 4. Recommended order

```
Phase 0 -> 1 -> 2 -> 4 -> 3 -> 5 / 6 -> 7 -> 8
```

The safety layer (4) comes **before** the hardware tools (3) on purpose.

---

## 5. Code sketches

These sketches show the shape of the code. Check the API names against the
current Agent SDK documentation before you use them.

### 5.1 Minimal agent loop (Phase 1)

```ts
import { query } from "@anthropic-ai/claude-agent-sdk";

const projectDir = process.argv[2];
const prompt = process.argv.slice(3).join(" ");

for await (const message of query({
  prompt,
  options: {
    cwd: projectDir,
    allowedTools: ["Read", "Grep", "Glob"],
  },
})) {
  if (message.type === "assistant") {
    for (const block of message.message.content) {
      if (block.type === "text") process.stdout.write(block.text);
    }
  }
  if (message.type === "result") {
    console.log(`\n[done] cost: $${message.total_cost_usd}`);
  }
}
```

### 5.2 Custom tool (Phase 3)

```ts
import { tool, createSdkMcpServer } from "@anthropic-ai/claude-agent-sdk";
import { z } from "zod";
import { execFile } from "node:child_process";
import { promisify } from "node:util";

const run = promisify(execFile);

const checkConnection = tool(
  "check_connection",
  "Check the ISP link to the MCU with blhost get-property",
  { port: z.string(), baud: z.number() },
  async ({ port, baud }) => {
    try {
      const { stdout } = await run("blhost", ["-p", `${port},${baud}`, "get-property", "1"]);
      return { content: [{ type: "text", text: JSON.stringify({ ok: true, stdout }) }] };
    } catch (e) {
      return { content: [{ type: "text", text: JSON.stringify({ ok: false, error: String(e) }) }] };
    }
  }
);

export const embeddedTools = createSdkMcpServer({
  name: "embedded",
  version: "0.1.0",
  tools: [checkConnection /*, build, sign, flash, serialRead, mapSize */],
});
// Pass it with: options.mcpServers = { embedded: embeddedTools }
// The agent sees it as: mcp__embedded__check_connection
```

### 5.3 Safety: permission callback and hook (Phase 4)

```ts
let connectionChecked = false;

const DANGEROUS = ["mcp__embedded__flash", "mcp__embedded__erase"];

const options = {
  // Ask the user before any dangerous tool runs
  canUseTool: async (toolName: string, input: unknown) => {
    if (DANGEROUS.includes(toolName)) {
      const ok = await askUser(`Allow ${toolName} with ${JSON.stringify(input)}?`);
      return ok
        ? { behavior: "allow", updatedInput: input }
        : { behavior: "deny", message: "User denied the hardware action." };
    }
    return { behavior: "allow", updatedInput: input };
  },

  // Hard rules that the user cannot accidentally approve
  hooks: {
    PreToolUse: [{
      hooks: [async (hookInput: any) => {
        if (hookInput.tool_name === "mcp__embedded__check_connection") {
          connectionChecked = true; // simplified: set on success in the real tool
        }
        if (DANGEROUS.includes(hookInput.tool_name) && !connectionChecked) {
          return {
            hookSpecificOutput: {
              hookEventName: "PreToolUse",
              permissionDecision: "deny",
              permissionDecisionReason: "Run check_connection first.",
            },
          };
        }
        return {};
      }],
    }],
  },
};
```

---

## 6. References

- Claude Agent SDK overview: <https://docs.claude.com/en/api/agent-sdk/overview>
- Agent SDK, TypeScript: <https://docs.claude.com/en/api/agent-sdk/typescript>
- Agent SDK, Python: <https://docs.claude.com/en/api/agent-sdk/python>
- Claude Code (option 1, configure instead of build): <https://docs.claude.com/en/docs/claude-code/overview>
- VS Code extension API: <https://code.visualstudio.com/api>
- NXP blhost user guide (MCUXpresso Secure Provisioning Tool / SPSDK documentation)
