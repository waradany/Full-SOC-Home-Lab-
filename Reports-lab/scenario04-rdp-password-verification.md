# السيناريو الرابع: التحقق من كلمة مرور RDP (Password Verification) وربطها بالكشف عبر SIEM/IDS

## نظرة عامة

| | |
|---|---|
| **التاريخ** | 2026-07-18 |
| **البيئة** | SOC Home Lab (pfSense + Snort IDS/IPS \| Kali Linux \| Splunk SIEM \| Windows + Sysmon \| Metasploitable) |
| **الهدف** | 192.168.2.101 (Windows + Sysmon) |
| **المهاجم** | 192.168.1.102 (Kali Linux) |
| **البروتوكول المستهدف** | RDP (منفذ 3389) |

## الهدف من السيناريو

التأكد من صحة بيانات دخول معروفة مسبقًا (Administrator/Ahmed - Admin2001$$) عبر بروتوكول RDP، بافتراض أن كلمة المرور الصحيحة موجودة داخل قائمة اختبار (wordlist)، مع مراقبة وتوثيق كيفية استجابة طبقات الحماية والكشف (IDS وSIEM) لهذا النشاط.

## الأدوات المستخدمة

- **Hydra** — أداة اختبار كلمات مرور آلية (brute force testing)
- **Nmap** — لفحص المنفذ والتحقق من إعدادات RDP (`rdp-enum-encryption`)
- **xfreerdp** — للاتصال اليدوي والتحقق المباشر من بيانات الدخول
- **Snort (على pfSense)** — لرصد نشاط الفحص/الاتصال
- **Splunk** — لتحليل الأحداث الأمنية (Windows Security Event Log)

## خطوات التنفيذ

### 1. تجهيز قوائم الاختبار
تم إنشاء `users.txt` و `passwords.txt` على Kali، مع تضمين كلمة المرور الصحيحة المفترضة (`Admin2001$$`) وسط مجموعة كلمات مرور شائعة أخرى، لمحاكاة سيناريو واقعي.

### 2. تشغيل Hydra
```bash
hydra -L users.txt -P passwords.txt rdp://192.168.2.101
```
**النتيجة:** `0 valid password found` — فشلت الأداة في اكتشاف كلمة المرور رغم وجودها في القائمة، مع تكرار رسالة:
```
account on 192.168.2.101 might be valid but account not active for remote desktop
```

📸 `screenshots/scenario04-hydra-bruteforce-execution.png`
📸 `screenshots/scenario04-bruteforce-result.png`

### 3. تشخيص سبب فشل Hydra

**أ. التحقق من حالة المنفذ:**
```bash
nmap -p 3389 --script rdp-enum-encryption 192.168.2.101 -Pn
```
**النتيجة:** المنفذ 3389 مفتوح، وطبقة الأمان `CredSSP (NLA): SUCCESS` تعمل بنجاح — أي أن RDP والشبكة سليمان تمامًا.

📸 `screenshots/scenario04-rdp-enum-check.png`

**الاستنتاج:** فشل Hydra لم يكن بسبب مشكلة في الشبكة أو RDP، بل بسبب صعوبة الأداة في إتمام مصافحة NLA (Network Level Authentication) بشكل صحيح — وهي مشكلة تقنية معروفة تواجهها أدوات brute force التقليدية أمام أنظمة الحماية الحديثة.

### 4. التحقق اليدوي عبر xfreerdp

**محاولة أولى (باليوزر Administrator):** فشلت بخطأ `NLA_LOGON_FAILURE` — أثبتت أن اليوزر Administrator غير صحيح لهذا الحساب تحديدًا.

📸 `screenshots/scenario04-manual-rdp-test.png`

**التحقق من اسم اليوزر الصحيح مباشرة على الجهاز الهدف:**
```cmd
whoami
```
**النتيجة:** `desktop-rom0e89\ahmed`

📸 `screenshots/scenario04-windows-username-check.png`

**محاولة ثانية (باليوزر الصحيح ahmed):**
```bash
xfreerdp /v:192.168.2.101 /u:ahmed /p:'Admin2001$$'
```
**النتيجة:** ✅ نجاح — ظهرت رسائل تحميل الجلسة بنجاح (`Local/Remote framebuffer format`)، مما يؤكد نجاح المصادقة قبل أن يتم قطع الجلسة لاحقًا من جهة الخادم (`ERRINFO_RPC_INITIATED_DISCONNECT`) — وهو تفصيل ثانوي لا يؤثر على نتيجة التحقق من كلمة المرور.

📸 `screenshots/scenario04-manual-rdp-ahmed-result.png`
📸 `screenshots/scenario04-manual-rdp-success.png`

### 5. الكشف عبر Snort (pfSense)

تم رصد تنبيهات متعددة بعنوان **"ET SCAN RDP Connection Attempt from Nmap"** من 192.168.1.102 إلى 192.168.2.101:3389، ما يؤكد قدرة IDS على رصد أنشطة الفحص/الاتصال بـ RDP.

📸 `screenshots/scenario04-snort-nmap-rdp-alerts.png`

### 6. الكشف عبر Splunk (SIEM)

**بحث أولي:**
```
index=* EventCode=4624
```
156 حدثًا خلال آخر 24 ساعة — تأكيد أن تسجيل أحداث Windows الأمنية يصل بنجاح إلى Splunk.

📸 `screenshots/scenario04-splunk-detection.png`

**بحث مُدقق (مطابق لحدث تسجيل الدخول محل الاختبار):**
```
index=* EventCode=4624 Account_Name=ahmed Logon_Type=10
```
**النتيجة:** 4 أحداث، أولها بتوقيت `01:50:58 PM` بتاريخ 18/7/2026 — متطابق زمنيًا مع لحظة نجاح الاتصال اليدوي عبر xfreerdp، ويؤكد أن Splunk سجّل حدث تسجيل الدخول عبر RDP تحديدًا (Logon Type 10 = RemoteInteractive).

📸 `screenshots/scenario04-splunk-4624-ahmed-rdp.png`

## النتائج والاستنتاجات

| العنصر | النتيجة |
|---|---|
| كلمة المرور الصحيحة | `ahmed / Admin2001$$` (تم التأكد منها فعليًا) |
| نجاح Hydra الآلي | ❌ فشل بسبب تعارض تقني مع NLA |
| نجاح التحقق اليدوي (xfreerdp) | ✅ نجح وأثبت صحة بيانات الدخول |
| رصد Snort/IDS | ✅ رصد نشاط الفحص والاتصال بـ RDP |
| رصد Splunk/SIEM | ✅ رصد حدث تسجيل الدخول (4624, Logon_Type=10) بدقة زمنية متطابقة |

## دروس مستفادة

1. **أدوات brute force التقليدية (مثل Hydra) قد تفشل تقنيًا** أمام آليات حماية حديثة مثل NLA، حتى لو كانت بيانات الدخول صحيحة فعليًا — وهذا لا يعني بالضرورة خطأ في البيانات نفسها.
2. **التحقق اليدوي يظل أداة تشخيصية أساسية** عند الشك في نتائج أداة آلية.
3. **الطبقات الدفاعية المتعددة (IDS + SIEM) أثبتت فاعليتها** في رصد النشاط بمراحله المختلفة (فحص → محاولة اتصال → تسجيل دخول ناجح)، وهو ما يمثل قيمة السيناريو التعليمية الحقيقية أكثر من نجاح أداة الهجوم نفسها.
4. أهمية التحقق من اسم المستخدم الصحيح (`whoami`) قبل افتراض أن الحساب الافتراضي (Administrator) هو المقصود دائمًا.

## قائمة الصور المطلوب حفظها (screenshots/)

- `scenario04-hydra-bruteforce-execution.png`
- `scenario04-bruteforce-result.png`
- `scenario04-rdp-enum-check.png`
- `scenario04-manual-rdp-test.png`
- `scenario04-windows-username-check.png`
- `scenario04-manual-rdp-ahmed-result.png`
- `scenario04-manual-rdp-success.png`
- `scenario04-snort-nmap-rdp-alerts.png`
- `scenario04-splunk-detection.png`
- `scenario04-splunk-4624-ahmed-rdp.png`
