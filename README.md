# MoodMirror

> A consent-first Egyptian-Arabic emotional reflection bot for Telegram, built with **n8n** and a **local LLM (Ollama)**.

MoodMirror reads a message the user sends on Telegram, identifies the emotions in it **without diagnosing or assuming causes**, and replies in natural Egyptian Arabic with a short reflection and a single follow-up question. Mood summaries are stored **only if the user explicitly agrees**.

![Architecture](assets/architecture.png)

## Features

- Telegram bot interface (no app to install)
- Egyptian-Arabic replies, 80-120 words, one follow-up question at most
- Structured emotion analysis: primary emotion, secondary emotions, intensity, confidence, summary, follow-up question
- Runs on a **local model** (`qwen3.5:9b` through Ollama), so message text is not sent to a cloud LLM
- **Two consent layers**: a one-time onboarding consent, and a per-message "save this mood?" prompt
- Mood history stored in n8n Data Tables (`moodmirror_users`, `moodmirror_moods`)

## Repository structure

```
MoodMirror_GitHub/
├── README.md
├── LICENSE
├── .gitignore
├── workflow/
│   └── MoodMirror_Workflow_TEMPLATE.json   # sanitized n8n workflow (no credentials)
├── database/
│   └── schema.md                           # data tables + LLM output schema
├── docs/
│   ├── MoodMirror_Documentation.md
│   └── MoodMirror_Documentation.pdf
└── assets/
    └── architecture.png
```

## Requirements

| Component | Notes |
|-----------|-------|
| n8n | A recent version with **Data Tables** and the LangChain nodes (workflow uses `executionOrder: v1`) |
| Ollama | Running and reachable from n8n, with the model pulled: `ollama pull qwen3.5:9b` |
| Telegram bot | Create one with [@BotFather](https://t.me/BotFather) and keep the token |

## Quick start

1. **Create the two Data Tables** in n8n exactly as described in [`database/schema.md`](database/schema.md).
2. **Import the workflow**: n8n → *Workflows* → *Import from File* → `workflow/MoodMirror_Workflow_TEMPLATE.json`.
3. **Create credentials** in n8n: a *Telegram API* credential (bot token) and an *Ollama* credential (base URL, e.g. `http://localhost:11434`).
4. **Re-link placeholders** in the imported workflow:
   - every Telegram node → your Telegram credential
   - *Ollama Chat Model* → your Ollama credential
   - the Data Table nodes → select your `moodmirror_users` / `moodmirror_moods` tables (the template contains `REPLACE_WITH_...` IDs)
5. **Activate** the workflow and send a message to your bot.

For a node-by-node explanation see [`docs/MoodMirror_Documentation.md`](docs/MoodMirror_Documentation.md).

## Privacy and safety

- MoodMirror is a **reflection tool, not a medical or therapeutic service**. It does not diagnose.
- Mood data is sensitive. Keep your n8n instance and Data Tables access-controlled, and never commit real exports (see `.gitignore`).
- The current version has **no crisis/self-harm detection**. Before offering the bot to others, add an escalation path (e.g. a safety reply with local helpline information).
- Never commit your bot token, Ollama credentials, or real `chat_id` values.

## Known issues

A few behaviours in the current flow are worth fixing before a public release (details and suggested fixes in the documentation, section 9):
intensity scale mismatch (`x10`), possible duplicate consent prompts for new users, the dead-end `Merge1` node, and the test chat trigger referencing a Telegram-only node.

## نبذة بالعربي

**MoodMirror** بوت تيليجرام بيساعد المستخدم يفهم مشاعره بالعربي المصري. بيحدد المشاعر من غير تشخيص ومن غير ما يفترض أسباب، وبيسأل سؤال متابعة واحد بس. تحليل المشاعر بيتحفظ **فقط بعد موافقة المستخدم**، والنموذج بيشتغل محليًا عن طريق Ollama. شوف خطوات التشغيل فوق.

## License

MIT - see [`LICENSE`](LICENSE).
