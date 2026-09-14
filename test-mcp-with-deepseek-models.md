# Connect an MCP Server to DeepSeek — Working Code + Zero-Setup Option

DeepSeek does not implement MCP at the API level. Neither does OpenAI, Google, xAI or anyone else — **Anthropic is the only vendor whose API accepts MCP server definitions directly.** Every other model reaches your MCP tools through a client that converts them into that provider's function-calling format.

That conversion is the whole job, and it is about 40 lines. This guide gives you those 40 lines, working, plus the parts that bite you afterwards.

- [The 60-second option (no code)](#the-60-second-option)
- [The code path](#the-code-path)
- [Full working example](#full-working-example)
- [Python version](#python-version)
- [Which DeepSeek model](#which-deepseek-model)
- [Five things that will bite you](#five-things-that-will-bite-you)
- [Testing without a server of your own](#testing-without-a-server-of-your-own)

---

## The 60-second option

If you only want to see whether DeepSeek calls your tools correctly, you do not need any of the code below.

**[MCP Agent Studio](https://mcpplaygroundonline.com/mcp-agent-studio)** — paste your MCP server URL, pick DeepSeek V4, and watch it discover your tools and call them in a real conversation. Every tool call is shown with its full JSON input and output, so you can see exactly which tool the model chose and what arguments it built. No API key, no install, no DeepSeek account.

There is a DeepSeek-specific entry point at **[mcpplaygroundonline.com/test-mcp-with/deepseek](https://mcpplaygroundonline.com/test-mcp-with/deepseek)**.

Use that to answer *"does this model handle my schemas?"*. Use the code below when you are building the thing for real.

---

## The code path

### Prerequisites

```bash
npm install @modelcontextprotocol/sdk openai
```

You need a remote MCP server speaking **Streamable HTTP** (or SSE), and a DeepSeek API key from
[platform.deepseek.com](https://platform.deepseek.com). The DeepSeek API is OpenAI-format compatible, so the
official `openai` package works against it with nothing but a `baseURL` change.

### Step 1 — Connect to the MCP server

```ts
import { Client } from '@modelcontextprotocol/sdk/client/index.js';
import { StreamableHTTPClientTransport } from '@modelcontextprotocol/sdk/client/streamableHttp.js';

const transport = new StreamableHTTPClientTransport(
  new URL('https://your-server.example.com/mcp'),
  {
    // Only if your server is authenticated.
    requestInit: { headers: { Authorization: `Bearer ${process.env.MCP_TOKEN}` } },
  }
);

const mcp = new Client({ name: 'deepseek-mcp-demo', version: '1.0.0' }, { capabilities: {} });
await mcp.connect(transport);
```

If your server uses the older HTTP+SSE transport, swap in `SSEClientTransport` from
`@modelcontextprotocol/sdk/client/sse.js`. Everything after this point is identical.

### Step 2 — Convert MCP tools into function definitions

This is the part people get wrong.

```ts
import type OpenAI from 'openai';

type ChatTool = OpenAI.Chat.Completions.ChatCompletionTool;

/**
 * MCP tool names are far more permissive than function-calling names — servers
 * commonly namespace them like `github:create_issue` or `db.query`. Most
 * providers reject anything outside [a-zA-Z0-9_-], so sanitise and keep a map
 * back to the real name for the call.
 */
export function toFunctionTools(mcpTools: Array<{
  name: string;
  description?: string;
  inputSchema?: unknown;
}>): { tools: ChatTool[]; nameMap: Map<string, string> } {
  const nameMap = new Map<string, string>();
  const seen = new Set<string>();

  const tools = mcpTools.map((t, i) => {
    let safe = t.name.replace(/[^a-zA-Z0-9_-]/g, '_') || `tool_${i}`;
    let n = 0;
    while (seen.has(safe)) safe = `${safe}_${++n}`; // collisions after sanitising
    seen.add(safe);
    nameMap.set(safe, t.name);

    return {
      type: 'function' as const,
      function: {
        name: safe,
        description: t.description ?? `MCP tool ${t.name}`,
        // MCP inputSchema is already JSON Schema — pass it straight through.
        parameters: (t.inputSchema as Record<string, unknown>) ?? {
          type: 'object',
          properties: {},
        },
      },
    };
  });

  return { tools, nameMap };
}
```

### Step 3 — Run the tool-call loop

```ts
import OpenAI from 'openai';

const llm = new OpenAI({
  baseURL: 'https://api.deepseek.com',
  apiKey: process.env.DEEPSEEK_API_KEY,
});

const MODEL = 'deepseek-v4-pro';

const { tools: mcpTools } = await mcp.listTools();
const { tools, nameMap } = toFunctionTools(mcpTools);

const messages: OpenAI.Chat.ChatCompletionMessageParam[] = [
  { role: 'user', content: 'What can you do? Then do the most useful one.' },
];

// Bound the loop. A model that keeps calling tools forever is a real failure
// mode, not a hypothetical one.
for (let turn = 0; turn < 10; turn++) {
  const res = await llm.chat.completions.create({ model: MODEL, messages, tools });
  const msg = res.choices[0].message;
  messages.push(msg);

  if (!msg.tool_calls?.length) {
    console.log(msg.content);
    break;
  }

  for (const call of msg.tool_calls) {
    const realName = nameMap.get(call.function.name) ?? call.function.name;

    let args: Record<string, unknown> = {};
    try {
      args = JSON.parse(call.function.arguments || '{}');
    } catch {
      // Models do emit malformed JSON. Feed the error back rather than crashing.
      messages.push({
        role: 'tool',
        tool_call_id: call.id,
        content: 'Error: arguments were not valid JSON. Try again.',
      });
      continue;
    }

    try {
      const result = await mcp.callTool({ name: realName, arguments: args });
      messages.push({
        role: 'tool',
        tool_call_id: call.id,
        content: renderResult(result),
      });
    } catch (err) {
      // Tool errors are information for the model, not a reason to abort.
      messages.push({
        role: 'tool',
        tool_call_id: call.id,
        content: `Error calling ${realName}: ${(err as Error).message}`,
      });
    }
  }
}

/** MCP returns content blocks; flatten the text ones for the model. */
function renderResult(result: unknown): string {
  const content = (result as { content?: Array<{ type: string; text?: string }> })?.content;
  if (!Array.isArray(content)) return JSON.stringify(result);
  const text = content
    .filter((c) => c.type === 'text' && c.text)
    .map((c) => c.text)
    .join('\n');
  return text || JSON.stringify(content);
}
```

That is the entire integration. Everything else is polish.

---

## Full working example

Self-contained — copy this whole block into one file and it runs.

```ts
// deepseek-mcp.ts — run with: npx tsx deepseek-mcp.ts "your prompt here"
import { Client } from '@modelcontextprotocol/sdk/client/index.js';
import { StreamableHTTPClientTransport } from '@modelcontextprotocol/sdk/client/streamableHttp.js';
import OpenAI from 'openai';

type ChatTool = OpenAI.Chat.Completions.ChatCompletionTool;

const SERVER_URL = process.env.MCP_SERVER_URL!;
const MODEL = 'deepseek-v4-pro';

function toFunctionTools(
  mcpTools: Array<{ name: string; description?: string; inputSchema?: unknown }>
): { tools: ChatTool[]; nameMap: Map<string, string> } {
  const nameMap = new Map<string, string>();
  const seen = new Set<string>();

  const tools = mcpTools.map((t, i) => {
    let safe = t.name.replace(/[^a-zA-Z0-9_-]/g, '_') || `tool_${i}`;
    let n = 0;
    while (seen.has(safe)) safe = `${safe}_${++n}`;
    seen.add(safe);
    nameMap.set(safe, t.name);

    return {
      type: 'function' as const,
      function: {
        name: safe,
        description: t.description ?? `MCP tool ${t.name}`,
        parameters: (t.inputSchema as Record<string, unknown>) ?? {
          type: 'object',
          properties: {},
        },
      },
    };
  });

  return { tools, nameMap };
}

function renderResult(result: unknown): string {
  const content = (result as { content?: Array<{ type: string; text?: string }> })?.content;
  if (!Array.isArray(content)) return JSON.stringify(result);
  const text = content
    .filter((c) => c.type === 'text' && c.text)
    .map((c) => c.text)
    .join('\n');
  return text || JSON.stringify(content);
}

async function main() {
  const mcp = new Client({ name: 'deepseek-mcp', version: '1.0.0' }, { capabilities: {} });
  await mcp.connect(new StreamableHTTPClientTransport(new URL(SERVER_URL)));

  const llm = new OpenAI({
    baseURL: 'https://api.deepseek.com',
    apiKey: process.env.DEEPSEEK_API_KEY,
  });

  const { tools: mcpTools } = await mcp.listTools();
  console.log(`Discovered ${mcpTools.length} tools:`, mcpTools.map((t) => t.name).join(', '));

  const { tools, nameMap } = toFunctionTools(mcpTools);
  const messages: OpenAI.Chat.ChatCompletionMessageParam[] = [
    { role: 'user', content: process.argv[2] ?? 'List your tools, then use one.' },
  ];

  for (let turn = 0; turn < 10; turn++) {
    const res = await llm.chat.completions.create({ model: MODEL, messages, tools });
    const msg = res.choices[0].message;
    messages.push(msg);

    if (!msg.tool_calls?.length) {
      console.log('\n' + msg.content);
      break;
    }

    for (const call of msg.tool_calls) {
      const realName = nameMap.get(call.function.name) ?? call.function.name;
      console.log(`→ ${realName}(${call.function.arguments})`);
      try {
        const out = await mcp.callTool({
          name: realName,
          arguments: JSON.parse(call.function.arguments || '{}'),
        });
        messages.push({ role: 'tool', tool_call_id: call.id, content: renderResult(out) });
      } catch (err) {
        messages.push({
          role: 'tool',
          tool_call_id: call.id,
          content: `Error: ${(err as Error).message}`,
        });
      }
    }
  }

  await mcp.close();
}

main().catch((e) => {
  console.error(e);
  process.exit(1);
});
```

---

## Python version

```python
# pip install mcp openai
import asyncio, json, os
from mcp import ClientSession
from mcp.client.streamable_http import streamablehttp_client
from openai import AsyncOpenAI

MODEL = "deepseek-v4-pro"

def to_function_tools(mcp_tools):
    tools, name_map, seen = [], {}, set()
    for i, t in enumerate(mcp_tools):
        safe = "".join(c if c.isalnum() or c in "_-" else "_" for c in t.name) or f"tool_{i}"
        while safe in seen:
            safe += "_1"
        seen.add(safe)
        name_map[safe] = t.name
        tools.append({
            "type": "function",
            "function": {
                "name": safe,
                "description": t.description or f"MCP tool {t.name}",
                "parameters": t.inputSchema or {"type": "object", "properties": {}},
            },
        })
    return tools, name_map

async def main():
    llm = AsyncOpenAI(
        base_url="https://api.deepseek.com",
        api_key=os.environ["DEEPSEEK_API_KEY"],
    )
    async with streamablehttp_client(os.environ["MCP_SERVER_URL"]) as (r, w, _):
        async with ClientSession(r, w) as session:
            await session.initialize()
            listed = await session.list_tools()
            tools, name_map = to_function_tools(listed.tools)

            messages = [{"role": "user", "content": "List your tools, then use one."}]
            for _ in range(10):
                res = await llm.chat.completions.create(model=MODEL, messages=messages, tools=tools)
                msg = res.choices[0].message
                messages.append(msg.model_dump(exclude_none=True))

                if not msg.tool_calls:
                    print(msg.content)
                    break

                for call in msg.tool_calls:
                    real = name_map.get(call.function.name, call.function.name)
                    try:
                        out = await session.call_tool(real, json.loads(call.function.arguments or "{}"))
                        text = "\n".join(c.text for c in out.content if getattr(c, "text", None))
                    except Exception as e:
                        text = f"Error calling {real}: {e}"
                    messages.append({"role": "tool", "tool_call_id": call.id, "content": text})

asyncio.run(main())
```

---

## Which DeepSeek model

| API model ID | Best for | Relative cost |
|---|---|---|
| `deepseek-v4-flash` | High-volume workflows with flat, well-described tools | Cheapest |
| `deepseek-v4-pro` | Anything with nested arguments or several similar tools | ~2× Flash |

> **If you are copying a tutorial that uses `deepseek-chat` or `deepseek-reasoner`, stop.** Both were
> **deprecated on 24 July 2026** and replaced by the two IDs above. Most guides still published online predate
> that change, which is a common source of confusing failures against the current API.

They do **not** select tools identically. Benchmark the one you intend to ship, not the one that is cheapest to test — a Flash result does not transfer to Pro or vice versa.

DeepSeek's real appeal for MCP is economic. Against a frontier model it costs a small fraction per call, so for a workflow running hundreds of times a day the question is never "is Claude better" (it often is) but "is DeepSeek good enough on *my* tools". That is a one-minute measurement, and the answer is frequently yes.

---

## Five things that will bite you

**1. Namespaced tool names get rejected.** `github:create_issue` is a legal MCP tool name and an illegal function name. Sanitise, and keep the map back — the code above does both. Skipping this is the single most common cause of "DeepSeek ignores my tools".

**2. Deeply nested arguments are where DeepSeek diverges most from Claude.** An object three levels deep survives the function-calling conversion differently than it does through Anthropic's native MCP path. If a tool works everywhere except DeepSeek, look at your argument shape before you look at your prompt.

**3. Tool descriptions do more work here than with frontier models.** Claude infers intent from a mediocre description; DeepSeek largely does not. Describe *intent*, not implementation — "finds a customer's past orders" beats "queries the orders table".

**4. Tool errors are information, not failures.** Push the error text back as the tool result and let the model retry. Aborting the loop on the first error throws away the model's ability to correct itself.

**5. Bound your loop.** A model that calls tools in a cycle is a real failure mode. The examples cap at 10 turns; pick a number and enforce it.

---

## Testing without a server of your own

If you are building a *client* and need something predictable to point it at, there are free public MCP servers with no signup:

| Server | URL | What it is for |
|---|---|---|
| Complex schema | `https://mcpplaygroundonline.com/mcp-complex-server` | Four tools with overlapping nested arguments — the best target for tool-selection testing |
| Error simulation | `https://mcpplaygroundonline.com/mcp-error-server` | Returns validation, rate-limit, timeout and five other error types on demand |
| Auth | `https://mcpplaygroundonline.com/mcp-auth-server` | 401 with a proper `WWW-Authenticate` challenge |
| Stateless 2026-07-28 | `https://mcpplaygroundonline.com/mcp-stateless-server` | Multi Round-Trip Requests and signed `requestState` |

Full list and `curl` examples: **[mcpplaygroundonline.com/mock-mcp-servers](https://mcpplaygroundonline.com/mock-mcp-servers)**

For point 2 above, call `process_order` on the complex-schema server — it has the deepest nesting of the four, which is precisely the structure DeepSeek handles least like Claude.

---

## Related

- [MCP Agent Studio](https://mcpplaygroundonline.com/mcp-agent-studio) — run any MCP server against 60+ models in the browser
- [Which AI models work with MCP](https://mcpplaygroundonline.com/mcp-model-comparison) — the full matrix, and why behaviour differs per model
- [Test with GLM](./test-mcp-with-glm-models.md) · [Test with Kimi](./test-mcp-with-kimi-models.md) · [Test with OpenAI](./test-mcp-with-openai-models.md)
- [Full DeepSeek + MCP guide](https://mcpplaygroundonline.com/blog/testing-mcp-with-deepseek)

---

*Code in this guide is MIT-licensed — copy it.*
