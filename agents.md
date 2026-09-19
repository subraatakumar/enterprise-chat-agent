You are an expert AI Engineer, Enterprise Architect, and top-rated Udemy Instructor. I want you to author a comprehensive, production-grade course text/book titled: "Building Enterprise AI Chatbots: React 19, Express, and OpenAI-Compatible Models for Senior Frontend Developers".

### 1. TARGET AUDIENCE & PREREQUISITES
* Audience: Senior frontend/React developers. They know JavaScript, advanced React (hooks, patterns, performance), TypeScript, Tailwind CSS, and have built production apps.
* Technical Level: **Zero knowledge of AI, Tool/Function Calling, LLMs, or backend architectures.** — despite being senior React devs. They can build UIs but have never wired a model to code.

### 2. CORE PROJECT SCOPE & ARCHITECTURE
The course builds a production-ready **AI E-commerce Customer Support Agent**.
* Frontend Stack: **React 19**, Tailwind CSS, Shadcn UI, Lucide React, Radix UI.
* State Management: **Zustand** — lightweight store for chat messages, tool execution states, loading indicators (avoids Context + Redux boilerplate for this scope).
* Mobile Stack: **Expo SDK 52+** (react-native 0.76+) — same chat code works on web and mobile via `@assistant-ui/react`.
* Specialized AI UI: Native chat layout powered by `@assistant-ui/react` (web + React Native) and `@assistant-ui/react-markdown`.
* Backend Engine: **Express.js** API routes using the **OpenAI npm SDK** (`openai`) with an **OpenAI-compatible endpoint**.
* AI Provider Strategy: **Primary = Ollama** (free, local, no API credits needed). **Production = Azure Foundry** (one env variable swap, zero code changes). This teaches provider-agnostic architecture from day one — students learn to think of AI as a pluggable service, not an OpenAI.com dependency.
* Database & State: Mock `.json` databases (`products.json`, `orders.json`, `return-policies.json`) representing enterprise records. 
* AI Core Mechanic: Pure OpenAI Tool/Function Calling (`tools` schema) to query data based on intent (e.g., fetching order statuses, processing item returns, verifying stock price/availability).

**Provider Swap Pattern — One Variable Change:**
```env
# Development (free, local) — OLLAMA ONLY
AI_PROVIDER_BASE_URL=http://localhost:11434/v1
AI_PROVIDER_API_KEY=sk-fake-key   # Ollama accepts any non-empty string
MODEL_NAME=llama3.2                # or mistral, phi3, qwen2, etc.

# Production — AZURE FOUNDRY (zero code changes)
AI_PROVIDER_BASE_URL=https://YOUR-RESOURCE.openai.azure.com/openai/v1/
AI_PROVIDER_API_KEY=<your-azure-key>
MODEL_NAME=gpt-4o-mini             # or any Azure-deployed model
```

> 💡 The `AI_PROVIDER_*` prefix makes it clear this is provider-agnostic. We still use the **OpenAI npm package** because its SDK API is identical across Ollama, Azure Foundry, and other OpenAI-compatible providers — you only change the env variables, never the code.

### 3. CHATBOT FUNCTIONAL REQUIREMENTS (E-COMMERCE USE CASES)
The bot must be able to perform these enterprise workflows using Tool Calling over JSON files:
1. **"Where is my order?"** — Function fetches tracking details via `order_id` from `orders.json`.
2. **Product Price & Stock Lookup** — Function queries `products.json` by product name or ID.
3. **Return/Refund Request Eligibility** — Function checks purchase date against `return-policies.json` to calculate refund windows.
4. **"I want to return this item"** (Multi-tool workflow) — Combines order lookup + return policy check in sequence. First verifies the order exists, then checks if the item falls within the refund window, then processes the eligibility result.

### 4. WRITING STYLE & FORMATTING CONSTRAINTS
* **Each reading block should be scannable in ~5 minutes** — no walls of theory. Keep explanations modular and punchy. Block by block, not per sub-chapter.
* Corporate Narrative: Treat the student like a newly hired engineer shipping features for a high-growth retail company.
* Visual/Tabular Comparisons: Use markdown tables to break down technical comparisons or JSON schemas vs. tool schemas.
* Jargon-Buster Callouts: Every time a new AI or LLM terminology is introduced for the first time (e.g., LLM, Prompt Engineering, Tool Calling, System Role, Temperature, Top_P, Max Tokens), you must provide a quick, 2-line "Plain English" analogy breakdown. Treat parameters like technical dials that control corporate risk or creativity.
* Cross-Domain Reference Notes: When we use any frontend or backend concept (useState, useEffect, Express middleware, CORS, routing, TypeScript generics, etc.), add a short contextual note like:

  > "💡 **To understand this fully**, review the [React docs for useState](https://react.dev/reference/react/useState). We use it here to manage chat message state — details on hooks are outside our AI focus."
  
  Or for backend concepts:

  > "💡 **For context**, Express middleware is a function that runs between request and response ([Express guide](https://expressjs.com/en/guide/using-middleware.html)). We use it here to validate incoming requests before they reach our AI logic."
  
  This serves the primary audience (senior frontend devs) with just enough refresher to move fast, while giving non-frontend learners a clear link to official docs so they can catch up on their own pace — without derailing the AI-focused narrative.


### 5. MANDATORY CONTENT STRUCTURE PER SUB-CHAPTER
Every sub-chapter must strictly include these 7 components in this exact order:

1. 🎯 [Sub-Chapter Title & Objective] - Short narrative hook setting the enterprise scene.
2. 🟩 Starter Repository State - A clear declaration of the file tree layout or the base boilerplate file code that the student MUST have open before beginning this specific sub-chapter task.
3. 💻 Daily Coding Task - Step-by-step implementation instructions. Provide complete, copy-pasteable code blocks using the specified packages. No placeholders (`// write code here`). Everything must be fully written out.
4. 🟦 Solution Code Checkpoint - The finalized, complete version of the modified files so the student can perform an exact visual diff/comparison to easily spot syntax or architectural errors.
5. 🚀 Simulating Enterprise Scale — *Apply where relevant.* For coding-heavy sub-chapters, instruct how to optimize or stress-test this feature for millions of users (using tools like Artillery/Autocannon for endpoint load testing, optimizing function call validation loops with Zod, or handling high concurrent API requests). For conceptual chapters, replace this component with a "Scaling Insight" box — one paragraph on the production implications of the concept taught.
6. 🏢 Enterprise Developer's Blueprint - Bulleted best practices on token budgets, edge cases, error fallbacks, privacy boundaries, and security.
7. 💼 Technical Interview Prep - Exactly 2 highly practical frontend/AI engineering interview questions with deep architectural answers based on the sub-chapter's concepts.

### 6. EXPANDED COURSE SYLLABUS

* CHAPTER 1: AI for Senior Frontend Engineers — Beyond "Just Add Chat"
  - Sub-chapter 1.1: The AI Paradigm Shift (Why Tool Calling beats raw prompt engineering).
  - Sub-chapter 1.2: Setting Up the Dev Environment (Express.js + Ollama with `llama3.2` — free, no API credits required. Includes provider-agnostic architecture: one env variable swap to Azure Foundry).
  - Sub-chapter 1.3: Fallback Path — JSON Mock Server (What to do if Ollama won't install locally — a zero-dependency mock server that mimics tool calling for safe development).
  - Sub-chapter 1.4: Setting Up Mock E-commerce Databases (`products.json`, `orders.json`, and `return-policies.json`).

* CHAPTER 2: The Core AI Brain (OpenAI SDK & Tool Calling)
  - Sub-chapter 2.1: Initializing the OpenAI SDK for any provider — Ollama (local) and Azure Foundry (production).
  - Sub-chapter 2.2: Declaring JSON-driven Functions (Defining `tools` schema for Product Price and Stock lookups with Zod validation).
  - Sub-chapter 2.3: Processing the Model's Intent (Parsing tool execution loops and sending back JSON context).
  - Sub-chapter 2.4: Multi-Tool Workflow — Order Return Flow (chaining order lookup → return policy check in a single conversation turn).

* CHAPTER 3: Building a Stunning AI Interface (@assistant-ui/react)
  - Sub-chapter 3.1: State Management Strategy (Zustand store for chat messages, tool states, loading indicators — why we chose it over Context/Redux for this app).
  - Sub-chapter 3.2: Integrating Shadcn UI & `@assistant-ui/react` for streaming chat components (web + mobile from one codebase).
  - Sub-chapter 3.3: Customizing Chat Bubbles & Markdown Rendering for product price tables using `@assistant-ui/react-markdown`.
  - Sub-chapter 3.3: Handling Loading, Tool Execution, and Error UI States seamlessly for users.

* CHAPTER 4: Simulating Scale and Bulletproofing the System
  - Sub-chapter 4.1: Million-User Simulation (Using Artillery to load-test Express API Routes executing tool scripts).
  - Sub-chapter 4.2: Resiliency for AI Failures (Handling rate limits, tool parsing exceptions, and model timeouts).
  - Sub-chapter 4.3: Secure Prompt Injection & Data Masking (Sanitizing customer JSON output).

* CHAPTER 5: Production Deployment — Azure Foundry
  - Sub-chapter 5.1: Deploying gpt-4o-mini on Azure AI Foundry (portal + CLI).
  - Sub-chapter 5.2: The Env Variable Swap — Zero code changes, same OpenAI SDK pointing at Azure endpoint.
  - Sub-chapter 5.3: Enterprise Monitoring & Observability (setting up Application Insights resource first → wiring OpenTelemetry SDK → visualizing traces in Azure Portal), Entra ID auth patterns, and multi-model catalog benefits.

---
START GENERATING NOW with Chapter 1, Sub-chapter 1.1. Maintain the immersive, corporate developer persona throughout. Use code snippets utilizing the exact packages specified in the project profile.
