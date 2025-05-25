# Environment Setup

# Overview

This Standard Operating Procedure (SOP) describes a reproducible process to bootstrap, configure, and verify a Node.js-based CLI project using pnpm, TypeScript, native ESM, and the OpenAI API.

# Prerequisites

* **Node.js** v20 or later  
    
* **pnpm** (install globally if needed):

```shell
npm install -g pnpm
```

* **OpenAI API key** (will be stored in `.env`)

# Procedure

### 1\. Create & initialize the project

```shell
mkdir agent-workshop && cd agent-workshop
pnpm init            # generates package.json

or

git clone https://github.com/canedy/agent-workshop.git
pnpm install
```

### 2\. Install dependencies

```shell
# Runtime dependencies
pnpm add openai dotenv zod lowdb uuid ora

# Development dependencies
pnpm add -D typescript tsx ts-node @types/node @types/lowdb @types/uuid prettier
```

**Note:** `dotenv` lets you load your `OPENAI_API_KEY` from a local `.env` file.

### 3\. Configure `package.json`

Replace the contents of `package.json` with:

```json
{
  "name": "agent-workshop",
  "version": "1.0.0",
  "description": "",
  "type": "module",
  "main": "index.js",
  "scripts": {
    "build": "tsc --project tsconfig.json",
    "start": "node dist/cli.js"
  },
  "bin": {
    "agent": "dist/cli.js"
  },
  "keywords": [],
  "author": "",
  "license": "ISC",
  "packageManager": "pnpm@10.11.0",
  "dependencies": {
    "dotenv": "^16.5.0",
    "lowdb": "^7.0.1",
    "openai": "^4.103.0",
    "uuid": "^11.1.0",
    "zod": "^3.25.28"
  },
  "devDependencies": {
    "@types/lowdb": "^2.0.3",
    "@types/node": "^22.15.21",
    "@types/uuid": "^10.0.0",
    "prettier": "^3.5.3",
    "ts-node": "^10.9.2",
    "typescript": "^5.8.3"
  }
}


```

**Changes summary:**

* Added `"type": "module"` for ESM support.  
* Defined `build` script and `bin.agent` entry.

### 5\. Configure TypeScript (`tsconfig.json`)

1. Initialize with recommended settings:

```shell
npx tsc --init \
  --module NodeNext \
  --moduleResolution NodeNext \
  --target ES2021 \
  --rootDir src/. \
  --outDir dist \
  --strict \
  --esModuleInterop \
  --resolveJsonModule \
  --forceConsistentCasingInFileNames \
```

2. Overwrite with a minimal, focused config:

```json
{
  "compilerOptions": {
    "target": "es2021",
    "module": "nodenext",
    "rootDir": "src/.",
    "moduleResolution": "nodenext",
    "resolveJsonModule": true,
    "outDir": "dist",
    "esModuleInterop": true,
    "forceConsistentCasingInFileNames": true,
    "strict": true,
    "skipLibCheck": true
  },
  "include": ["src/cli.ts", "src/**/*.ts"],
  "exclude": ["node_modules", "dist"]
}
```

**Why `include`/`exclude`?** Ensures only your `src` folder is compiled and that `node_modules`/`dist` are ignored.

### 6\. Configure Prettier

Create a `.prettierrc` in the project root:

```json
{
  "semi": true,
  "singleQuote": true,
  "printWidth": 80
}
```

# **Optional Information**

### **Uninstall each linked package via pnpm**

**List your globally installed packages** (this will include any you linked):

`pnpm ls -g --depth=0`

**Uninstall (unlink) them one by one**:

 `pnpm remove -g <package-name>`

 or the alias

 `pnpm uninstall --global <package-name>`

**`Why`** This is the inverse of `pnpm link --global` .  

# Lab 1 \- Solo Thinker

# Overview

This lab guides you through setting up a TypeScript-based CLI that calls OpenAI's LLM via a simple `agent` command. You'll configure your directory, write the necessary code, build, link to your shell, and troubleshoot common path issues.

# Prerequisites

* Node.js (v20+)  
* npm or pnpm  
* TypeScript  
* An OpenAI API key

# Project Structure

```
my-app/
├─ src/
│  ├─ agents/
│  │  └─ simple-non-agent.ts ← OpenAI client instantiation
│  └─ tools/
│     └─ weatherTool.ts      ← optional example tool
├─ dist/                     ← compiled output
├─ ai.ts                     ← OpenAI client instantiation
├─ cli.ts                    ← TypeScript CLI entrypoint
├─ package.json
├─ tsconfig.json
├─ .env                      ← environment variables
├─ .gitignore
└─ .prettierrc
```

# Setup

1. **Create directories**

```
mkdir -p src/{agents,tools} && touch .env src/ai.ts src/cli.ts src/llm.ts src/memory.ts src/types.ts src/tools/runner.ts src/agents/simpleAgent.ts src/agents/memoryAgent.ts src/agents/toolUsingAgent.ts
```

2.   
   **Add your `.env`** at the project root:

```
OPENAI_API_KEY="your_api_key_here"
```

**Tip:** Add `.env` to your `.gitignore` to avoid committing secrets.

## **Code**

### **`src/agents/ai.ts`**

```ts
import { config } from 'dotenv';
config();  // load variables from .env

import OpenAI from 'openai';

export const openai = new OpenAI({
  apiKey: process.env.OPENAI_API_KEY!,
});
```

# CLI Integration

Update the current `cli.ts` in the /src root:

```ts
#!/usr/bin/env node
import "dotenv/config";
import readline from "readline";
import { simpleAgent } from "./agents/simpleAgent.js";
import { z } from "zod";

async function main() {
  // Set up CLI
  const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout,
    prompt: "🧑‍💻 User: ",
  });

  console.log('Interactive Agent CLI. Type "exit" to quit.');
  rl.prompt();

  for await (const line of rl) {
    const userMessage = line.trim();
    if (!userMessage) {
      rl.prompt();
      continue;
    }

    if (userMessage.toLowerCase() === "exit") {
      rl.close();
      process.exit(0);
    }

    try {
      const response = await simpleAgent({
        userMessage,
      });

      console.log(`🤖 Agent: ${response}\n`);
    } catch (err) {
      console.error("Error:", err instanceof Error ? err.message : err);
    }

    rl.prompt();
  }
}

main();
```

### **`src/agent/simpleAgent.ts`**

```ts
import { runLLM } from "../llm.js";

export const simpleAgent = async ({ userMessage }: { userMessage: string }) => {
  const response = await runLLM({ userMessage: userMessage });

  if (response.content) {
    return response.content;
  }
};
```

### **`src/llm.ts`**

```ts
import { openai } from "./ai.js";
export const runLLM = async ({ userMessage }: { userMessage: string }) => {
  const response = await openai.chat.completions.create({    model: "gpt-4o-mini",    temperature: 0.1,    messages: [{ role: "user", content: userMessage }],  });
return response.choices[0].message;
};
```

### **`src/types.ts`**

```ts
import OpenAI from "openai";
export type AIMessage =
  | OpenAI.Chat.Completions.ChatCompletionAssistantMessageParam
  | { role: "user"; content: string }
  | { role: "tool"; content: string; tool_call_id: string };
export interface ToolFn<A = any, T = any> {   (input: { userMessage: string; toolArgs: A }): Promise<T>;}

```

## 

## **Quick Run:**

```
pnpm exec tsx src/cli.ts
```

## **Build & Link**

### Build

```
pnpm run build    # runs tsc --project tsconfig.json
```

## Link your package.json ‘bin’ globally

```
# creates a global symlink:`agent`→dist/cli.jspnpm link --global
```

```
# start your cli
agent # this is the name you gave in your package.json bin
```

# Optional Information

## Clearing Old Stubs

If you have a leftover executable (e.g., from pnpm) earlier in your `$PATH`, do:

```
# remove stale stub
rm ~/.local/share/pnpm/agent

# clear shell cache
hash -r           # bash
rehash            # zsh
```

Verify:

```
which -a agent
# should only list ~/.nvm/.../bin/agent
```

# Troubleshooting

* **No dist output**: run `npx tsc --project tsconfig.json --listFiles` to check ts input includes.  
* **API key missing**: ensure `.env` exists and `dotenv.config()` is called before client instantiation.  
* **Wrong binary**: run `which -a agent` and remove any unwanted stubs ahead of your linked path.

# Lab 2 – Stateful Memory

# Objective

By the end of this lab, you will extend a simple CLI agent to maintain conversational context across turns by persisting and retrieving messages from a JSON-backed memory store.

# Memory Manager Implementation

Create a file `memory.ts` in the projects /src directory:

```ts
import { JSONFilePreset } from "lowdb/node";
import type { AIMessage } from "./types.js";
import { v4 as uuidv4 } from "uuid";

export type MessageWithMetadata = AIMessage & {
  id: string;
  createdAt: string;
};

type Data = {
  messages: MessageWithMetadata[];
};

export const addMetadata = (message: AIMessage) => {
  return {
    ...message,
    id: uuidv4(),
    createdAt: new Date().toISOString(),
  };
};

export const removeMetadata = (message: MessageWithMetadata) => {
  const { id, createdAt, ...rest } = message;
  return rest;
};

const defaultData: Data = {
  messages: [],
};

export const getDb = async () => {
  const db = await JSONFilePreset<Data>("memory.json", defaultData);
  return db;
};

export const addMessages = async (messages: AIMessage[]) => {
  const db = await getDb();
  db.data.messages.push(...messages.map(addMetadata));
  await db.write();
};

export const getMessages = async () => {
  const db = await getDb();
  return db.data.messages.map(removeMetadata);
};

export const saveToolResponse = async (
  toolCallId: string,
  toolResponse: string
) => {
  return addMessages([
    {
      role: "tool",
      content: toolResponse,
      tool_call_id: toolCallId,
    },
  ]);
};
```

**Note**: A memory.json file will be created in the root directory when running the code.

# CLI Integration

Update the current `cli.ts` in the /src root:

```ts
#!/usr/bin/env node
import "dotenv/config";
import readline from "readline";
// import { simpleAgent } from "./agents/simpleAgent.js";
import { memoryAgent } from "./agents/memoryAgent.js";
import { z } from "zod";

async function main() {
  // Set up CLI
  const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout,
    prompt: "🧑‍💻 User: ",
  });

  console.log('Interactive Agent CLI. Type "exit" to quit.');
  rl.prompt();

  for await (const line of rl) {
    const userMessage = line.trim();
    if (!userMessage) {
      rl.prompt();
      continue;
    }

    if (userMessage.toLowerCase() === "exit") {
      rl.close();
      process.exit(0);
    }

    try {
      const response = await memoryAgent({
        userMessage,
      });

      console.log(`🤖 Agent: ${response}\n`);
    } catch (err) {
      console.error("Error:", err instanceof Error ? err.message : err);
    }

    rl.prompt();
  }
}

main();
```

# Agent with Memory

Create `agents/memoryAgent.ts`:

```ts
import type { AIMessage } from '../types.js';
import { addMessages, getMessages, saveToolResponse } from '../memory.js';
import { runLLM } from '../llm.js';

export const memoryAgent = async ({ userMessage }: { userMessage: string }) => {
  await addMessages([{ role: 'user', content: userMessage }]);
  const history = await getMessages();
  const response = await runLLM({ messages: history });

  await addMessages([response]);

  if (response.content) {
    return response.content;
  }
};

```

### **`src/llm.ts`**

```ts
import type { AIMessage } from "./types.js";
import { openai } from "./ai.js";

export const runLLM = async ({ messages }: { messages: AIMessage[] }) => {
  const response = await openai.chat.completions.create({
    model: "gpt-4o-mini",
    temperature: 0.1,
    messages,
  });

  return response.choices[0].message;
};
```

## 

## **Quick Run:**

```
pnpm exec tsx src/cli.ts
```

## **Build & Link**

### Build

```
pnpm run build    # runs tsc --project tsconfig.json
```

## Link your package.json ‘bin’ globally

```
# creates a global symlink:`agent`→dist/cli.jspnpm link --global
```

```shell
# start your cli
agent # this is the name you gave in your package.json bin
```


# Lab 3 – Tooling & Action Execution

# Objective

By the end of this lab, you will extend a simple CLI agent to call several tools, which will allow our agent to start taking action.

# CLI Updates

Update the current `cli.ts` in the /src root:

```ts
#!/usr/bin/env node
import "dotenv/config";
import readline from "readline";
// import { simpleAgent } from "./agents/simpleAgent.js";
// import { memoryAgent } from "./agents/memoryAgent.js";
import { toolUsingAgent } from "./agents/toolUsingAgent.js";
import { z } from "zod";

async function main() {
  // Set up CLI
  const rl = readline.createInterface({
    input: process.stdin,
    output: process.stdout,
    prompt: "🧑‍💻 User: ",
  });

  console.log('Interactive Agent CLI. Type "exit" to quit.');
  rl.prompt();

  for await (const line of rl) {
    const userMessage = line.trim();
    if (!userMessage) {
      rl.prompt();
      continue;
    }

    if (userMessage.toLowerCase() === "exit") {
      rl.close();
      process.exit(0);
    }

    try {
      // adding tools to the agent
      const weatherTool = {
        name: "get_weather",
        description: `use this to get the weather`,
        parameters: z.object({
          reasoning: z.string().describe("why did you pick this tool?"),
        }),
      };

      const heaterTool = {
        name: "get_heater_command",
        description: `Use this to decide whether to turn the chicken-coop heater ON or OFF based on current temperature.`,
        parameters: z.object({
          reasoning: z.string().describe("why you chose this tool"),
          currentTemperature: z
            .number()
            .describe("the current temperature in °C"),
          threshold: z
            .number()
            .describe(
              "temperature threshold in °C (defaults to 20 if not provided)"
            ),
        }),
      };

      const response = await toolUsingAgent({
        userMessage,
        tools: [weatherTool, heaterTool],
      });

      console.log(`🤖 Agent: ${response}\n`);
    } catch (err) {
      console.error("Error:", err instanceof Error ? err.message : err);
    }

    rl.prompt();
  }
}

main();
```

# Agent with Memory and Tools

Create `agents/toolUsingAgent.ts`:

```ts
import type { AIMessage } from '../types.js';
import { addMessages, getMessages, saveToolResponse } from '../memory.js';
import { runLLM } from '../llm.js';
import { runTool } from '../tools/runner.js';

export const toolUsingAgent = async ({
  userMessage,
  tools,
}: {
  userMessage: string;
  tools: any[];
}) => {
  await addMessages([{ role: 'user', content: userMessage }]);

  while (true) {
    const history = await getMessages();
    const response = await runLLM({ messages: history, tools });
    await addMessages([response]);

    if (response.content) {
      return response.content;
    }

    if (response.tool_calls) {
      const toolCall = response.tool_calls[0];
      console.log(`🤖 Executing: ${toolCall.function.name}`);
      const toolResponse = await runTool(toolCall, userMessage);
      await saveToolResponse(toolCall.id, JSON.stringify(toolResponse));
    }
  }
};

```

### **`src/tools/runner.ts`**

```ts
import type OpenAI from "openai";

export const getWeather = () => [
  {
    location: "Atlanta, Georgia",
    description: "Hot",
    temperature: 90,
    unit: "F",
    humidity: 45,
    wind: { speed: 8, unit: "mph" },
  },
  {
    location: "Columbus, Ohio",
    description: "Humid",
    temperature: 60,
    unit: "F",
    humidity: 70,
    wind: { speed: 10, unit: "mph" },
  },
  {
    location: "Cleveland, Ohio",
    description: "Windy",
    temperature: 19,
    unit: "F",
    humidity: 60,
    wind: { speed: 15, unit: "mph" },
  },
];

export const getHeaterCommand = ({
  reasoning,
  currentTemperature,
  threshold,
}: {
  reasoning: string;
  currentTemperature: number;
  threshold: number;
}) => {
  const effectiveThreshold = threshold ?? 20; // fallback at runtime
  return {
    currentTemperature,
    unit: "C" as const,
    threshold: effectiveThreshold,
    heaterState: currentTemperature < effectiveThreshold ? "ON" : "OFF",
    reasoning,
  };
};

export const runTool = async (
  toolCall: OpenAI.Chat.Completions.ChatCompletionMessageToolCall,
  userMessage: string
) => {
  const input = {
    userMessage,
    toolArgs: JSON.parse(toolCall.function.arguments || "{}"),
  };

  switch (toolCall.function.name) {
    case "get_weather":
      return getWeather();
    case "get_heater_command":
      return getHeaterCommand(input.toolArgs);
    default:
      throw new Error(`Unknown tool: ${toolCall.function.name}`);
  }
};

```

### **`src/llm.ts`**

```ts
import type { AIMessage } from "./types.js";
import { openai } from "./ai.js";
import { zodFunction } from "openai/helpers/zod";

export const runLLM = async ({
  messages,
  tools,
}: {
  messages: AIMessage[];
  tools: any[];
}) => {
  const formattedTools = tools.map(zodFunction);

  const response = await openai.chat.completions.create({
    model: "gpt-4o-mini",
    temperature: 0.1,
    messages,
    tools: formattedTools,
    tool_choice: "auto",
    parallel_tool_calls: false,
  });

  return response.choices[0].message;
};
```

## **Quick Run:**

```
pnpm exec tsx src/cli.ts
```

## **Build & Link**

### Build

```
pnpm run build    # runs tsc --project tsconfig.json
```

## Link your package.json ‘bin’ globally

```
# creates a global symlink:`agent`→dist/cli.jspnpm link --global
```

```
# start your cli
agent # this is the name you gave in your package.json bin
```

