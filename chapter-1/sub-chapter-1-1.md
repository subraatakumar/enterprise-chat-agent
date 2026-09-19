# Chapter 1: Next-Gen AI Stack for Frontend Engineers

> *You're a new frontend engineer at NexusRetail, a high-growth DTC fashion brand.* Your CTO says: *"By Friday we need a working AI agent on staging."* This chapter gets your environment ready and shifts your mindset from traditional CRUD apps to intent-driven AI applications.

---

## Sub-chapter 1.1 — The AI Paradigm Shift (Why Tool Calling Beats Raw Prompt Engineering)

### 🎯 Objective & Narrative Hook

Most developers still treat LLMs like **magic textboxes**: write a long prompt, hope the model "understands," and pray it doesn't hallucinate. That approach worked in 2023. It breaks in production.

Tool/Function Calling changes the game entirely -- you stop *telling* the model what to do, and start *describing* your app's actual APIs as executable functions. The model picks which function to call with what arguments. You execute it server-side. This turns unstructured LLM output into **structured, auditable enterprise workflows**.

Before we write a single line of code, let's define the key terms so nothing gets lost in translation:

::jargon-buster
**LLM (Large Language Model):** Basically a super-powered autocomplete trained on everything from books to docs.stackoverflow.com. It "speaks" in tokens (chunks of 1-4 characters), not words you'd write in JSON or SQL.
::

::jargon-buster
**Tool Calling:** A formal API feature that lets the model emit a structured function-invocation request instead of making stuff up. You define the rules (the schema); it follows them.
::

::jargon-buster
**JSON Schema:** The "rulebook" you give an LLM so it outputs valid data — like a TypeScript type that also describes its own structure. If the model's output doesn't match, it's wrong. Period.
::

::jargon-buster
**Zod:** A validation library for JavaScript/TypeScript. Think of it like PropTypes for React but at runtime. It catches malformed data before it touches your database.
::

### 🟩 Starter Repository State

Open this exact file tree in VS Code starting from your project root:

```
nexus-retail-ai-agent/          ← you created this with create-next-app
├── .env.local                  ← add OPENAI_API_KEY here (never commit!)
├── package.json                ← after npm install (see Step 1)
├── tsconfig.json               ← provided by create-next-app
└── src/
    ├── lib/
    │   └── tools/
    │       └── schemas.ts      ← CREATE THIS
    └── app/
        ├── api/
        │   └── chat/
        │       └── route.ts    ← CREATE THIS
        └── lib/
            └── ai-client.ts    ← CREATE THIS
```

You should have **zero** TypeScript files in `src/lib/tools/` or `src/app/api/chat/` right now. This is your blank canvas.

### 💻 Implementing Your First Tool Call in Next.js 16

#### Step 1: Install Dependencies

```bash
npm install openai zod zod-to-json-schema
```

| Package | Why You Need It in This Project |
|---|---|
| `openai` (v4+) | Official OpenAI Node.js SDK — handles Chat Completions API + Tool Calling types out of the box. No wrapper layer needed. |
| `zod` | Runtime validation for tool input/output. Catches malformed LLM output before your DB crashes. Think of it as a safety net between the model and your business logic. |
| `zod-to-json-schema` | Auto-generates OpenAI-compatible JSON schemas from Zod types. One source of truth — never write the schema twice, which means no drift when you change a field. |

#### Step 2: Define the Tool Schemas (One Source of Truth)

Create **`src/lib/tools/schemas.ts`**:

```typescript
// src/lib/tools/schemas.ts
import { z } from 'zod';
import { zodToJsonSchema } from 'zod-to-json-schema';

// ===== Zod Schemas (validation rules for our e-commerce tools) =====

export const ProductStockSchema = z.object({
  product_id: z.string().min(1, 'product_id is required'),
  quantity: z.number().int().positive().max(50).optional(),
});

export const OrderStatusSchema = z.object({
  order_id: z
    .string()
    .regex(/^ORD-\d{5}$/g, 'Order ID must be ORD-XXXXX (exactly 5 digits)'),
});

export const ReturnEligibilitySchema = z.object({
  order_id: z.string().regex(/^ORD-\d{5}$/g),
  reason: z.enum(['defective', 'wrong_item', 'changed_mind', 'late_delivery']),
});

// ===== OpenAI Tool Declarations =====
// Each entry is converted to JSON Schema automatically by zod-to-json-schema.
// The model receives these rules and must output arguments matching them exactly.

export function getTools() {
  return [
    {
      type: 'function' as const,
      function: {
        name: 'get_product_stock',
        description: "Look up a product's price available variants and current stock level from the NexusRetail inventory database.",
        parameters: zodToJsonSchema(ProductStockSchema),
      },
    },
    {
      type: 'function' as const,
      function: {
        name: 'get_order_status',
        description: "Fetch the current tracking status courier company and estimated delivery date for a customer order.",
        parameters: zodToJsonSchema(OrderStatusSchema),
      },
    },
    {
      type: 'function' as const,
      function: {
        name: 'check_return_eligibility',
        description: "Check whether an order qualifies for return or refund based on purchase date and reason.",
        parameters: zodToJsonSchema(ReturnEligibilitySchema),
      },
    },
  ];
}

// TypeScript types inferred from Zod — used in our client-side code below.
export type ProductStockInput = z.infer<typeof ProductStockSchema>;
export type OrderStatusInput = z.infer<typeof OrderStatusSchema>;
export type ReturnEligibilityInput = z.infer<typeof ReturnEligibilitySchema>;
```

#### Step 3: Create the Tool Calling Handler (`src/app/api/chat/route.ts`)

| Concept | Plain English Equivalent |
|---|---|
| `finish_reason: 'tool_calls'` | "The model wants to make a function call" — not a text reply. Check this before you try to read `.content`. |
| `max_tokens` | How many words the model can spew back. Set it or the model runs wild on responses and your cost bill goes up. |
| `temperature` | Creativity dial. 0 = deterministic/consistent, 1 = creative/unpredictable. For tool calling, keep at 0.7 max — you want predictable function names, not "creative" ones. |

Create **`src/app/api/chat/route.ts`**:

```typescript
// src/app/api/chat/route.ts
import { NextResponse } from 'next/server';
import OpenAI from 'openai';
import { getTools } from '@/lib/tools/schemas';

// Singleton pattern: one HTTP client per server process.
// If you create `new OpenAI()` inside the handler, every request opens
// a new TCP connection pool → eventually crashes at scale with "EMFILE".
const openai = new OpenAI({ apiKey: process.env.OPENAI_API_KEY });

export async function POST(req: Request) {
  const { messages } = await req.json();

  // Pull tools — they're generated from Zod, so one source of truth.
  const tools = getTools();

  const response = await openai.chat.completions.create({
    model: process.env.OPENAI_DEFAULT_MODEL || 'gpt-4o-mini',
    messages: messages,
    tools: tools,
    tool_choice: 'auto',        // Let the model decide whether to call a tool.
    max_tokens: Number(process.env.OPENAI_MAX_TOKENS) || 1024,
    temperature: Number(process.env.OPENAI_TEMPERATURE) || 0.7,
  });

  const choice = response.choices[0];

  if (choice.finish_reason === 'tool_calls') {
    return NextResponse.json({
      role: choice.message.role,
      tool_calls: choice.message.tool_calls,
    });
  }

  // Normal text response — no function call needed.
  return NextResponse.json({
    role: 'assistant',
    content: choice.message.content || '',
  });
}
```

#### Step 4: The Multi-Turn Loop (Client-Side)

::jargon-buster
**Multi-turn conversation:** A back-and-forth where the model asks *you* to do something in between turns — like fetch real data via tool calls. This is what makes AI feel "smart": the model reads your backend's results and formulates a natural-language reply.
::

Create **`src/app/lib/ai-client.ts`**:

```typescript
// src/app/lib/ai-client.ts
'use client';

import type { ProductStockInput, OrderStatusInput, ReturnEligibilityInput } from '@/lib/tools/schemas';

export async function callChatAPI(
  messages: Array<{ role: string; content?: string }>
) {
  const response = await fetch('/api/chat', {
    method: 'POST',
    headers: { 'Content-Type': 'application/json' },
    body: JSON.stringify({ messages }),
  });

  const data = await response.json();

  if (data.tool_calls && data.tool_calls.length > 0) {
    // Push tool call to history first — tells the model you're acknowledging it.
    messages.push({ role: 'assistant', content: null, tool_calls: data.tool_calls });

    for (const tc of data.tool_calls) {
      const args = JSON.parse(tc.function.arguments);
      let result: Record<string, unknown>;

      switch (tc.function.name) {
        case 'get_product_stock':
          result = await executeProductLookup(args as ProductStockInput);
          break;
        case 'get_order_status':
          result = await executeOrderTracking(args as OrderStatusInput);
          break;
        case 'check_return_eligibility':
          result = await executeReturnCheck(args as ReturnEligibilityInput);
          break;
        default:
          throw new Error(`Unknown tool: ${tc.function.name}`);
      }

      console.log(`[${tc.function.name}]`, result);

      // Push real data back as a "tool" message — the model reads this.
      messages.push({
        role: 'tool',                    // Special role OpenAI SDK expects.
        tool_call_id: tc.id,             // Links back to the original tool call ID.
        content: JSON.stringify(result),
      });
    }

    // Third call — model sees real data and formats it for the user.
    const nextResponse = await fetch('/api/chat', {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({ messages }),
    });

    return nextResponse.json();
  }

  return data;
}

// ===== Real Execution Logic (Stubbed for now — powered by .json databases in Chapters 2–3) =====

async function executeProductLookup(args: ProductStockInput) {
  // Replaced in Ch2.3 when we read from products.json
  return { found: false, message: `Product ${args.product_id} not yet connected to live DB` };
}

async function executeOrderTracking(args: OrderStatusInput) {
  return {
    order_id: args.order_id,
    status: 'shipped',
    estimated_delivery: '2024-12-28T15:30:00Z',
    courier: 'UPS Ground',
    tracking_url: 'https://tracking.example.com/' + args.order_id,
  };
}

async function executeReturnCheck(args: ReturnEligibilityInput) {
  return { eligible: true, days_remaining: 30, reason_accepted: args.reason };
}
```

### 🟦 Solution Code Checkpoint

After finishing Steps 1–4, your `src/` directory should look like this:

```
src/
├── lib/
│   └── tools/
│       └── schemas.ts              ← Zod schemas + getTools()
├── app/
│   ├── api/
│   │   └── chat/
│   │       └── route.ts            ← OpenAI handler with multi-path response logic
│   └── lib/
│       └── ai-client.ts            ← Browser-side multi-turn loop + stub executors
```

**Diff Checklist — verify each match:**
- [x] `schemas.ts` — exports 3 Zod schemas, 1 `getTools()`, and 3 inferred types
- [x] `route.ts` — imports `getTools()`, uses singleton OpenAI, handles both `tool_calls` and text paths
- [x] `ai-client.ts` — `'use client'`, pushes tool call history + real results, makes third API call for the final answer

If you're missing any of these or a file doesn't match, check TypeScript errors before moving on.

### 🚀 Simulating Enterprise Scale

We'll start with load testing and scale-ready patterns now so you never rebuild later.

#### Step 1: Install Artillery for Load Testing

```bash
npm install -g artillery
```

#### Step 2: Create `load-test.yml` (at project root)

| Term | Plain English |
|---|---|
| **arrivalRate** | How many virtual users start sending requests per second. Start small (5) and ramp up to see where your endpoint cracks. |
| **rampTo** | The peak number of concurrent users you want to simulate. In production, NexusRetail might hit 100–500 concurrent shoppers during sales events. |

```yaml
config:
  target: "http://localhost:3000"
  phases:
    - duration: 60        # Run for 1 minute
      arrivalRate: 5     # Start at 5 users/sec
      rampTo: 25         # Gradually increase to 25 users/sec
  scenarios:
    - flow:
        - post:
            url: "/api/chat"
            json:
              messages:
                - role: user
                  content: "Where is my order #ORD-98765?"
```

#### Step 3: Run the Test

```bash
artillery run load-test.yml
```

Watch the terminal output. Focus on these metrics:
| Metric | What to Look For |
|---|---|
| **avg/min/max (ms)** | If average latency > 2 seconds, your customers will abandon. Target <1s for the first API call. |
| **http.codes** | Any `429` = rate limited by Vercel/your infrastructure. Any `500` = crash — fix before scaling. |
| **http.errors** | Must read zero. If you see errors at 25 users, they compound fast at 100+. |

#### Scaling Checklist for Millions of Users

| Issue | Fix Now / Future-Proofing |
|---|---|
| OpenAI API cost per request | Cache repeated order lookups in Redis — don't pay the model for data that doesn't change every second |
| Zod validation per request | `getTools()` already precompiled schemas at import time. For 10K RPS, move JSON schema output to a `const` and skip runtime conversion entirely |
| Tool call latency > user patience | Stream partial UI state with `@assistant-ui/react` progress events — show "Checking your order…" while the model fetches data |
| Unhandled JSON parse errors | Always wrap `JSON.parse(tc.function.arguments)` in try/catch + Zod validation before business logic runs |

### 🏢 Enterprise Developer's Blueprint — Best Practices

- **ALWAYS validate tool arguments with Zod.** LLMs sometimes output malformed JSON. Zod catches it server-side before your DB crashes or a null reference destroys production data.
- **Never trust the model's function name match.** Validate `tc.function.name === 'allowed-name'`. This prevents injection attacks where the model calls functions you didn't intend for users to invoke.
- **Set max_tokens explicitly.** GPT-4o-mini charges per-token. Long tool outputs → long responses → wasted money. Cap it at ~1024 for this use case (orders, product lookup are short by nature).
- **Use gpt-4o-mini not gpt-4 for first drafts.** It's 10x cheaper and already excellent at tool calling. Upgrade to Claude or GPT-4 only when accuracy matters for complex reasoning.
- **Keep the tools list under 25 items.** Each tool adds ~200–300 tokens to every prompt sent to the model. Beyond 25 your context window fills up and accuracy drops sharply.
- **Singleton OpenAI client.** Instantiate `new OpenAI()` once at module top-level. Per-request instantiation creates a new TCP pool — kills you under load with "EMFILE: too many open files".
- **Mask customer PII before sending to the model.** Only send `order_id` and `status` to the LLM response stream. Never echo back emails, phone numbers, or addresses unless the user explicitly requested them.

### 💼 Technical Interview Prep

#### Question 1: "Explain the difference between raw prompts and function/tool calling in enterprise contexts."

> **Answer:**
> Raw prompting forces the LLM to predict text it was never trained on (`live_order_status`, `current_inventory_level`) — essentially asking a weather forecast from someone who's read every newspaper. Tool calling lets the model *choose from your app's real functions via a structured schema*. In production this matters because:
> - **Accuracy**: Model returns factual data only when hitting your API rather than making up an order status from its training data.
> - **Observability**: Every tool call produces a structured log entry for audit trail — mandatory in regulated enterprise environments (SOX, GDPR).
> - **Determinism**: The same valid `order_id` always yields the same DB query and result regardless of model version changes, as long as your backend is consistent.

#### Question 2: "When would you reject a tool_call from an LLM, and how do you handle it?"

> **Answer:**
> You reject when:
> - The parsed arguments fail Zod validation (`safeParse` returns `false`).
> - The function name isn't in your allowed list.
> - Required fields are missing or invalid (e.g., `order_id` doesn't match the `^ORD-\d{5}$` pattern).
> 
> Handling: **Do NOT forward garbage to production.** Catch the error server-side, log it with details (`ZodError.name`, input snapshot), and push a structured tool result back saying `"Invalid request format — please provide order ID in ORD-XXXXX format."`. The model then reformulates its next attempt using this real feedback (it learns from structured data, not text fluff). If this repeats excessively (>3 times), implement a fallback: ask the customer for clarification or escalate to human support via your queue manager.

---

> **Next Up**: Sub-chapter 1.2 — Deep Dive into Tech Stack (Next.js 16 setup with `create-next-app` flags, React Server Components, Shadcn UI, and `@assistant-ui/react`)
