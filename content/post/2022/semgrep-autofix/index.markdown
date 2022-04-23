---
title: "قابلیت ترمیم خودکار Semgrep"
date: 2022-04-23T11:40:33-07:00
draft: false
ShowToc: true
---

در این نوشتار می‌خواهم در مورد قابلیت ترمیمِ خودکار ([autofix][autofix]) کُد Semgrep صحبت کنم. این قابلیت آزمایشی به ما امکان می‌دهد که بعد از یافتن نتایج در فایلهای منبع، آنها را به صورت اتوماتیک دستکاری کنیم.

[autofix]: https://semgrep.dev/docs/experiments/overview/#autofix

اگر علاقه به خواندن نسخه انگلیسی دارید:
* https://parsiya.net/blog/2021-10-25-a-hands-on-intro-to-semgreps-autofix/

همه نمونه‌ها در Semgrep playground هستند ولی می‌توانید آنها را آفلاین نیز اجرا کنید:

* https://github.com/parsiya/Parsia-Code/tree/master/semgrep-autofix

# پیش‌نیازهای این بلاگ
برای استفاده بهینه از این بلاگ، بهتر است این موارد را بدانید:

* استفاده از Semgrep و نوشتن rule (قانون؟).  [چرا از Semgrep برای Static Analysis استفاده کنیم؟]({{< relref "/post/2021/semgrep-1/index.markdown" >}}) را بخوانید.
* آشنایی سَرسَری با خواندن کدُ Java، Python و Go.
* آشنایی مختصر با بعضی مفاهیم امنیت مانند HttpOnly و XSS.



نکته: ترمیمِ خودکار، یکی از [قابلیت‌های آزمایشی Semgrep][semgrep-exp] است. در زمان نگارش (آوریل 2022 برای نسخه فارسی و اکتبر 2021 برای نسخه انگلیسی) همه مثال‌ها درست هستند. اگر از آینده میایید، شاید بعضی چیزها متفاوت باشند.

[semgrep-exp]: https://semgrep.dev/docs/experiments/overview/

# اجرای قوانین
برای اجرا و تست قوانین دو راه داریم:

 1. استفاده از Semgrep playground در آدرس https://semgrep.dev/playground. من بیشتر از این روش استفاده می‌کنم چون راحت و سریع می‏‌توانم قانون را عوض کنم و نتیجه را ببینم.
2. اجرا در خطِ فرمان (ترجمه فارسی command line).

بعد از نوشتن یک قانون می‌توانید درستی ساختار فایل yaml آن را تست کنید:

```
semgrep-crule1.yaml--validate 
```

اگر یک قانون دارای بخش autofix باشد، هنگام اجرا فایل اصلی دستکاری نمی‌شود ولی می‌توانید تغییرات را ببینید. برای تغییر فایل اصلی از سویچ `autofix--` استفاده کنید. برای مشاهده تغییرات بدون تغییر فایل اصلی از سویچ `dry-run--` استفاده کنید (حضور همزمان autofix و dry-run معادل نبودِ autofix است یعنی تغییرات فقط نشان داده می‌شوند):

```
semgrep -c rule1.yaml example.java --autofix --dryrun 
```

{{< figure src="00-autofix-dryrun.png" title="استفاده از سویچ‌های مختلف" >}}

# روش‌های استفاده از autofix

دو روش مختلف برای استفاده از autofix داریم:

1. fix
2. fix-regex

## فرمان Fix
هر چیزی که قانون پیدا کرده است را با مقدار این فرمان جایگزین میکنیم. بهترین کاندیدها برای این قابلیت، توابع یا رشته (string) های ناامنی هستند که باید با معادل امن جایگزین شوند. چند مثال با این دستور ببینیم.

### Python - sys.exit
این مثال را از آموزش Semgrep در آدرس (https://semgrep.dev/s/R6g) قرض گرفته‌ام. در این قانون ما به دنبال تابع exit هستیم و می‌خواهیم آن را با sys.exit جایگزین کنیم. Semgrep تفاوت تابع exit با دیگر استفاده‌های این کلمه را در کُد می‌فهمد و می‎‌تواند راحت آن را جایگزین کنید. بعد از اجرای قانون در وب‌سایت بالا می بینید که دکمه Apply Fix فعال شده و می‌توانید به صورت خودکار کد برنامه را دستکاری کنید.

{{< figure src="01-sys-exit.png" title="تغییر اتوماتیک exit به sys.exit" >}}

یکی از نکات جالب این است که می‌توانیم پارامتر تابع را توسط یک metavariable (در اینجا با نام X) در قانون ذخیره کنیم و در autofix استفاده کنیم. دیگر لازم نیست حتی نگران مقدار پارامتر باشیم.

### Java - CBC Padding Oracle
برای این مثال، قانون cbc-padding-oracle زبان جاوا را از رجیستری Semgrep دستکاری کرده‌ام تا راحت‌تر خوانده شود.

``` yaml
# java-cbc-padding-oracle/cbc-padding-oracle.yaml
rules:
  - id: cbc-padding-oracle
    severity: WARNING
    message: Match found
    languages:
      - java
    pattern: $CIPHER.getInstance("=~/.*\/CBC\/PKCS5Padding/")
    fix: $CIPHER.getInstance("AES/GCM/NoPadding")
```

این قانون به دنبال هر چیزی که شبیه `object.getInstance("string")` باشد می‌گردد و در صورتی که رشته داخل شامل `CBC/PKCS5Padding` باشد، نتیجه را گزارش می‌دهد. این قانون از قابلیت قدیمی [string matching][string-matching] استفاده می‌کند که دیگر پشتیبانی نمی‌شود. برای دست‌گرمی باید آن را با [metavariable-regex][meta-regex] جایگزین کنیم. این کار ساده است، اول یک metavariable به عنوان پارامتر تعریف می‌کنیم (در اینجا `INS$`) و بعد regex را روی آن اجرا می‌کنیم و نتیجه همان است:

* https://semgrep.dev/s/parsiya:java-cbc-padding-oracle-metavariable-regex

[string-matching]: https://semgrep.dev/docs/writing-rules/pattern-syntax/#string-matching
[meta-regex]: https://semgrep.dev/docs/writing-rules/rule-syntax/#metavariable-regex

``` yaml
# java-cbc-padding-oracle/cbc-padding-oracle-metavariable-regex.yaml
rules:
  - id: cbc-padding-oracle-metavariable-regex
    message: Match found
    languages:
      - java
    severity: WARNING
    patterns:
      - pattern: $CIPHER.getInstance($INS)
      - metavariable-regex:
          metavariable: $INS
          regex: .*\/CBC\/PKCS5Padding
    fix: $CIPHER.getInstance("AES/GCM/NoPadding")
```

مقدار "محاسبه" قانون جدید در playground عدد بزرگتری بود و من خواستم آن را آزمایش کنم. قانون جدید خیلی پیچیده‌تر از قانون قدیمی نیست.

``` yaml
$ multitime -q -n 50 ./cbc-padding-oracle.sh
===> multitime results
1: -q ./cbc-padding-oracle.sh
            Mean        Std.Dev.    Min         Median      Max
real        0.781       0.006       0.773       0.780       0.806
user        0.501       0.041       0.406       0.500       0.609
sys         0.256       0.044       0.172       0.258       0.359
$ multitime -q -n 50 ./cbc-padding-oracle-metavariable-regex.sh
===> multitime results
1: -q ./cbc-padding-oracle-metavariable-regex.sh
            Mean        Std.Dev.    Min         Median      Max
real        0.788       0.007       0.778       0.786       0.813
user        0.516       0.047       0.406       0.516       0.609
sys         0.247       0.048       0.156       0.250       0.359
```

 این مقدار محاسبه و عددی که در playground نشان داده می‌شود بحثی است که بعدا باید به آن بپردازم. خلاصه بگم، معمولاً نباید نگران عملکرد قانون خود باشید. مقدار زیادی از وقت Semgrep صرف خواندن و پردازش فایل‌ها و سپس درست کردن Abstract Syntax Tree (AST) کُد می‌شود. خیلی regex های پیچیده ننویسید اما دیگر نگران ذره ذره مسائل هم نباشید. برای دیدن زمانی که صرف هر بخش شده از سوییچ `time--` استفاده کنید.
  
### Java - HttpOnly Cookies
می‌خواهیم چک کنیم که آیا کوکی ما دارای خصوصیت `HttpOnly` است و اگر نیست آن را درست کنیم. من قانون [cookie-missing-httponly][java-missing-httponly] جاوا را خلاصه کرده‌ام:

[java-missing-httponly]: https://github.com/returntocorp/semgrep-rules/blob/develop/java/lang/security/audit/cookie-missing-httponly.yaml

``` yaml
# java-httponly/httponly-practice.yaml
rules:
- id: cookie-missing-httponly
  message: Match found
  severity: WARNING
  languages: [java]
  patterns:
  - pattern-not-inside: $COOKIE.setValue(""); ...
  - pattern-either:
    - pattern: $COOKIE.setHttpOnly(false);
    - patterns:
      - pattern-not-inside: $COOKIE.setHttpOnly(...); ...
      - pattern: $RESPONSE.addCookie($COOKIE);
```

این قانون (https://semgrep.dev/s/parsiya:java-httponly-practice) چک می‌کند که آیا:

1. ما به صورت دستی مقدار HttpOnly را false کرده‌ایم. در این صورت باید مقدار false را به true تغییر دهیم.
2. یک کوکی به response (پاسخ؟) اضافه کرده‌ایم ولی HttpOnly را سِت نکرده‌ایم. در این صورت باید مقدار HttpOnly را به true تغییر دهیم

برای رفع این مشکل باید این قانون را به این دو بخش بشکانیم و دو قانون مجزا درست کنیم زیرا بخش fix برای همه قوانین اجرا می‌شود ولی ما دو نوع ترمیم مختلف داریم.

#### ترمیم HttpOnly اول
در این ترمیم تنها باید مقدار false در ;COOKIE.setHttpOnly(false)$را با true جایگزین کنیم (برای تمرین از https://semgrep.dev/s/parsiya:java-httponly-practice-1 استفاده کنید):

``` yaml
# java-httponly/httponly-practice-1.yaml
rules:
- id: cookie-missing-httponly-1
  message: Match found
  severity: WARNING
  languages: [java]
  patterns:
  - pattern-not-inside: $COOKIE.setValue(""); ...
  - pattern: $COOKIE.setHttpOnly(false);
  fix: $COOKIE.setHttpOnly(true);
```

{{< figure src="02-httponly-1.png" title="semgrep -c httponly-practice-1.yaml httponly.java" >}}

#### ترمیم HttpOnly دوم

در این بخش ما `;RESPONSE.addCookie($COOKIE)$` را می‌بینیم اما خبری از `setHttpOnly` نیست.باید آن را اضافه کنیم. این قانون (https://semgrep.dev/s/parsiya:java-httponly-practice-2) این کار را انجام می‌دهد اما کد اضافه شده "خوشگل" نیست (برای جاوا فاصله و غیره مهم نیستند و شاید شما بگویید که این قانون برای من کافی است زیرا ابزارهای دیگری دارید که اتوماتیک کُد شما را فُرمَت می‎‌کنند).

{{< figure src="03-httponly-2.png" title="کُدِ زشت" >}}

ما می‌توانیم این مشکل را با فرمان `fix-regex` حل کنیم.

## فرمان fix-regex
همانطور که دیدیم برای جابجایی‌های ساده (مثلا تغییر `badFunc` به `goodFunc`) فرمان fix کافی است. اما fix-regex به ما قدرت مانور بیشتری می‌دهد.

* https://semgrep.dev/docs/experiments/overview/#autofix-with-regular-expression-replacement

این فرمان سه بخش دارد:

 1. بخش regex که regular expression (فارسیش واقعا چی میشه؟ اصلاً معادل داره؟) را روی بخشی از کد که توسط قانون پیدا شده است، اجرا می‌کند.
2. بخش replacement: چیزی که باید با بخش بالا جایگزین شود.
3. بخش count: تعداد جایگزینی‌ها.

*نکته: در حال حاضر (آوریل 2022) Semgrep از metavariable در fix-regex پشتیبانی نمی‌کند.*

با استفاده از fix-regex می‌توانیم قانون قبلی را دوباره بنویسیم. برای این کار اول باید ببینیم که چه چیزی capture می‌شود. این قانون را می‌توانید در https://semgrep.dev/s/parsiya:java-httponly-fix-regex-practice اجرا کنید.

``` yaml
# java-httponly/httponly-fix-regex-practice.yaml
rules:
  - id: cookie-missing-httponly-fix-regex-practice
    message: Match found
    severity: WARNING
    languages:
      - java
    patterns:
      - pattern-not-inside: $COOKIE.setValue(""); ...
      - pattern-not-inside: $COOKIE.setHttpOnly(...); ...
      - pattern: $RESPONSE.addCookie($COOKIE);
    fix-regex:
      regex: (.*)
      replacement: //\1
```

{{< figure src="04-httponly-fix-regex-1.png" title="نتیجه اجرای قانون بالا" >}}

در جواب، `//`را دو بار می‌بینیم. چون regex ما greedy (حریص) است. برای این کار باید از فیلد `count` با مقدار یک استفاده کنیم که فقط اولین جایگزینی انجام شود.

{{< figure src="05-httponly-fix-regex-2.png" title="اجرای قانون با count برابر یک" >}}

ولی هنوز کُدِ خوشگل نداریم. برای این کار در بخش regex باید فاصله بین ابتدای خط تا اول کُد را capture کنیم. بعد در بخش replacement اول فاصله را جایگزین کنیم (`1\`) و بعد `//` و در انتها خود کُد (`2\`).

``` yaml
    fix-regex:
      regex: (\s*)(.*)
      replacement: \1// \2
      count: 1
```

{{< figure src="06-httponly-fix-regex-3.png" title="" >}}

حالا باید در خط جدید، کُدِ درست را وارد کنیم. اگر می‌توانستیم از metavariable ها استفاده کنیم کار ما خیلی راحت‌تر بود اما ترمیم پایین درست اجرا نمی‌شود. این را در https://semgrep.dev/s/parsiya:java-httponly-fix-regex-practice-2 می‌توانید امتحان کنید.

``` yaml
# java-httponly/httponly-fix-regex-practice-2.yaml
    fix-regex:
      regex: (\s*)(.*)
      replacement: |
        \1$COOKIE.setHttpOnly(true);
        \1\2
      count: 1
```

{{< figure src="07-httponly-fix-regex-4.png" title="مقدار COOKIE$ در ترمیم جایگزین نشده است" >}}

برای این کار باید از [قانون زیر][rule1] استفاده کنیم (واقعا حوصله توضیح دوباره آن را ندارم 😅):

[rule1]: https://semgrep.dev/s/parsiya:java-httponly-fix-regex-practice-2

{{< figure src="08-httponly-fix-regex-final.png" title="" >}}

``` yaml
# java-httponly/httponly-fix-regex-practice-final.yaml
rules:
  - id: cookie-missing-httponly-fix-regex-practice-final
    message: Match found
    severity: WARNING
    languages:
      - java
    patterns:
      - pattern-not-inside: $COOKIE.setValue(""); ...
      - pattern-not-inside: $COOKIE.setHttpOnly(...); ...
      - pattern: $RESPONSE.addCookie($COOKIE);
    fix-regex:
      regex: (\s*)(.*addCookie\((.*)\).*)
      replacement: |
        \1\3.setHttpOnly(true);
        \1\2
      count: 1
```

بقیه بلاگ دیگر چیز جدیدی به شما یاد نمی‎‌دهد. اما می‌توانید در [بلاگ انگلیسی][english-blog] بخوانید.

[english-blog]: https://parsiya.net/blog/2021-10-25-a-hands-on-intro-to-semgreps-autofix/

# چه یاد گرفتم؟
قابلیت ترمیم خودکار Semgrep بسیار جالب است. من از آن برای اضافه کردن comment به کد استفاده می‌کنم تا هنگام مرور آن بدانم هر بخش چه مشکلی دارد.