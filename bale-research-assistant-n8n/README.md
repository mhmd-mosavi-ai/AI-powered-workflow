# Bale Research Assistant (n8n)

> 🇮🇷 [Persian version below / نسخه فارسی در ادامه](#-نسخه-فارسی)

A research assistant that runs as a **Bale messenger bot** on top of **n8n**. A user sends a research question (text or voice), the bot restates it as a structured brief and asks for confirmation, then a research agent searches the web and writes a source-backed report. A second, independent reviewer agent checks every source and claim before the report is delivered.

The prompts and bot messages are in **Persian** (the bot is built for Persian-speaking users). Node names, code structure and this documentation are in English.

## Architecture

Two workflows work together:

| File | Workflow | Role |
|---|---|---|
| `workflows/01-bale-intake-and-confirmation.json` | Psychology Research Assistant - Bale Intake & Confirmation | Receives Bale updates, handles commands, transcribes voice, extracts a brief, asks the user to confirm, starts the research engine |
| `workflows/02-research-engine.json` | Research Engine - Full Report with Independent Review | Runs the researcher agent, reviews it with a second agent, applies an evidence gate, sends the report to Bale and stores results |

### Workflow 01 – Intake and confirmation

```
Bale Webhook → Route Input
  ├─ button "confirm" → Answer Callback Query → Get Session (Confirm)
  │                      → Notify Research Started → Run Research Engine (async)
  ├─ /start           → Send Welcome Message
  ├─ /new             → Reset Session → Notify Session Reset
  ├─ voice message    → Get Voice File Path → Download Voice File → Transcribe Voice (fa)
  │                      → Prepare Voice Input ┐
  └─ text message     → Prepare Text Input ────┴→ Normalized Input
                          → Get Session → Extract Brief (LLM + structured output)
                          → Build Confirmation Payload → Save Session (Awaiting Confirmation)
                          → Send Confirmation Message (inline "confirm" button)
```

### Workflow 02 – Research engine

```
Research Request Trigger → Prepare Research Context → Researcher Input
  → Main Researcher (LLM + web search)
  → Package Report for Review → Independent Source Reviewer (LLM + web search)
  → Review Decision → Route by Review
       ├─ approved → Apply Evidence Gate ─┬→ Format Bale Messages → Send Messages Loop → Send Bale Message
       │                                  ├→ Split Evidence Rows → Save Evidence
       │                                  ├→ Save Research
       │                                  └→ Update Session
       ├─ revise   → back to Researcher Input (at most ONE revision round)
       └─ failed   → Build Unapproved Report → Format Bale Messages (report is still sent, flagged as unverified)
```

Key design points:

- **Two-agent check:** the researcher produces a structured JSON report with source IDs (S1, S2, …); the reviewer independently verifies each source and each claim.
- **Deterministic gate:** the `Review Decision` code node does not trust the reviewer blindly. It re-checks that every source is `verified`, every claim is `supported`, and every cited source ID exists.
- **One revision round:** if the report is rejected, the researcher gets the reviewer feedback once. If it is still rejected, the report is delivered with the unapproved items clearly marked.
- **Long messages** are split into 3500-character chunks for Bale.

## Requirements

- n8n with **Data Tables** support and the **AI / LangChain nodes** (built and tested on a recent n8n 1.x/2.x release with node versions as found in the JSON files)
- A **Bale bot token** (from Bale's BotFather)
- An **OpenAI API key** (models used: `gpt-4.1-mini` for all three LLM nodes, OpenAI built-in web search tool, and OpenAI audio transcription)
- A public HTTPS URL for your n8n instance (webhook)

## Setup

1. **Create three n8n Data Tables** (all columns are `string`):

   | Table | Columns |
   |---|---|
   | `sessions` | `chat_id`, `state`, `research_id`, `brief`, `topic_title`, `updated_at` |
   | `researches` | `research_id`, `chat_id`, `datetime`, `topic_title`, `status` |
   | `evidence` | `research_id`, `kind`, `title`, `author_year`, `source_type`, `tier`, `evidence_strength`, `note`, `link` |

2. **Insert your Bale token** (do this on a local copy, never commit it). The placeholder `YOUR_BALE_BOT_TOKEN` appears in 8 HTTP Request nodes. Replace it with `<bot_id>:<secret>`:

   ```bash
   sed -i 's/YOUR_BALE_BOT_TOKEN/123456789:YOUR_REAL_SECRET/g' workflows/*.json
   ```

   Or edit the URL in each node after import.

3. **Import `02-research-engine.json` first**, then `01-bale-intake-and-confirmation.json` (File → Import from file).

4. **Connect credentials** (credentials are intentionally not exported). Select your OpenAI credential in:
   - Workflow 01: `Brief Extraction LLM`, `Transcribe Voice`
   - Workflow 02: `Researcher LLM (Web Search)`, `Reviewer LLM (Web Search)`

5. **Re-select the Data Tables** (IDs were replaced with `YOUR_*_TABLE_ID`):
   - Workflow 01 → `sessions` in: `Get Session (Confirm)`, `Reset Session`, `Get Session`, `Save Session (Awaiting Confirmation)`
   - Workflow 02 → `evidence` in `Save Evidence`, `researches` in `Save Research`, `sessions` in `Update Session`

6. **Link the sub-workflow:** in workflow 01, open `Run Research Engine` and select workflow 02 (placeholder: `YOUR_RESEARCH_ENGINE_WORKFLOW_ID`).

7. **Replace the course outline:** the node `Prepare Research Context` (workflow 02) contains a hard-coded `course_outline` string that the researcher maps findings to. Replace it with your own outline, or adapt the prompts if you don't need course mapping.

8. **Activate both workflows** and register the production webhook URL with Bale (`setWebhook`):

   ```
   https://<your-n8n-host>/webhook/bale-research-bot
   ```

## Configuration reference

| Placeholder | Where | What to put |
|---|---|---|
| `YOUR_BALE_BOT_TOKEN` | 8 HTTP Request nodes | `<bot_id>:<secret>` from Bale |
| `YOUR_RESEARCH_ENGINE_WORKFLOW_ID` | `Run Research Engine` | Select workflow 02 in the UI |
| `YOUR_SESSIONS_TABLE_ID` / `YOUR_EVIDENCE_TABLE_ID` / `YOUR_RESEARCHES_TABLE_ID` | Data Table nodes | Select your tables in the UI |
| Webhook path `bale-research-bot` | `Bale Webhook` | Change if you want a less guessable path |

## Bot commands

| Input | Behaviour |
|---|---|
| `/start` | Sends a welcome message |
| `/new` | Clears the current session so you can start a new topic |
| Text or voice | Builds a brief and asks for confirmation |
| Inline button "confirm" | Starts the research |

## Security notes

- **Never commit** a real bot token or API key. Credentials are not exported by n8n; the token lives inside HTTP Request URLs, so inject it locally (step 2) or move it to an n8n credential.
- If a token was ever pushed to a repository or shared in plain text, **revoke and regenerate** it in BotFather. Deleting the file later does not remove it from git history.
- The webhook has **no sender verification**. Anyone who knows the URL can post fake updates. Use a hard-to-guess webhook path and consider checking the sender's `chat_id` against an allow-list.
- Every request triggers paid LLM + web-search calls (researcher + reviewer, possibly twice). Consider an allow-list or rate limit before exposing the bot publicly.

## Known issues

- `Reset Session` (filters and column `chat_id`) and `Prepare Voice Input` (`chat_id`) use `=={{ … }}` instead of `={{ … }}`. n8n evaluates this as the string `"=" + value`, so the stored `chat_id` becomes `=12345`. Effect: `/new` may not reset the real session, and sessions created from voice messages may not be found when the user presses "confirm". Fix: remove one `=`.
- The reviewer is an LLM with web search. Its verification is a strong filter, not a guarantee. Treat reports marked unverified as drafts.

## License

No license has been chosen yet. Add a `LICENSE` file before publishing if you want others to reuse this.

---

<div dir="rtl">

# 🇮🇷 نسخه فارسی

# دستیار پژوهش بله (n8n)

یک دستیار پژوهشی که به‌صورت **ربات پیام‌رسان بله** روی **n8n** اجرا می‌شود. کاربر یک سؤال پژوهشی (متن یا ویس) می‌فرستد، ربات آن را به‌صورت یک Brief ساختاریافته بازنویسی می‌کند و تأیید می‌گیرد، سپس یک ایجنت پژوهشگر در وب جست‌وجو می‌کند و گزارشی مستند به منابع می‌نویسد. پیش از ارسال گزارش، یک ایجنت بازبین مستقل تک‌تک منابع و ادعاها را بررسی می‌کند.

پرامپت‌ها و پیام‌های ربات **فارسی** هستند؛ نام نودها، ساختار کد و این مستندات انگلیسی‌اند.

## معماری

دو ورک‌فلو در کنار هم کار می‌کنند:

| فایل | ورک‌فلو | نقش |
|---|---|---|
| `workflows/01-bale-intake-and-confirmation.json` | ورودی و تأیید بله | دریافت پیام‌های بله، مدیریت دستورها، تبدیل ویس به متن، استخراج Brief، گرفتن تأیید کاربر و راه‌اندازی موتور تحقیق |
| `workflows/02-research-engine.json` | موتور تحقیق | اجرای ایجنت پژوهشگر، بازبینی با ایجنت دوم، اعمال Evidence Gate، ارسال گزارش به بله و ذخیره نتایج |

### ورک‌فلو ۰۱ – ورودی و تأیید

- پیام از `Bale Webhook` وارد `Route Input` می‌شود و پنج مسیر دارد: دکمه «تأیید»، دستور `/start`، دستور `/new`، پیام صوتی، پیام متنی.
- ویس: دریافت مسیر فایل ← دانلود ← تبدیل به متن (فارسی) ← ورودی نرمال‌شده.
- متن: ورودی نرمال‌شده ← دریافت Session ← استخراج Brief با مدل زبان و خروجی ساختاریافته ← ذخیره Session با وضعیت «در انتظار تأیید» ← ارسال پیام تأیید همراه دکمه.
- دکمه تأیید: پاسخ به callback ← دریافت Session ← اطلاع شروع تحقیق ← اجرای ورک‌فلو ۰۲ (بدون انتظار برای پایان آن).

### ورک‌فلو ۰۲ – موتور تحقیق

- `Main Researcher` (مدل زبان + جست‌وجوی وب) گزارش JSON ساختاریافته با شناسه منابع (S1، S2، …) تولید می‌کند.
- `Independent Source Reviewer` به‌صورت مستقل منابع و ادعاها را بررسی می‌کند.
- نود `Review Decision` به بازبین کورکورانه اعتماد نمی‌کند: دوباره بررسی می‌کند که همه منابع `verified`، همه ادعاها `supported` و همه شناسه‌های ارجاع موجود باشند.
- مسیرها: **approved** ← Evidence Gate، ارسال به بله و ذخیره در جدول‌ها. **revise** ← بازگشت به پژوهشگر (حداکثر **یک** دور اصلاح). **failed** ← گزارش همچنان ارسال می‌شود، اما موارد تأییدنشده مشخص شده‌اند.
- پیام‌های بلند در قطعه‌های ۳۵۰۰ کاراکتری ارسال می‌شوند.

## پیش‌نیازها

- n8n با پشتیبانی از **Data Table** و **نودهای AI / LangChain**
- **توکن ربات بله** (از BotFather بله)
- **کلید API اوپن‌ای‌آی** (مدل `gpt-4.1-mini` در هر سه نود LLM، ابزار جست‌وجوی وب داخلی OpenAI و تبدیل صوت به متن)
- آدرس HTTPS عمومی برای n8n (وب‌هوک)

## راه‌اندازی

۱. **سه Data Table در n8n بساز** (همه ستون‌ها از نوع `string`):

| جدول | ستون‌ها |
|---|---|
| `sessions` | `chat_id`, `state`, `research_id`, `brief`, `topic_title`, `updated_at` |
| `researches` | `research_id`, `chat_id`, `datetime`, `topic_title`, `status` |
| `evidence` | `research_id`, `kind`, `title`, `author_year`, `source_type`, `tier`, `evidence_strength`, `note`, `link` |

۲. **توکن بله را وارد کن** (روی نسخه محلی؛ هرگز کامیت نکن). عبارت `YOUR_BALE_BOT_TOKEN` در ۸ نود HTTP Request وجود دارد و باید با `<bot_id>:<secret>` جایگزین شود:

```bash
sed -i 's/YOUR_BALE_BOT_TOKEN/123456789:YOUR_REAL_SECRET/g' workflows/*.json
```

یا بعد از Import، آدرس هر نود را دستی ویرایش کن.

۳. **ابتدا `02-research-engine.json` و بعد `01-bale-intake-and-confirmation.json` را Import کن.**

۴. **Credentialها را وصل کن** (عمداً Export نشده‌اند). Credential اوپن‌ای‌آی خودت را در این نودها انتخاب کن:
   - ورک‌فلو ۰۱: `Brief Extraction LLM` و `Transcribe Voice`
   - ورک‌فلو ۰۲: `Researcher LLM (Web Search)` و `Reviewer LLM (Web Search)`

۵. **Data Tableها را دوباره انتخاب کن** (شناسه‌ها با `YOUR_*_TABLE_ID` جایگزین شده‌اند):
   - ورک‌فلو ۰۱ ← جدول `sessions` در نودهای `Get Session (Confirm)`، `Reset Session`، `Get Session` و `Save Session (Awaiting Confirmation)`
   - ورک‌فلو ۰۲ ← جدول `evidence` در `Save Evidence`، جدول `researches` در `Save Research` و جدول `sessions` در `Update Session`

۶. **زیرورک‌فلو را وصل کن:** در ورک‌فلو ۰۱ نود `Run Research Engine` را باز کن و ورک‌فلو ۰۲ را انتخاب کن.

۷. **سرفصل‌های دوره را جایگزین کن:** نود `Prepare Research Context` (ورک‌فلو ۰۲) یک رشته ثابت `course_outline` دارد که پژوهشگر یافته‌ها را به آن نگاشت می‌کند. آن را با سرفصل‌های خودت عوض کن، یا اگر نگاشت به دوره لازم نداری پرامپت‌ها را تغییر بده.

۸. **هر دو ورک‌فلو را فعال کن** و آدرس Production وب‌هوک را در بله ثبت کن (`setWebhook`):

```
https://<your-n8n-host>/webhook/bale-research-bot
```

## جدول جایگزین‌ها

| Placeholder | کجا | چه چیزی بگذاری |
|---|---|---|
| `YOUR_BALE_BOT_TOKEN` | ۸ نود HTTP Request | `<bot_id>:<secret>` از بله |
| `YOUR_RESEARCH_ENGINE_WORKFLOW_ID` | `Run Research Engine` | ورک‌فلو ۰۲ را از UI انتخاب کن |
| `YOUR_SESSIONS_TABLE_ID` / `YOUR_EVIDENCE_TABLE_ID` / `YOUR_RESEARCHES_TABLE_ID` | نودهای Data Table | جدول‌ها را از UI انتخاب کن |
| مسیر وب‌هوک `bale-research-bot` | `Bale Webhook` | برای امنیت بیشتر یک مسیر غیرقابل‌حدس بگذار |

## دستورهای ربات

| ورودی | رفتار |
|---|---|
| `/start` | ارسال پیام خوش‌آمد |
| `/new` | پاک‌کردن Session فعلی برای شروع موضوع جدید |
| متن یا ویس | ساخت Brief و درخواست تأیید |
| دکمه «تأیید» | شروع تحقیق |

## نکات امنیتی

- **هرگز** توکن یا کلید API واقعی را کامیت نکن. n8n خودش Credentialها را Export نمی‌کند؛ اما توکن بله داخل URL نودها بود، پس آن را محلی تزریق کن (مرحله ۲) یا به یک Credential در n8n منتقل کن.
- اگر توکنی روزی در مخزن یا متن ساده منتشر شد، آن را در BotFather **باطل و دوباره صادر کن**. حذف فایل بعدی، آن را از تاریخچه git پاک نمی‌کند.
- وب‌هوک **احراز هویت فرستنده ندارد**. هر کسی که آدرس را بداند می‌تواند پیام جعلی بفرستد. از مسیر غیرقابل‌حدس استفاده کن و بررسی `chat_id` با فهرست مجاز را در نظر بگیر.
- هر درخواست هزینه LLM و جست‌وجوی وب دارد (پژوهشگر + بازبین، و گاهی دو بار). پیش از عمومی‌کردن ربات، فهرست مجاز یا محدودیت نرخ اضافه کن.

## مشکلات شناخته‌شده

- نودهای `Reset Session` (فیلتر و ستون `chat_id`) و `Prepare Voice Input` (فیلد `chat_id`) به‌جای `={{ … }}` از `=={{ … }}` استفاده می‌کنند. n8n این را به‌صورت رشته `"=" + مقدار` ارزیابی می‌کند و `chat_id` ذخیره‌شده مثلاً `=12345` می‌شود. نتیجه: ممکن است `/new` Session واقعی را ریست نکند و Session ساخته‌شده از پیام صوتی هنگام زدن «تأیید» پیدا نشود. راه‌حل: یک علامت `=` را حذف کن.
- بازبین یک مدل زبان با جست‌وجوی وب است؛ فیلتر قوی است اما تضمین نیست. گزارش‌های علامت‌خورده به‌عنوان «تأییدنشده» را پیش‌نویس در نظر بگیر.

## مجوز

هنوز مجوزی انتخاب نشده است. اگر می‌خواهی دیگران بتوانند از پروژه استفاده کنند، پیش از انتشار فایل `LICENSE` اضافه کن.

</div>
