# AICAMSR AI Admissions & RAG Chat Assistant

An end-to-end **n8n automation system** for AICAMSR (All India Council of Alternative Medical Science & Research). It scrapes the AICAMSR website, indexes the content as a searchable knowledge base, captures and deduplicates admission leads, and serves a conversational AI agent that answers course/admissions and general medical questions while promoting AICAMSR admissions.

The system is made up of **three connected n8n workflows**:

| # | Workflow | File | Purpose |
|---|----------|------|---------|
| 1 | **Website Scraper → RAG Ingestion Pipeline** | `web_scrap.json` | Crawls `aicamsr.com`, cleans the content, stores it in Google Sheets, and indexes it into Pinecone for retrieval. |
| 2 | **Register User (Lead Capture Sub-Workflow)** | `Ragister_user.json` | Validates a lead's details, deduplicates by email/phone against Google Sheets, creates new leads, and sends admin + student email notifications. |
| 3 | **AICAMSR RAG Chat Agent** | `Ragretrive.json` | The public-facing chatbot. Greets the user, collects lead info (calling Workflow 2 as a tool), answers course/admission and general medical questions using the Pinecone knowledge base (from Workflow 1). |

---

## 1. Architecture Overview

```
                ┌───────────────────────────┐
                │   1. Scraper & RAG         │
                │   Ingestion Pipeline       │
                │  (web_scrap.json)          │
                │                            │
   Form/URL ───▶│ Fetch pages → Clean text   │
                │ → Google Sheets → Pinecone │
                └─────────────┬──────────────┘
                              │ indexes content into
                              ▼
                     ┌─────────────────┐
                     │  Pinecone Index  │
                     │ "testbyaicamsr"  │
                     └────────┬─────────┘
                              │ retrieved as context
                              ▼
User ──chat──▶ ┌─────────────────────────────┐
               │  3. AICAMSR RAG Chat Agent   │
               │     (Ragretrive.json)        │
               │  - Greets & collects lead    │
               │  - Calls "Register User"     │
               │    tool  ───────────────────┼───┐
               │  - Answers via Pinecone      │   │
               │    Document Search tool      │   │
               │  - Session memory (buffer)   │   │
               └─────────────────────────────┘   │
                                                  ▼
                                   ┌───────────────────────────┐
                                   │ 2. Register User workflow │
                                   │   (Ragister_user.json)    │
                                   │  - Normalize/validate     │
                                   │  - Dedupe by email/phone  │
                                   │  - Append to Google Sheets│
                                   │  - Gmail: admin + student │
                                   └───────────────────────────┘
```

---

## 2. Workflow 1 — Website Scraper & RAG Ingestion Pipeline

**File:** `web_scrap.json`

Crawls the AICAMSR website, extracts clean text from each page, stores raw rows in Google Sheets, and feeds the content into a Pinecone vector store so the chat agent can retrieve it.

### Trigger
- **On form submission** — kicks off the scrape (e.g., submitting a base URL/site to crawl).

### Main flow (route discovery & scraping)
1. **HTTP Request** — fetches the target site/sitemap.
2. **Extract all routes** *(Code)* — parses out the list of page URLs to crawl.
3. **Loop Over Items** — iterates over each discovered route.
4. **Input URL** *(Set)* — prepares the current URL for validation.
5. **Validate URL** *(Code)* → **Is URL Valid?** *(If)*
   - **False** → **Build Invalid URL Error JSON** *(Code)* → looped back / logged.
   - **True** → continues to fetch.
6. **HTTP Request (Fetch Page)** — downloads the page HTML.
7. **Fetch Successful?** *(If)*
   - **False** → **Build Fetch Error JSON** *(Code)*.
   - **True** → **Extract & Clean Data** *(Code)* — strips HTML/boilerplate down to usable text.
8. **Final Output** *(NoOp)* — merges success/error branches.
9. **Make structure** *(Code)* — shapes the final structured row for storage.
10. **Append row in sheet** *(Google Sheets)* — writes the scraped page data into the raw data sheet.

### Indexing branch (RAG ingestion)
1. **Google Sheets (Read Rows)** — reads back the stored scraped rows.
2. **Edit Fields (Combine Columns)** *(Set)* — merges relevant columns into a single text block per document.
3. **Pinecone Vector Store** (`testbyaicamsr` index, insert mode) — receives:
   - **Embeddings OpenAI** — generates vector embeddings for each chunk.
   - **Default Data Loader1** → **Token Splitter1** — loads and splits the combined text into token-sized chunks before embedding.
4. **Success Response** *(Set)* — confirms the batch was indexed.

### Outputs
- A Google Sheet of scraped/cleaned page content (source of truth / audit trail).
- A populated Pinecone index (`testbyaicamsr`) used by Workflow 3 for retrieval.

---

## 3. Workflow 2 — Register User (Lead Capture Sub-Workflow)

**File:** `Ragister_user.json`

An **Execute Workflow / Tool Workflow** invoked by the chat agent (Workflow 3) once it has collected a lead's name, email, phone, and interested course. Handles validation, deduplication, storage, and notifications.

### Trigger
- **Replace me with your logic** *(NoOp placeholder)* — entry point when called as a sub-workflow/tool (receives `name`, `email`, `phone`, `interested_field`, `chat_session_id`, `source`).

### Steps
1. **Normalize and Validate User** *(Code)*
   - Trims/normalizes the name.
   - Lowercases and trims the email.
   - Normalizes the phone number (strips non-digits, auto-prefixes Indian mobile numbers with `+91`, accepts 10–15 digit numbers).
   - Validates name (letters, min length), email format, phone digit count, and interested field.
   - Returns `valid`, `errors`, `error_message`, a derived `user_id` (email or phone), timestamps, and a default `status` of `"Admission Enquiry"`.
2. **User Details Valid?** *(If)*
   - **False** → **Return Validation Error** *(Set)* — returned to the agent so it can ask the user to correct the field.
   - **True** → continues to deduplication.
3. **Search User by Email** *(Google Sheets)* — reads the `AICAMSR Admission Leads` sheet, filters for a matching email.
4. **Email Match Found?** *(If)*
   - **True** → **Return Existing User by Email** *(Set)* — returns `existing_user: true` plus a message instructing the agent *"do not ask for details again, continue with RAG-based assistance."*
   - **False** → **Search User by Phone** *(Google Sheets)* → **Phone Match Found?** *(If)*
     - **True** → **Return Existing User by Phone** *(Set)* — same existing-user behavior as above.
     - **False** → **Create New User** *(Google Sheets — append)* — writes a new lead row.
       - **Send Admin Alert** *(Gmail)* — notifies AICAMSR staff of the new enquiry.
       - **Send User Admission Email** *(Gmail)* — sends a confirmation/welcome email to the student.
       - **Return New User Success** *(Set)* — returns `new_user_success` to the calling agent.

### Return values (consumed by the chat agent)
- `new_user_success` — lead created, notifications sent.
- `existing_user` — lead already on file (matched by email or phone); agent should not re-ask for details.
- `validation_error` — one or more fields invalid; agent should politely re-prompt.

---

## 4. Workflow 3 — AICAMSR RAG Chat Agent

**File:** `Ragretrive.json`

The user-facing conversational agent, built with n8n's LangChain **AI Agent** node.

### Nodes
- **When chat message received** *(Chat Trigger, public webhook)* — entry point for the chat widget; sends an initial greeting ("Hi there! 👋 My name is Akshay...").
- **AICAMSR Agent** *(LangChain Agent)* — the core orchestrator, driven by a detailed system prompt (see below).
- **OpenAI Chat Model** (`gpt-5-mini`) — the LLM powering the agent.
- **Session Memory** *(Buffer Window Memory, last 10 messages)* — keyed by the chat `sessionId`, so context persists across turns in a conversation.
- **AICAMSR Document Search** *(Pinecone Vector Store, `retrieve-as-tool` mode, index `testbyaicamsr`)* — a tool the agent can call to search the scraped AICAMSR knowledge base.
  - **Embeddings** *(OpenAI Embeddings)* — embeds the user's query for similarity search.
- **Register User** *(Tool Workflow → Workflow 2)* — a tool the agent calls with `name`, `phone`, `email`, `interested_field`, and `chat_session_id` to save/lookup a lead.

### System Prompt — Behavior Summary
The agent acts as AICAMSR's official Admissions & Information Assistant with a strict conversation flow:

1. **Scope guard** — only answers medical / alternative-medicine / health / AICAMSR-admissions questions. Politely redirects anything else.
2. **Step 1 — Lead capture**: on the first message, asks (one at a time) for Full Name → Phone → Email → Interested Field/Course (from a fixed list: Bachelor, Diploma, Certificate, M.D., PG Diploma, Ph.D., R.M.P., Other).
3. **Step 1.5 — Mandatory tool call**: as soon as all four details are collected, it **must** call the `Register User` tool before continuing, and handle the result (`new_user_success`, `existing_user`, or `validation_error`).
4. **Step 2 — Course category selection**: lists the specific courses under the chosen category (full course list embedded in the prompt).
5. **Step 3 — Course guidance**: explains the selected course, degree level, duration/eligibility/scope, and relevant recognition context.
6. **Step 4 — Admission push**: encourages the user to proceed with enquiry/admission on the AICAMSR website.
7. **Step 5 — Open medical Q&A**: once the flow is complete, answers general medical/alternative-medicine questions using the `AICAMSR Document Search` (Pinecone) tool for AICAMSR-specific facts, while continuing to promote admissions naturally.

**Hard rules:** never answers unrelated topics, never skips lead capture on a new conversation, never claims to be a licensed doctor or gives diagnoses/prescriptions, and never fabricates AICAMSR-specific facts (fees, dates, recognition) not found in the knowledge base.

---

## 5. Data Flow Summary

1. **Ingestion:** `web_scrap.json` scrapes `aicamsr.com` → stores clean text in Google Sheets → embeds and indexes it into the `testbyaicamsr` Pinecone index.
2. **Conversation:** A visitor opens the chat → `Ragretrive.json`'s AICAMSR Agent greets them, collects lead details, and calls `Ragister_user.json` to validate/dedupe/store the lead and fire off notification emails.
3. **Retrieval:** For course and health questions, the agent queries the Pinecone index (populated in step 1) via the `AICAMSR Document Search` tool to ground its answers in real AICAMSR content.

---

## 6. Setup Requirements

| Service | Used for |
|---|---|
| **n8n** (with LangChain nodes) | Hosting all three workflows |
| **OpenAI API** | Chat model (`gpt-5-mini`) and embeddings |
| **Pinecone** | Vector store / index: `testbyaicamsr` |
| **Google Sheets API** | Raw scraped-content storage + `AICAMSR Admission Leads` sheet |
| **Gmail API** | Admin alert email + student admission confirmation email |

### Credentials to configure in n8n
- OpenAI account (chat + embeddings)
- Pinecone API
- Google Sheets OAuth2
- Gmail OAuth2

### Import order
1. Import `web_scrap.json` and run it once (or on a schedule) to populate Google Sheets and Pinecone.
2. Import `Ragister_user.json` and activate it (it is called as a sub-workflow/tool).
3. Import `Ragretrive.json`, link the **Register User** node to the Workflow 2 ID, link the **AICAMSR Document Search** node to your Pinecone index, and activate the chat trigger.

---

## 7. Notes / Known Placeholders

- Node/credential/workflow IDs in the exported JSON are redacted as `XXXXXXXXXXXXXXXXXXXXXX` — replace with your own IDs after import.
- The **Replace me with your logic** node in `Ragister_user.json` is a placeholder `NoOp` node; when this workflow is called as a tool, its output data (the `$fromAI`-mapped fields) flows into **Normalize and Validate User**.
- The Pinecone index name `testbyaicamsr` should be renamed to a production index name before going live.
