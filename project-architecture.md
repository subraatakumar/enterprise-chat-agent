# Enterprise AI Chat — Project Architecture

## Tech Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Runtime | **Node.js 20+** (LTS) | Backend + serverless function support |
| Backend Framework | **Express.js** | Lightweight API layer, familiar to React devs |
| AI SDK | **`openai`** (latest npm) | Provider-agnostic chat completions + tool calling — same code for Ollama, Azure Foundry, and any OpenAI-compatible cloud |
| Validation | **Zod** | Runtime validation for tool parameters and request/response schemas |
| Secrets | **`dotenv`** | Environment variable management — never hardcode API keys |

### Frontend

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Framework | **React 19** | Hooks, concurrent features |
| Chat UI | **`@assistant-ui/react`** | Streaming chat bubbles, tool execution, markdown — web + React Native (Expo) |
| Markdown | **`@assistant-ui/react-markdown`** | Syntax-highlighted code blocks, product tables in chat |
| Styling | **Tailwind CSS** (web) / **react-native styles** (mobile) | Platform-appropriate styling |
| Icons | **Lucide React** | Tree-shakeable icons |

### Data

- Mock `.json` databases (`products.json`, `orders.json`, `return-policies.json`) — zero dependencies for learning
- Replace with PostgreSQL + Prisma/Drizzle in production

---

## Provider-Agnostic Pattern

The entire application uses the **OpenAI npm SDK** as its AI abstraction. The only thing that changes between environments is the `baseURL` env variable — the tool calling code stays identical.

```
┌───────────────────────────────────────────────┐
│                  CLIENT                       │
│                                               │
│  ┌───────────┐       ┌───────────────────┐   │
│  │  Web      │       │ Mobile (Expo)     │   │
│  │ @assistant-ui/react          │           │   │
│  └──────┬────┘       └─────────┬─────────┘   │
│         └───────────┬───────────┘             │
│                     ▼                         │
│              Chat UI Components               │
└─────────────────┬─────────────────────────────┘
                  │ HTTPS/fetch
                  ▼
┌───────────────────────────────────────────────┐
│               EXPRESS SERVER                  │
│                                               │
│  POST /api/chat → OpenAI SDK                  │
│        │  baseURL ← $AI_PROVIDER_BASE_URL     │
│        ├── Ollama (localhost:11434) — Dev     │
│        └── Azure Foundry (*.openai.azure.com) — Prod  │
│                                               │
│  Tool Calls: order_lookup, product_search, return_eligibility │
└───────────────────────────────────────────────┘
```

### Environment Swapping — One Variable Change

```bash
# Development (free, local, no API credits needed) — OLLAMA ONLY
AI_PROVIDER_BASE_URL=http://localhost:11434/v1
AI_PROVIDER_API_KEY=sk-fake-key # Ollama accepts any non-empty string
MODEL_NAME=llama3.2             # or mistral, phi3, qwen2, etc.

# Production (Azure Foundry) — ZERO CODE CHANGES
AI_PROVIDER_BASE_URL=https://YOUR-RESOURCE.openai.azure.com/openai/v1/
AI_PROVIDER_API_KEY=<your-azure-key>
MODEL_NAME=gpt-4o-mini          # or any Azure-deployed model

# Any OpenAI-compatible provider (AWS Bedrock, GCP Vertex, etc.)
AI_PROVIDER_BASE_URL=https://api.provider.com/v1
AI_PROVIDER_API_KEY=<provider-key>
MODEL_NAME=llama3.2             # provider's model name
```

**Same code. Zero changes.** Just swap the env vars and restart. This is how enterprise apps switch between dev, staging, and production — and it teaches students to think about AI as a pluggable service, not an OpenAI.com dependency.

---

## Architecture

```
┌───────────────────────────────────────────────┐
│                  CLIENT                       │
│                                               │
│  ┌───────────┐       ┌───────────────────┐   │
│  │  Web      │       │ Mobile (Expo)     │   │
│  │ @assistant-ui/react          │           │   │
│  └──────┬────┘       └─────────┬─────────┘   │
│         └───────────┬───────────┘             │
│                     ▼                         │
│              Chat UI Components               │
└─────────────────┬─────────────────────────────┘
                  │ HTTPS/fetch
                  ▼
┌───────────────────────────────────────────────┐
│               EXPRESS SERVER                  │
│                                               │
│  POST /api/chat → OpenAI SDK (gpt-4o-mini)   │
│        │                                      │
│        ├── tools: order_lookup(order_id)      │
│        ├── tools: product_search(query)       │
│        └── tools: return_eligibility(id)      │
│                                               │
│  Auth: API key in env · Validation: Zod       │
└───────────────────────────────────────────────┘
```

---

## Tool Calling Pattern

```typescript
// Core pattern: prompt → tools → parse results → send back → final response
// Works identically for Ollama (local) and Azure Foundry (production)
import OpenAI from 'openai';
import { z } from 'zod';
import { zodToJsonSchema } from 'zod-to-json-schema';

const openai = new OpenAI({
  baseURL: process.env.AI_PROVIDER_BASE_URL,
  apiKey: process.env.AI_PROVIDER_API_KEY,
});

const OrderLookupSchema = z.object({ order_id: z.string() });

async function handleChat(messages: Array<{ role: string; content: string }>) {
  let response = await openai.chat.completions.create({
    model: process.env.MODEL_NAME || 'gpt-4o-mini',
    messages,
    tools: [
      {
        type: 'function' as const,
        function: {
          name: 'order_lookup',
          description: 'Fetch order tracking details by order ID',
          parameters: zodToJsonSchema(OrderLookupSchema),
        },
      },
    ],
  });

  const toolCall = response.choices[0].message.tool_calls?.[0];
  if (toolCall) {
    const args = JSON.parse(toolCall.function.arguments);
    const validatedArgs = OrderLookupSchema.parse(args); // always validate!
    const result = await lookupOrder(validatedArgs.order_id);

    const finalResponse = await openai.chat.completions.create({
      model: process.env.MODEL_NAME || 'gpt-4o-mini',
      messages: [
        ...messages,
        { role: 'assistant', content: null, tool_calls: [toolCall] },
        { role: 'tool', tool_call_id: toolCall.id, content: JSON.stringify(result) },
      ],
    });
    return finalResponse.choices[0].message.content;
  }

  return response.choices[0].message.content;
}
```

---

## Security Checklist

- [ ] `AI_PROVIDER_BASE_URL` and `AI_PROVIDER_API_KEY` in `.env` — never commit to git
- [ ] Zod validation on **all** tool parameters (prevents injection via model output)
- [ ] Rate limiting on Express routes (`express-rate-limit`)
- [ ] Input sanitization before sending to AI provider
- [ ] Mask PII in responses (phone numbers, emails, SSNs)
- [ ] Tool calling max turns limit (prevent infinite loops)

---

## Deployment

| Layer | Options |
|-------|---------|
| Web Frontend | Vercel, Netlify, Cloudflare Pages |
| Mobile App | EAS Build → App Store / Google Play |
| Express API | Railway, Render, Fly.io, AWS EC2, DigitalOcean Droplet |
| AI Backend (dev) | **Ollama** — run locally, free, no credits needed |
| AI Backend (prod) | **Azure Foundry** — deploy any OpenAI-compatible model |
| Database (production) | PostgreSQL + Prisma/Drizzle |

---

## Common Pitfalls

| Issue | Fix |
|-------|-----|
| `@assistant-ui/react` not rendering in Expo | Use Expo SDK 52+ with react-native 0.76+. Run `npx expo install`. Check [react-native docs](https://www.assistant-ui.com/docs/react-native). |
| CORS errors from Express to Expo dev server | Add `cors('http://localhost:19000')` or wildcard during dev |
| Tool calling returns wrong params | Always use `.parse()` on tool output — never skip Zod validation |
| Ollama model not found | Run `ollama pull llama3.2` (or your chosen model) before starting the server |
| .json file not updating during dev | Use `fs.readFileSync` per request during dev, or restart server (Node caches `require()`) |
| Azure Foundry auth fails | Ensure you're using Entra ID token provider: `getBearerTokenProvider(DefaultAzureCredential(), "https://ai.azure.com/.default")` |

---

## Azure Foundry Integration Reference

Microsoft Foundry is Microsoft's unified AI platform — consolidating Azure OpenAI into a single portal. Key points for production deployment:

| Aspect | Details |
|--------|---------|
| **Endpoint** | `https://YOUR-RESOURCE.openai.azure.com/openai/v1/` (v1 API, no explicit `api-version` needed) |
| **Auth** | Entra ID token or API Key — same OpenAI npm package works with either |
| **Model deployment** | Deploy in Azure Portal → Models + endpoints → Deploy model → pick gpt-4o-mini or any OpenAI-compatible model |
| **Tool calling** | Standard `tools` parameter — identical to Ollama. No SDK changes required. |
| **Upgrade path** | Opt-in and reversible from classic Azure OpenAI — preserves existing endpoints, keys, and fine-tunes |

```bash
# Deploy gpt-4o-mini via Azure CLI
az cognitiveservices account deployment create \
  --name <your-resource> \
  --resource-group <your-rg> \
  --deployment-name my-gpt4o-mini \
  --model-name gpt-4o-mini \
  --model-version "2024-07-18" \
  --model-format OpenAI \
  --sku-capacity 1 \
  --sku-name Standard
```

### Why Azure Foundry in Production?

| Factor | Benefit |
|--------|---------|
| **Enterprise security** | VNet integration, private endpoints, RBAC via Entra ID |
| **Built-in monitoring** | OpenTelemetry + App Insights — no custom tracing setup |
| **Multi-model catalog** | 1,900+ models (not just OpenAI) — swap providers without code changes |
| **Single Azure bill** | Models, agents, evals all on one subscription |

> **Source:** [Azure AI Foundry Function Calling Docs](https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/function-calling) · [Working with Models](https://learn.microsoft.com/en-us/azure/foundry/openai/how-to/working-with-models?tabs=powershell) · [Upgrade Guide](https://learn.microsoft.com/en-us/azure/foundry-classic/how-to/upgrade-azure-openai)
