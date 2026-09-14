# 🚀 Introducing MCP Agent Studio — Connect Any MCP Server, Chat with 40+ AI Models

We're shipping a major new feature on [MCP Playground](https://mcpplaygroundonline.com): **MCP Agent Studio**, a browser-based AI agent that lets you connect your MCP servers and query them in plain English — no code, no local install, no API key setup.

---

## What is MCP Agent Studio?

MCP Agent Studio is a full agentic chat interface built on top of the Model Context Protocol. You paste one or more MCP server URLs, pick any AI model, and start asking questions. The agent automatically discovers all available tools, decides which ones to call, executes them against your server, and returns results with a natural-language summary.

Every tool invocation is traced live in a side panel — full JSON input and output, so you always know exactly what the model did.

---

## What's new in this release

### 🔗 Multiple MCP Servers per Conversation (up to 4)

You can now connect **up to 4 MCP servers in a single chat**. Tools from all connected servers are merged into one unified tool set. When more than one server is active, tools are automatically namespaced (`s0__tool_name`, `s1__tool_name`) to prevent collisions.

**Example:** Connect your Postgres MCP server *and* your GitHub MCP server in the same conversation, then ask:

> *"Show me the top 5 customers by revenue this month, then open a GitHub issue for each one that hasn't received an update."*

The agent picks the right tools from each server and chains them together automatically.

### 🤖 30+ AI Models — Swap Without Re-configuring

Choose from every major provider and switch mid-conversation:

| Provider | Models |
|---|---|
| Anthropic | Claude Opus 4.6, Sonnet 4.6 |
| OpenAI | GPT-5.4, GPT-4o, o4-mini |
| Google | Gemini 3.1 Pro, Flash, Gemma 4 |
| DeepSeek | DeepSeek V3, R1 |
| xAI | Grok 4, Grok 3 Mini |
| Alibaba | Qwen 3 235B, 32B |
| Mistral | Mistral Small 2603 |
| NVIDIA | Nemotron Super 120B |
| + more | MiniMax, Z.AI GLM, Meta Llama |

Run the same prompt across models to find the cheapest one that gives great results — teams routinely cut inference costs **5–10×** this way.

### 🔍 Live Tool Inspector

Every tool call is traced in real time in a dedicated side panel. You see:
- The exact tool name and which server it came from
- Full JSON input sent to the MCP server
- Full JSON output returned
- Token usage per step

### 💬 Conversation History

All sessions are saved and accessible from the history sidebar. Load any past conversation, continue where you left off, or share context across your team.

### 🪙 Credit-Based Pricing — Free to Start

| Tier | Credits per run | Example models |
|---|---|---|
| Nano | 1 credit | Gemini Flash, Qwen Flash |
| Standard | 3 credits | Claude Sonnet, GPT-4o, Gemini Pro |
| Heavy | 8 credits | Claude Opus, GPT-5.4, Grok 4 |

New accounts receive free starter credits. No credit card required to sign up.

---

## How it works

**1. Add your MCP server(s)**
Paste any HTTP, SSE, or Streamable HTTP endpoint URL. Official servers (GitHub, Supabase, Notion, Slack, Stripe, Linear, Postgres, Playwright) or your own custom server — anything works.

**2. Pick an AI model**
Select from 30+ models. Switch at any point mid-chat to compare.

**3. Ask in plain English**
The agent discovers available tools, calls them with the right arguments, and explains the results. You watch every invocation happen live.

---

## Use cases

- **Query your database** — *"Show me my top 10 customers by revenue this month"* via Postgres / Supabase MCP
- **Explore your GitHub** — *"List all open PRs older than 2 weeks in our main repo"* via GitHub MCP
- **Search your docs** — *"Summarise my customer call notes from this week"* via Notion / Filesystem MCP
- **Monitor Slack** — *"What were the key decisions in #engineering this month?"* via Slack MCP
- **Check payments** — *"Which subscriptions churned last month?"* via Stripe MCP
- **Cross-server workflows** — combine any of the above in a single conversation

---

## FAQ

**Do I need to write code?**
No. Paste a URL, pick a model, start chatting.

**Does it work with my own MCP server?**
Yes — any HTTP, SSE, or Streamable HTTP endpoint. Deploy on Cloudflare Workers, Vercel, Fly, or anywhere else.

**Is my data secure?**
Your MCP credentials stay in your browser session and are never stored on our servers. You can delete conversation history at any time from Settings.

**How is this different from Claude Desktop or ChatGPT?**
Claude Desktop only works with Claude. ChatGPT only works with OpenAI models. MCP Agent Studio works with any model from any provider — swap between Claude, GPT, Gemini, and DeepSeek on the same MCP server without re-configuring anything.

---

## Try it

👉 **[mcpplaygroundonline.com/mcp-agent-studio](https://mcpplaygroundonline.com/mcp-agent-studio)**

Free account · No credit card · Works in your browser

---

## Feedback

We'd love to hear what you build with it. Open an issue, leave a comment, or reach us at [support@mcpplaygroundonline.com](mailto:support@mcpplaygroundonline.com).
