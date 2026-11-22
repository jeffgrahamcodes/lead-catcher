# Lead Catcher

An after-hours AI receptionist for small service businesses. A caller phones the business after closing, gets answers to their questions, books an appointment on the owner's calendar, and the owner wakes up to the lead in their CRM with a summary of the call.

Built with **ElevenLabs** (live voice agent), **n8n** (tools and back office), **Supabase** (knowledge base vector search), **Google Calendar** and **Pipedrive**.

> **Status: in progress.** Sprint runs Nov 21 to 27, 2026. The voice agent, knowledge lookup and CRM subagent work today. Booking, owner notifications and the after-call summary are being built this week. See [Roadmap](#roadmap).

## The problem

Small businesses lose leads every night. A potential client calls at 8pm, gets voicemail, and calls a competitor instead. Calendly only helps once someone has already decided to book. It can't answer "do you handle small business taxes?" or tell a caller you're closed on Sundays.

Lead Catcher picks up the phone, answers from the business's own information, books the appointment, and logs the lead.

## Architecture

The ElevenLabs agent only talks and decides which tool to call. Every action runs as its own n8n webhook, so each piece can be tested alone with curl before it goes on a live call.

```mermaid
flowchart LR
    Caller["Caller<br/>phone or web chat"] <--> Agent["ElevenLabs agent<br/>picks the tool to call"]
    Agent -->|tools| Tools
    subgraph Tools["n8n tools (webhooks)"]
        KL["knowledge_lookup<br/>Supabase vector search"]
        CA["check_availability<br/>Google Calendar"]
        BA["book_appointment<br/>Google Calendar"]
        LL["log_lead<br/>RevOps subagent"]
        NO["notify_owner<br/>hot leads"]
    end
    Agent -->|after the call| Sum["Post-call summary"]
    LL --> CRM["Pipedrive CRM"]
    Sum -->|note| CRM
    CRM --> Dig["7am digest<br/>to the owner"]
```

## What's in this repo

| Path | What it is | Status |
|---|---|---|
| `workflows/knowledge-lookup-rag.json` | Webhook the voice agent calls to answer questions from the knowledge base (Gemini agent + Supabase vector store) | Working |
| `workflows/knowledge-base-ingestion.json` | Loads FAQ rows from a Google Sheet into Supabase with OpenAI embeddings | Working |
| `workflows/revops-subagent.json` | Turns a plain-text lead into structured JSON, then creates the organization, person, lead and note in Pipedrive | Working |
| `workflows/experiments/` | Earlier voice experiments: ElevenLabs calling n8n, and n8n replying with ElevenLabs text-to-speech | Reference |
| `prompts/` | System prompts and output schemas, kept as files so changes show up as readable diffs | |
| `knowledge-base/` | The fictional business's services, hours and FAQ | Coming this week |

## Roadmap

- [x] Live voice agent (ElevenLabs)
- [x] Knowledge lookup over Supabase (RAG)
- [x] RevOps subagent writing leads to Pipedrive
- [ ] Header auth on every webhook
- [ ] `check_availability` (Google Calendar)
- [ ] `book_appointment`
- [ ] `log_lead` wired to the RevOps subagent
- [ ] `notify_owner` for hot leads
- [ ] Post-call summary from the ElevenLabs transcript
- [ ] 7am digest email
- [ ] Real phone number
- [ ] Demo video
- [ ] Move to AWS (Lambda, API Gateway, Bedrock) as a multi-tenant service

## Setup

1. In n8n, import each file in `workflows/` (Workflows → Import from File).
2. Reconnect credentials on each node: OpenAI, Google Gemini, Supabase, Google Sheets and Pipedrive. Credential IDs were removed from the exports.
3. In the ingestion workflow, point the Google Sheets node at your own sheet (`YOUR_GOOGLE_SHEET_ID`) and run it once to load the knowledge base.
4. Add header auth to each Webhook node before activating it.
5. In ElevenLabs, create an agent and add each webhook as a tool, using the same header secret.

The ingestion workflow and the lookup workflow must use the **same OpenAI embedding model**, or retrieval quietly returns poor matches instead of erroring.

## What broke and what I learned

- **Agent tools vs. a fixed workflow.** The first version of the RevOps subagent gave the agent four Pipedrive tools. It sometimes skipped them and returned JSON straight away, because the output parser offered an easier way to finish. Moving the Pipedrive steps out of the agent and into a fixed sequence of nodes (organization, then person, then lead, then note) made every run create all four records. The agent now does only what models are good at: turning messy text into structure.
- **Disabled nodes pass data through.** An organization ID kept coming back `undefined` because the Pipedrive nodes were still switched off from testing. A disabled n8n node hands its input along unchanged, so the next node never saw an `id`.
- **Deleting a person in Pipedrive can take their lead with it.** I thought lead creation was broken. It had worked; I had deleted the test person afterwards.
- **ElevenLabs Voice Library voices need a paid plan through the API.** On the free tier, a Library voice returns `402 Payment Required`. Default voices work.

## Security notes

- No API keys are stored in these files. All secrets live in n8n credentials.
- Saved test data (`pinData`), sheet IDs, webhook paths and email addresses were stripped before publishing.
