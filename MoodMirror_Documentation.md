# MoodMirror - Documentation

## 1. Overview

MoodMirror is a Telegram bot built as an n8n workflow. It reads a user's message, identifies the emotions expressed, and replies in natural Egyptian Arabic with a short reflection and one follow-up question. It runs on a local LLM through Ollama and stores mood summaries only when the user gives consent.

Design goals:

- Understand before advising: no diagnosis, no assumed causes, no generic motivation.
- Privacy by consent: nothing is logged unless the user agrees.
- Local inference: the model runs on the owner's machine.

## 2. Architecture

See `assets/architecture.png`. The workflow has four stages:

1. **Ingestion** - Telegram Trigger receives the message; Edit Fields1 extracts `message` and `chat_id`.
2. **Consent gate** - the workflow looks up the user in `moodmirror_users`. Unknown or non-consenting users get the onboarding consent prompt.
3. **Analysis and reply** - the Basic LLM Chain (Ollama + Structured Output Parser) produces structured JSON; Edit Fields formats it; Telegram sends the reply.
4. **Mood logging** - a second prompt asks whether to save this specific analysis; if approved, it is written to `moodmirror_moods`.

## 3. Technology stack

| Layer | Technology |
|-------|------------|
| Orchestration | n8n (workflow `executionOrder: v1`) |
| Channel | Telegram Bot API (Trigger + Send nodes) |
| LLM runtime | Ollama, model `qwen3.5:9b` |
| Output control | n8n Structured Output Parser (JSON schema example) |
| Storage | n8n Data Tables |

## 4. Node-by-node reference

| Node | Type | Purpose |
|------|------|---------|
| Telegram Trigger | telegramTrigger | Receives `message` and `callback_query` updates |
| Edit Fields1 | set | Extracts `message` (text) and `chat_id` (number) |
| Get row(s) | dataTable (get) | Looks up the chat in `moodmirror_users` (limit 1, always outputs data) |
| If1 | if | Passes if `consent == true`; otherwise goes to the consent prompt |
| If row does not exist | dataTable (rowNotExists) | Detects first-time users and sends the consent prompt |
| Send message and wait for response | telegram (sendAndWait) | Onboarding consent with Approve / Disapprove buttons |
| Insert row | dataTable | Saves `chat_id`, `consent`, `consent_updated_at` |
| If | if | Continues to analysis only if consent was approved |
| Basic LLM Chain | chainLlm | Runs the MoodMirror prompt and returns parsed JSON |
| Ollama Chat Model | lmChatOllama | `qwen3.5:9b`, temperature 0.8, `numCtx` 4096, `numPredict` 256, thinking off, keepAlive 5m |
| Structured Output Parser | outputParserStructured | Enforces the six-field JSON schema |
| Merge | merge (by position) | Joins the LLM output with the original `chat_id` |
| Edit Fields | set | Builds the formatted `reply` string |
| Send a text message | telegram | Sends the reply to the user |
| Send message and wait for response1 | telegram (sendAndWait) | Asks whether to save this mood entry |
| If2 | if | Checks `data.approved == true` |
| Insert row1 | dataTable | Writes the mood entry to `moodmirror_moods` |
| Merge1 | merge | Currently a dead end (see known issues) |
| When chat message received | chatTrigger | n8n test chat entry point (see known issues) |

## 5. Data flow (one message)

1. User sends text to the bot.
2. `message` and `chat_id` are extracted.
3. Consent is looked up. If missing or false, the onboarding prompt is shown; the flow continues only on approval.
4. The LLM returns JSON: `primary_emotion`, `secondary_emotions`, `intensity`, `confidence`, `summary`, `follow_up_question`.
5. The reply is built from those fields and sent.
6. The user is asked whether to save this analysis; on approval it is inserted into `moodmirror_moods`.

Reply format:

- Primary emotion
- Accompanying emotions
- Intensity (as a score out of 10)
- Reflection (the summary)
- One follow-up question

## 6. Prompt design

The system prompt is bilingual (English rules plus Arabic rules). Key principles:

- Use natural Egyptian Arabic, not formal Arabic.
- Identify emotions without diagnosing; never claim to know exactly how the user feels.
- Never assume causes; do not read comparison as fear, jealousy, or insecurity unless stated.
- No advice unless requested or clearly useful; understand first.
- 80-120 words, at most one follow-up question, avoid repeating the user's message.
- Vary openings; avoid canned templates.
- Return only valid JSON matching the parser schema (no Markdown, no code fences).

## 7. Setup

1. Create the data tables from `database/schema.md`.
2. Pull the model: `ollama pull qwen3.5:9b`.
3. Create a Telegram bot with BotFather and add its token as an n8n Telegram credential.
4. Add an Ollama credential in n8n.
5. Import `workflow/MoodMirror_Workflow_TEMPLATE.json`.
6. Re-link credentials and replace the `REPLACE_WITH_...` table IDs.
7. Activate the workflow and message the bot.

## 8. Privacy and safety

- The tool offers reflection, not diagnosis or treatment.
- Consent is requested at onboarding and again before each saved entry.
- Only analysis fields are stored; the original message is not.
- Credentials, real `chat_id` values, and data exports must never be committed.
- No crisis detection exists yet. Before wider release, add detection of self-harm or acute distress and a safe response with local helpline information.
- Add a command or flow for users to delete their data.

## 9. Known issues and suggested fixes

| # | Issue | Suggested fix |
|---|-------|---------------|
| 1 | **Intensity scale mismatch.** The schema example uses 7 (a 1-10 scale), but the reply formula multiplies by 10, so 7 displays as 70/10. | Remove `* 10` from the formula, or change the prompt/schema to a 0-1 scale. |
| 2 | **Duplicate consent prompts.** Both `If row does not exist` and `If1` (false branch) lead to the same consent prompt, so a new user can be asked twice. | Keep one path: use only `Get row(s)` + `If1`, or only `If row does not exist`. |
| 3 | **Declined users are re-asked on every message.** A declined user has `consent = false`, so `If1` sends them to the consent prompt again, and a new row is inserted each time. | Upsert by `chat_id`; decide whether declined users may still receive replies without logging. |
| 4 | **Merge1 is a dead end** and `Insert row1` is reached only through the reply chain. | Remove Merge1 or connect it to its intended purpose. |
| 5 | **Test chat trigger breaks.** `When chat message received` feeds the LLM chain, whose input reads from `Edit Fields1` (Telegram only). | Delete the trigger or give it its own input mapping. |
| 6 | **Stray tab** at the start of the `secondary_emotions` expression in Insert row1 (`=\t{{ ... }}`). | Remove the tab so the stored value is clean JSON. |
| 7 | **Empty `approverIds`** on the onboarding prompt and a hard-coded English label (`Disapprove`) beside Arabic text. | Set the approver ID to the chat ID and use an Arabic label. |
| 8 | **Output length limit.** `numPredict: 256` can truncate Arabic JSON and break the parser. | Raise `numPredict` (e.g. 512) and add retry-on-fail for the parser. |
| 9 | **`confidence` is generated but never used or stored.** | Store it, or drop it from the schema. |

## 10. Roadmap ideas

- Weekly mood summary sent automatically.
- Charts of intensity over time.
- Crisis-keyword detection with a safe escalation reply.
- `/delete_my_data` and `/export` commands.
- Multi-language support.
