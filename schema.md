# MoodMirror - Database Schema

MoodMirror uses two **n8n Data Tables**. Create them before importing the workflow, then select them in the Data Table nodes.

## 1. `moodmirror_users`

Stores each Telegram chat's consent decision.

| Column | Type | Description |
|--------|------|-------------|
| `chat_id` | string | Telegram chat ID (stored as string, e.g. `"123456789"`) |
| `consent` | boolean | `true` if the user agreed to mood logging, `false` otherwise |
| `consent_updated_at` | dateTime | ISO timestamp of the last consent decision |

Written by: **Insert row** (after the onboarding consent prompt).
Read by: **Get row(s)** and **If row does not exist**.

## 2. `moodmirror_moods`

Stores one row per mood entry the user agreed to save.

| Column | Type | Description |
|--------|------|-------------|
| `chat_id` | string | Telegram chat ID (string) |
| `created_at` | dateTime | ISO timestamp when the entry was saved |
| `primary_emotion` | string | Main emotion detected (Arabic text) |
| `secondary_emotions` | string | JSON array serialized as a string, e.g. `["ضغط","قلق"]` |
| `intensity` | number | Intensity as returned by the model (see known issue on scale) |
| `summary` | string | Short description of the emotions mentioned |

Written by: **Insert row1** (only after the user taps approve on the "save mood?" prompt).

> The original message text is **not** stored - only the analysis fields above.

## 3. LLM structured output (not a table)

The *Structured Output Parser* node forces the model to return this JSON:

```json
{
  "primary_emotion": "إحباط",
  "secondary_emotions": ["ضغط", "قلق"],
  "intensity": 7,
  "confidence": 0.85,
  "summary": "وصف مختصر للمشاعر المذكورة",
  "follow_up_question": "ما أكثر شيء يشغلك الآن؟"
}
```

| Field | Type | Used in reply | Saved to DB |
|-------|------|---------------|-------------|
| `primary_emotion` | string | yes | yes |
| `secondary_emotions` | string[] | yes | yes (as JSON string) |
| `intensity` | number | yes | yes |
| `confidence` | number (0-1) | no | no |
| `summary` | string | yes | yes |
| `follow_up_question` | string | yes | no |

## Example queries / analysis ideas

- Most frequent emotion per user: group `moodmirror_moods` by `chat_id`, count `primary_emotion`.
- Intensity trend over time: sort by `created_at`, plot `intensity`.
- Parse `secondary_emotions` with `JSON.parse()` when you need individual values.

## Privacy notes

- `chat_id` identifies a Telegram account; treat the tables as personal data.
- Provide a way to delete a user's rows on request (not implemented in the current workflow).
