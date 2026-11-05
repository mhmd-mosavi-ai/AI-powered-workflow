# Content Machine

An n8n workflow that automates the creation of SEO-focused articles with AI, generates supporting images, and saves the finished article as a draft in WordPress.

## Features

1. Retrieves an unpublished content request from Google Sheets.
2. Creates an SEO strategy, including keywords, search intent, audience pain points, and an article outline.
3. Writes the article in WordPress-ready HTML.
4. Generates four English image prompts: one featured image and three in-article images.
5. Generates the four images using OpenAI image generation.
6. Uploads the images to the WordPress media library.
7. Inserts the images into the article and creates a WordPress post with `draft` status.
8. Updates the corresponding Google Sheets row to mark it as processed.

## Workflow File

Import this file into n8n:

`workflow/ماشین-محتوا.json`

## Requirements

- An n8n instance with the node types used in this workflow available.
- A Google account with access to Google Sheets.
- An OpenAI account and API access to the models selected in the workflow.
- A WordPress website with REST API access and credentials that can upload media and create posts.
- A Google spreadsheet with the columns listed below.

## Google Sheets Structure

The workflow expects the following columns:

| Column | Purpose |
|---|---|
| `id` | Row identifier used when updating the row. |
| `is published` | Selects unprocessed rows. Expected value: `No`; changed to `Yes` after processing. |
| `موضوع` | Article topic. |
| `مخاطب هدف مقاله` | Target audience. |
| `لحن` | Desired writing tone. |
| `طول محتوا` | Requested article length. |
| `کال تو اکشن` | Call to action to include in the article. |
| `زبان انتشار` | Article language. |
| `رنگ سازمانی` | Brand color to include in the image prompts. |

The spreadsheet ID and sheet selection must be configured in the Google Sheets nodes in n8n.

## Setup

1. In n8n, open **Workflows** and select **Import from File**.
2. Import `workflow/ماشین-محتوا.json`.
3. Open both Google Sheets nodes, select your Google Sheets credentials, and configure your spreadsheet and sheet.
4. Open the OpenAI and language-model nodes and select your own OpenAI credentials.
5. In the WordPress HTTP Request nodes, replace `https://YOUR_WORDPRESS_DOMAIN` with your WordPress site's base URL and select your WordPress credentials.
6. Run a test using a sample spreadsheet row. Check the generated article, images, and WordPress draft before activating the workflow.
7. Activate the workflow only after the test succeeds.

## Important Notes

- The Schedule Trigger is currently configured to run every minute.
- Each run looks for the first row where `is published` is set to `No`.
- Posts are created with `draft` status. The workflow does not publish articles publicly.
- After a post is created, the corresponding spreadsheet row is updated to `Yes`.
- Node types, model selections, and node versions are based on the exported workflow. Depending on your n8n version, some settings may need adjustment.

## Security

This repository version has been prepared for sharing by removing n8n credential references and replacing the spreadsheet ID and WordPress domain with placeholders.

- Never commit API keys, passwords, access tokens, customer data, or an unredacted n8n export.
- Store and manage credentials inside n8n's **Credentials** section.
- If a real secret has ever been committed to a public repository, revoke or rotate it immediately. Deleting the file does not remove the secret from Git history.




# ماشین محتوا

ورکفلو n8n برای ساخت محتوای مقاله با کمک هوش مصنوعی، از مرحله تدوین استراتژی سئو تا تولید تصویر و ایجاد پیش‌نویس در وردپرس.

## قابلیت‌ها

1. دریافت ردیف منتشرنشده از Google Sheets
2. تولید استراتژی سئو شامل کلمه کلیدی، هدف جستجو، دغدغه‌های مخاطب و ساختار هدینگ‌ها
3. تولید متن مقاله با HTML مناسب وردپرس
4. تولید چهار پرامپت انگلیسی برای تصویر شاخص و سه تصویر داخل مقاله
5. تولید چهار تصویر با مدل تصویر OpenAI
6. بارگذاری تصاویر در کتابخانه رسانه وردپرس
7. قراردادن تصاویر در متن و ایجاد نوشته با وضعیت `draft`
8. تغییر وضعیت ردیف Google Sheets به `Yes` پس از ایجاد نوشته

## فایل اصلی

فایل قابل ورود به n8n در مسیر زیر قرار دارد:

`workflow/ماشین-محتوا.json`

## پیش‌نیازها

- یک نمونه n8n با دسترسی به نودهای استفاده‌شده در ورکفلو
- حساب و دسترسی Google Sheets
- اعتبارنامه OpenAI و دسترسی به مدل‌های انتخاب‌شده در نودها
- سایت WordPress و دسترسی REST API برای ساخت نوشته و بارگذاری رسانه
- یک Google Sheet با ستون‌های مورد انتظار

## ساختار مورد انتظار Google Sheets

ورکفلو در فایل فعلی از این ستون‌ها استفاده می‌کند:

| نام ستون | کاربرد |
|---|---|
| `id` | شناسه ردیف برای به‌روزرسانی |
| `is published` | فیلتر ردیف‌های منتشرنشده؛ مقدار مورد انتظار `No` و پس از اجرا `Yes` است |
| `موضوع` | موضوع مقاله |
| `مخاطب هدف مقاله` | مخاطب هدف |
| `لحن` | لحن محتوا |
| `طول محتوا` | طول درخواستی مقاله |
| `کال تو اکشن` | دعوت به اقدام |
| `زبان انتشار` | زبان مقاله |
| `رنگ سازمانی` | رنگ مورد استفاده در پرامپت تصاویر |

نام شیت و شناسه فایل Google Sheets باید در نودهای مربوطه با فایل خودتان تنظیم شوند.

## راه‌اندازی در n8n

1. در n8n از بخش Workflows گزینه Import from File را انتخاب کنید.
2. فایل `workflow/ماشین-محتوا.json` را وارد کنید.
3. نودهای Google Sheets را باز کنید، اعتبارنامه Google خودتان را انتخاب کنید و فایل و شیت خودتان را دوباره انتخاب کنید.
4. در نودهای OpenAI و مدل‌های زبانی، اعتبارنامه OpenAI خودتان را انتخاب کنید.
5. در نودهای HTTP Request مربوط به وردپرس، آدرس سایت خودتان را جایگزین `https://YOUR_WORDPRESS_DOMAIN` کنید و اعتبارنامه WordPress را انتخاب کنید.
6. ورکفلو را با یک ردیف آزمایشی بررسی کنید. تا قبل از اطمینان از نتیجه، آن را فعال نکنید.
7. پس از تست موفق، ورکفلو را فعال کنید.

## نکات مهم قبل از اجرا

- زمان‌بند فعلی هر یک دقیقه اجرا می‌شود.
- ورکفلو اولین ردیفی را که مقدار `is published` آن `No` باشد انتخاب می‌کند.
- نوشته وردپرس با وضعیت پیش‌نویس (`draft`) ساخته می‌شود؛ این ورکفلو مستقیماً مقاله را عمومی منتشر نمی‌کند.
- پس از ایجاد نوشته، ردیف مربوطه در شیت به `Yes` تغییر می‌کند.
- مدل‌ها و نسخه نودها مطابق فایل اصلی حفظ شده‌اند؛ در صورت تفاوت نسخه n8n یا نودها ممکن است لازم باشد تنظیمات را تطبیق دهید.
- مقادیر نمونه‌ای مانند `YOUR_GOOGLE_SHEET_ID` و `YOUR_WORDPRESS_DOMAIN` باید در n8n با اطلاعات واقعی خودتان جایگزین شوند.

## امنیت

این نسخه برای انتشار عمومی آماده‌سازی شده است: بلوک‌های credentials و شناسه‌های داخلی نمونه حذف شده‌اند و شناسه Google Sheet و دامنه وردپرس با مقدار جایگزین شده‌اند.

**هیچ‌وقت API Key، رمز عبور، توکن، فایل خروجی حاوی اطلاعات مشتریان یا Export خام n8n را در مخزن عمومی قرار ندهید.** اعتبارنامه‌ها را فقط از بخش Credentials داخل n8n متصل کنید.

اگر قبلاً کلید یا رمز واقعی را در یک مخزن عمومی قرار داده‌اید، صرفاً حذف فایل کافی نیست؛ آن کلید را از سرویس مربوطه باطل و دوباره صادر کنید، چون تاریخچه Git ممکن است نسخه قبلی را نگه دارد.
