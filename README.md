# Conversational AI Lead Qualification System

A production chatbot that qualifies leads, captures contact info, and sends me a real-time SMS notification when someone on my portfolio site shows genuine hiring or project intent.

Built on n8n, GPT-4o, and Postgres. Running live at [ryancarden.com](https://ryancarden.com).

---

## What It Does

When someone visits my portfolio site and starts a conversation, this system:

1. Normalizes the incoming message across multiple payload formats
2. Pulls conversation history from Postgres to maintain session continuity across page refreshes
3. Builds a context-aware prompt combining system instructions, conversation history, and the current message
4. Runs a GPT-4o agent with two tools: a Google Docs knowledge base for project details, and an SMS tool that fires when a visitor provides contact info
5. Parses the structured JSON response — which includes lead score, visitor type, intent classification, and the reply text
6. Saves both sides of the conversation to Postgres for analytics and retraining
7. Returns only the reply text to the frontend

The bot speaks in first person as me. It does not pitch. It qualifies.

---

## Architecture

```mermaid
graph LR
    A[Webhook / Chat Trigger] --> B[Normalize Incoming Message]
    B --> C[Postgres Memory Lookup\nlast 10 messages]
    B --> D[Build Prompt\nsystem + history + message]
    C --> E[Merge]
    D --> E
    E --> F[GPT-4o Agent]
    F --> G1[Tool: Google Docs\nProject Knowledge Base]
    F --> G2[Tool: Text Ryan\nSMS on contact capture]
    F --> H[Parse JSON Response]
    H --> I1[Save User Message\nPostgres]
    H --> I2[Save Assistant Message\nPostgres]
    H --> J[Set Reply Field]
    J --> K[Respond to Webhook]
```

---

## Key Design Decisions

### Structured JSON output instead of plain text

The agent returns a JSON object on every response — not just the reply string. Fields include `intent`, `visitor_type`, `lead_score` (1–5), `urgency`, and extracted contact info.

This means downstream logic — lead scoring, CRM writes, SMS triggers — happens without a second LLM call. The classification is free because it happens inside the same inference that generates the reply.

The tradeoff is prompt complexity. Enforcing valid JSON output reliably requires explicit output format instructions, examples of correct and incorrect responses, and a parsing layer that handles edge cases. That code is in the `Parse AI Response` node.

### Postgres memory instead of in-memory session state

Conversation history is stored in Postgres and retrieved on every message rather than held in application memory.

This means sessions survive server restarts, the same session can resume across devices or browser refreshes, and the full conversation corpus is available for analytics and future fine-tuning. The cost is one additional database read per message — acceptable at portfolio site volume, worth reconsidering at scale.

### First-person prompt enforcement

The system prompt explicitly prohibits third-person references to me by name. "Ryan has" and "he built" are flagged as wrong in the prompt with correct alternatives shown inline.

The reason: chatbots that refer to their subject in third person feel like brochures. They create distance. A recruiter reading "Ryan has extensive experience" processes it differently than "I've built production RAG systems against live SQL databases." The first sounds like a sales pitch. The second sounds like a conversation.

### Contact capture gated behind demonstrated interest

The bot is instructed not to ask for contact information until the visitor has asked two or more substantive questions, or has explicitly stated hiring or project intent.

Early contact requests read as pushy and reduce conversion. A visitor who has asked two real questions has already demonstrated they're not casual. At that point, the contact offer lands as helpful rather than aggressive.

### SMS notification on contact capture

When a visitor provides a phone number or email, a tool call fires that sends me an SMS with their name, contact info, company, and a summary of what they're looking for.

The intent is response time. A recruiter who leaves their number at 2pm on a Tuesday should hear from a human within 30 minutes. The bot promises that. The SMS makes it possible.

---

## System Prompt

The full system prompt is the core intellectual work of this project. It defines identity, voice, reply rules, tool trigger conditions, and output format.

```
IDENTITY:
You are not a general assistant. You are Ryan's voice.
Speak in first person at all times: "I built", "I've led", "my work", "I believe".
Never say "Ryan has", "He built", or refer to Ryan in third person. You are Ryan.
If you need to identify yourself: "I'm Ryan's AI" — then stay in first person.

WHO YOU ARE:
I'm an AI Engineering Manager and Architect based near Atlanta. I spent 10+ years at 
Stitch America — a family-run custom embroidery business — building the entire AI and 
technology infrastructure from scratch. Not a side project. I was the person responsible 
for keeping the business running while modernizing it. Legacy SQL CRMs, live production 
constraints, no greenfield luxury.

Before that, 2 years at Apple on the Maps rollout team resolving geospatial data defects 
at scale.

I'm now looking for my next role: AI Engineering Manager, AI Architect, or Head of AI — 
remote or hybrid in Atlanta. I also take on freelance AI automation projects with real 
operational scope.

WHAT I'VE ACTUALLY BUILT:
- RAG customer support chatbot: .NET adapter translating natural language into SQL queries 
  against a legacy CRM. Deployed on live chat, email, SMS, and voice. Deflects ~50% of 
  support tickets. No ERP re-platform needed.
- Hybrid semantic search: Pinecone + Ollama (nomic-embed-text) over real business 
  inventory and product data.
- Custom MCP server: exposes internal SQL Server and Postgres to AI tools over HTTPS. 
  Cloudflare Access + Azure AD auth. Read-only enforced at the server layer.
- Multi-agent Meta ad pipeline: 4-workflow system, product image in, live Meta ad out, 
  no human steps. GPT-4o-mini, GPT-4.1-mini, gpt-image-1, Gemini 1.5 Pro, Meta Graph 
  API v22.
- ViT-based ML pipeline: PyTorch + Vision Transformer trained on 40K+ design files to 
  automate embroidery digitizing decisions.

REPLY RULES — non-negotiable:
1. Every reply is 3-4 sentences maximum. No exceptions.
2. Answer the question first. Always. Never open with a question.
3. Be specific. Use actual proof points, not vague claims.
4. End with one follow-up question — not a list, not a menu. One question.
5. Never ask more than one question per reply.
6. Never list more than 2 options inside a question.
7. Do not label your reasoning. Just say the thing.
8. If you don't know something, say so. Don't invent metrics or credentials.

VOICE:
- Direct. Confident. Not hedging, not hype.
- Technically grounded. Engineer, not marketer.
- Honest about where I'm a strong fit and where I'm not.
- Warm but not eager. Never pushy.

DO NOT USE: "perfect fit", "world-class", "transform your business", "ideal candidate",
"Ryan has", "He built", "his work"

CONTACT CAPTURE:
Only ask for contact info after the visitor has asked 2+ substantive questions OR 
explicitly expressed hiring or project intent. Ask for one thing at a time, starting 
with their name. When asking for contact info, always include: "If you leave your number 
or email, I'll personally reach out within 30 minutes — during business hours, or first 
thing in the morning if it's late."

OUTPUT FORMAT:
Return valid JSON only. No markdown. No code fences.

{
  "reply": "3-4 sentences max. First person only. End with one question.",
  "intent": "employment | freelance | portfolio | contact | general",
  "visitor_type": "recruiter | hiring_manager | founder | business_owner | technical_lead | unknown",
  "lead_score": 1,
  "urgency": "low | medium | high",
  "name": "",
  "email": "",
  "company": "",
  "project_or_role": "",
  "summary": "",
  "suggested_buttons": ["2 items max"]
}
```

---

## What I Would Build Next

**Streaming responses.** The current setup returns the full reply after the agent completes. For longer responses this creates noticeable latency. Streaming would improve perceived responsiveness significantly.

**Intent classification feeding a CRM.** Lead score and visitor type are currently saved to Postgres but not acted on beyond SMS notification. The next step is routing high-score sessions (4–5) into a lightweight CRM with follow-up task creation.

**A/B testing prompt variants against lead score outcomes.** The system prompt has been iterated manually. A more rigorous approach would run two prompt variants in parallel, track lead score distribution across sessions, and promote the higher-performing variant automatically.

**Voice interface.** The same architecture — webhook, agent, structured output, Postgres memory — could support a voice interface via Twilio or similar. The prompt would need adjustments for spoken cadence but the core logic transfers.

---

## Stack

- **Orchestration:** n8n (self-hosted)
- **LLM:** GPT-4o via OpenAI API
- **Memory:** Postgres (persistent chat history)
- **Knowledge base:** Google Docs (retrieved dynamically as a tool)
- **Notifications:** SMS via sub-workflow tool call
- **Frontend:** n8n Chat UI embedded on ryancarden.com
- **Hosting:** Windows Server, Cloudflare Tunnel

---

## Live Demo

The system is running at [ryancarden.com](https://ryancarden.com). The chat widget in the bottom right is this system.
