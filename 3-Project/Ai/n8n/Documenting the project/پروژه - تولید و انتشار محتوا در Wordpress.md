# ۱ تعریف پروژه

خلاصه و مفهوم: کلیت کار به این صورت است که با فرم ورودی «Article Request Form» موضوع و نوع محتوای موردنظر ثبت می‌شود، سپس با مدل‌های زبانی Groq یک پیش‌نویس ساختاریافته شامل عنوان، نامک (slug)، توضیح متا، متن مقاله و پرامپت تصویر تولید می‌شود، متن به HTML تبدیل و برای سئو بهینه می‌شود، با سرویس Pollinations.ai یک تصویر ۱۲۰۰×۶۳۰ ساخته می‌شود، فایل تصویر در کتابخانه رسانه وردپرس روی https://sysart.ir آپلود می‌گردد و در پایان یک پست با وضعیت draft به همراه تصویر شاخص (Featured Image) در وردپرس ثبت می‌شود. خروجی نهایی، پیش‌نویس کامل یک مقاله سئو شده در پنل مدیریت وردپرس است.

این جریان به‌صورت یکپارچه در n8n Cloud پیاده‌سازی شده و تمام مدل‌های مورد استفاده از نسخه‌های رایگان و پرسرعت هستند.

# ۲ نقشه پروژه

## ۲.۱ تعریف نحوه کار هر نود

**نود** **Article Request Form:** نقطه شروع پروژه است و یک فرم وب در اختیار مدیر سایت یا نویسنده قرار می‌دهد تا موضوع، نوع محتوا (مقاله، محصول، دسته‌بندی یا شبکه اجتماعی) و جزئیات موردنظر ثبت شود. داده‌های فرم به نود بعدی در قالب JSON منتقل می‌شود.

**نود** **Prepare Form Input:** فیلدهای خام دریافتی از فرم را پاک‌سازی، اعتبارسنجی و در قالبی یکدست (کلیدهای استاندارد موضوع، زبان و نوع محتوا) سازمان‌دهی می‌کند تا نودهای بعدی با ورودی قابل اتکا کار کنند.

**نود** **Switch1:** دروازه تصمیم‌گیری جریان است. بر اساس مقدار فیلد «نوع محتوا» ورودی، چهار Route مجزا تعریف شده است: «مقاله»، «محصول»، «دسته‌بندی» و «شبکه‌های اجتماعی». در سناریوی فعال فعلی، Route «مقاله» به نود تولید پیش‌نویس متصل است و بقیه شاخه‌ها به‌عنوان مسیرهای توسعه‌یافته و غیرفعال باقی مانده‌اند.

**نود** **Generate Article Draft (Basic LLM Chain):** با زیرنودهای Groq Chat Model2 و Structured Output Parser2، موضوع مقاله را پردازش می‌کند و خروجی را به‌صورت JSON ساختاریافته با فیلدهای title، slug، meta_description، content و image_prompt تحویل می‌دهد. کار اصلی این نود، جدا کردن داده‌های مقاله از متن آزاد است تا هر فیلد بعداً جداگانه قابل ارجاع باشد.

**نود** **Markdown:** متن خام خروجی مرحله قبل را به HTML استاندارد تبدیل می‌کند تا هدینگ‌ها، لیست‌ها، پاراگراف‌ها و لینک‌ها در ویرایشگر وردپرس درست نمایش داده شوند. خروجی این نود، محتوای نهایی مقاله است.

**نود** **Prepare Article Fields:** فیلدهای مقاله (عنوان، نامک، توضیح متا و محتوای HTML) را در یک شیء واحد جمع‌آوری می‌کند و آن را به مرحله بهینه‌سازی سئو و سپس به ساخت پست تحویل می‌دهد. در کنار این نود، یک نود معلق با نام Upload to WordPress Media2 در پایین ورک‌فلو رها شده و در مسیر اجرایی فعال نیست.

**نود** **Optimize Article for SEO (Basic LLM Chain):** با زیرنودهای Groq Chat Model3 و Structured Output Parser3، متن را از نظر چینش کلمات کلیدی، ساختار هدینگ‌ها، طول پاراگراف و لحن مناسب جست‌وجو پالایش می‌کند. توجه: در خروجی ثبت‌شده از این نود، فیلد اصلی در دسترس output.prompt است و عنوان، متن و نامک نهایی از خروجی Generate Article Draft خوانده می‌شود.

**نود** **Generate Image (HTTP Request):** پرامپت تصویر (image_prompt) را به قالب URL سرویس Pollinations.ai تبدیل می‌کند و با پارامترهای nologo=true و ابعاد 1200×630 درخواست می‌فرستد. پاسخ به‌صورت فایل باینری دریافت و مستقیماً به نود بعدی منتقل می‌شود.

**نود** **Upload to WordPress Media (HTTP Request):** فایل باینری تصویر را با متد POST به مسیر https://sysart.ir/wp-json/wp/v2/media و با هدر Content-Disposition و احراز هویت Basic Auth (Application Password وردپرس) آپلود می‌کند. خروجی این نود شامل شناسه رسانه (id: 16) و آدرس مستقیم تصویر در وردپرس است.

**نود** **Create WordPress Draft1 (HTTP Request):** با متد POST به مسیر https://sysart.ir/wp-json/wp/v2/posts، بدنه‌ای با Content-Type: application/json ارسال می‌کند که شامل عنوان، محتوای HTML، نامک، وضعیت draft و فیلد featured_media برابر با شناسه تصویر آپلودشده است. نتیجه، ساخت پیش‌نویس در وردپرس همراه با تصویر شاخص است.

# ۳ خطاها و چالش‌ها

**در نود** **Create WordPress Draft1** به علت اینکه نود رسمی WordPress در n8n در فهرست فیلدهای بخش Add Field گزینه‌ای با عنوان Featured Media نداشت و تنها فیلدهایی مانند Author Name or ID، Password، Date، Comment Status، Ping Status، Format، Sticky، Category Names or IDs، Tag Names or IDs و Template را ارائه می‌کرد، چالش داشتیم؛ چون امکان تعیین مستقیم تصویر شاخص وجود نداشت. برای همین ساخت پست را از طریق HTTP Request و فراخوانی مستقیم REST API وردپرس در مسیر /wp-json/wp/v2/posts انجام دادیم و فیلد featured_media را در بدنه درخواست گنجاندیم.

**در نود** **Create WordPress Draft1** به علت اینکه در حالت Body Content Type روی مقدار JSON، متن پیش‌فرض کادر بدنه ارسال می‌شد و اکسپریشن نادیده گرفته می‌شد، با خطای ۴۰۰ وردپرس با پیام Content, title, and excerpt are empty مواجه شدیم. برای همین Body Content Type را روی حالت Raw و Content Type را روی application/json تنظیم کردیم و کل بدنه را با JSON.stringify به شکل یک رشته JSON خالص ساختیم:

{{ JSON.stringify({  
  title: $('Generate Article Draft').item.json.output.title,  
  content: $('Generate Article Draft').item.json.output.content,  
  slug: $('Generate Article Draft').item.json.output.slug,  
  status: "draft",  
  featured_media: Number($json.id)  
}) }}

پس از این اصلاح، نود با وضعیت سبز Node executed successfully اجرا شد و پست با شناسه 18، وضعیت draft و نامک exercise-bodybuilding-kung-fu در وردپرس ساخته شد.

**در نود** **Upload to WordPress Media** به علت ارجاع اشتباه به خودِ نود جاری (که با نام Upload to WordPress Media1 در آن اجرا ثبت شده بود) و خطای No path back to referenced node، چالش داشتیم؛ چون شناسه رسانه دریافت نمی‌شد. برای همین ارجاع را به نود واقعی آپلود قبلی (Upload to WordPress Media) اصلاح کردیم و پس از آن شناسه 16 و آدرس https://sysart.ir/wp-content/uploads/2026/10/featured-image.jpg به‌درستی در خروجی ثبت شد.

**در نود** **Generate Article Draft** به علت ابهام در ساختار خروجی نود Optimize Article for SEO که تنها output.prompt را برمی‌گرداند، چالش داشتیم؛ چون عنوان، متن و نامک از این مسیر قابل خواندن نبود. برای همین منبع این فیلدها را به خروجی ساختاریافته Generate Article Draft تغییر دادیم و همه ارجاع‌های بعدی را به همان نود متصل کردیم.

**در نود** **Upload to WordPress Media** به علت نداشتن ستون Thumbnail در جدول نوشته‌های وردپرس، چالش داشتیم؛ چون به نظر می‌رسید تصویر شاخص ثبت نشده است. برای همین صفحه ویرایش نوشته را باز کردیم و در سایدبار Post بخش Featured image بررسی شد؛ مشخص شد تصویر به‌درستی به پست متصل شده است و مشکل فقط نبودِ ستون پیش‌نمایش در فهرست نوشته‌ها و رفتار قالب در نمای پیش‌نمایش بود.

**نکته باقی‌مانده:** در برخی قالب‌های وردپرس تصویر شاخص به‌صورت پیش‌فرض در بالای متن نمایش داده نمی‌شود. برای اطمینان از نمایش تصویر در همه قالب‌ها، می‌توان تگ تصویر را به ابتدای محتوای مقاله اضافه کرد:

content: `<img src="${$('Upload to WordPress Media').item.json.source_url}" style="width:100%; border-radius:8px; margin-bottom:20px;" />` + $('Generate Article Draft').item.json.output.content