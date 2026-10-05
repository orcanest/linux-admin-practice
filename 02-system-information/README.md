## System Information
### 🐧 An Overview of Monitoring, System Information, and System Metadata Tools in Linux

در لینوکس ، ابزارهایی مانند uname و hostname و  lscpu و free و uptime برای مشاهده اطلاعات سیستم ، وضعیت resource و برخی Metadata های مربوط به سیستم استفاده می‌ شوند. این ابزارها در User Space اجرا می‌ شوند و بسته به نوع Command ، اطلاعات مورد نیاز خود را از Interface های مختلف Linux مانند System Call ها و procfs (/proc) و  sysfs (/sys) و  فایل‌ های Configuration و سایر Interface های User Space دریافت می‌ کنند.

بنابراین همه Command ها الزاماً یک مسیر یکسان مانند ```Command → glibc → System Call → Kernel``` را طی نمی‌ کنند. برخی مستقیماً proc/ یا sys/ را می‌ خوانند ، برخی از System Call استفاده می‌ کنند و برخی نیز اطلاعات را از چند منبع مختلف جمع‌آوری و ترکیب می‌ کنند.

---

### 🧠 Under the Hood Architecture

به‌ صورت کلی ، هنگام اجرای ابزارهای System Information و Monitoring ، داده‌ ها می‌ توانند از مسیرهای مختلفی در اختیار User Space قرار بگیرند :

#### 1️⃣ -  استفاده از System Call یا Kernel API

وقتی یک User یک command را اجرا می‌ کند ، معمولاً خود command به‌ تنهایی نمی‌ تواند مستقیماً به منابع اصلی سیستم مثل Process ، Memory ، Disk ، Network یا Hardware دسترسی داشته باشد. برای این کار باید از امکاناتی که Kernel در اختیار User Space قرار داده استفاده کند که جریان کلی به این شکل است :

```bash
User
  ↓
Shell
  ↓
Command / Executable
  ↓
Library / User-Space Interface
  ↓
System Call / Kernel Interface
  ↓
Kernel Subsystem
  ↓
Kernel Data Structures
  ↓
User-Space Output
```

در ابتدا User یک command را در Terminal وارد می‌ کند. Shell command را بررسی می‌ کند و اگر لازم باشد executable مربوط به آن را اجرا می‌ کند. خود Command / Executable در User Space اجرا می‌ شود. اگر برای انجام کارش به اطلاعات یا منابعی نیاز داشته باشد که در اختیار Kernel هستند، نمی‌ تواند مستقیماً وارد Kernel شود. در اینجا از Library یا سایر User-Space Interface ها استفاده می‌ کند. این Interface ها معمولاً در نهایت یک System Call را اجرا می‌ کنند. System Call راه استانداردی است که یک برنامه در User Space از طریق آن از Kernel درخواست سرویس می‌ کند.

مثلاً یک برنامه برای خواندن فایل ، نوشتن در فایل ، ساختن Process ، گرفتن اطلاعات سیستم ، ارسال Packet در شبکه و گرفتن اطلاعات از Memory می‌تواند از System Call های مربوطه استفاده کند.  بعد از System Call ، درخواست وارد **Kernel Space** می‌ شود و Kernel آن را به **Kernel Subsystem** مربوطه مثل File System یا Process Management یا Memory Management یا Networking یا Device Drivers  می‌ دهد. Kernel با استفاده از **Kernel Data Structures** اطلاعات مورد نیاز را پیدا یا تغییر می‌ دهد و نتیجه را دوباره به User Space برمی‌ گرداند. در نهایت command یا برنامه نتیجه را دریافت کرده و آن را به شکل ** User-Space Output** ، مثلاً در Terminal نمایش می‌ دهد. 

به عنوان فرض کنیم دستور ```uname -r``` را اجرا کنیم و جریان کلی می‌ تواند به این شکل باشد :
```bash
User
  ↓
Shell
  ↓
uname
  ↓
glibc / User-Space Interface
  ↓
uname() System Call
  ↓
Kernel
  ↓
UTS Namespace / Kernel Data
  ↓
Kernel → User Space
  ↓
uname -r
  ↓
6.x.x-xx-generic
```

💡 نکته مهم این است که **System Call همان مرز اصلی بین User Space و Kernel Space است**. برنامه‌ های User Space برای انجام بسیاری از کارهای حساس و دسترسی به منابع سیستم ، از طریق همین Interface با Kernel ارتباط برقرار می‌ کنند. به زبان خیلی ساده **Command درخواست را می‌ دهد ، System Call درخواست را به Kernel می‌ رساند ، Kernel کار را انجام می‌ دهد و نتیجه را به برنامه بر می‌ گرداند**.

#### 💎 مرز بین User Space و Kernel Space

در Linux دو محیط اصلی داریم به نام User Space و Kernel Space که برنامه‌هایی مثل bash ، ls ، uname ، ssh و بیشتر برنامه‌ های معمولی در **User Space** اجرا می‌ شوند. Kernel در **Kernel Space** اجرا می‌ شود و دسترسی بسیار بیشتری به resource های سیستم دارد. یک برنامه در User Space نمی‌ تواند هر کاری که خواست مستقیماً روی این منابع انجام دهد. برای درخواست بعضی از این کارها باید از **System Call** استفاده کند. پس می‌توان گفت **System Call یکی از مرزهای اصلی ارتباط بین User Space و Kernel Space است**.

##### 🔹 تعریف Library : 

در لینوکس **Library خودش Kernel نیست و System Call هم نیست** ، در واقع Library یک مجموعه کد آماده است که در User Space قرار دارد و برنامه‌ ها می‌ توانند از آن استفاده کنند. مثلاً در Linux یکی از مهم‌ ترین Library ها glibc است. برنامه‌ ای که می‌ خواهد یک کار مشخص انجام دهد ، خیلی وقت‌ ها به‌ جای اینکه مستقیماً با جزئیات System Call کار کند ، از یک function داخل Library استفاده می‌ کند و Library کار برنامه‌ نویس را ساده‌ تر می‌کند و خیلی از جزئیات Low-level را خودش مدیریت می‌ کند ، مثلاً :
```bash
Application
     ↓
glibc function
     ↓
System Call
     ↓
Kernel
```

##### 🔹 تعریف System Call :

در لینوکس **System Call یک درخواست رسمی از User Space به Kernel است**. وقتی یک برنامه نیاز دارد کاری را انجام دهد که فقط Kernel می‌ تواند انجام دهد ، از System Call استفاده می‌ کند ، مثلاً :

```bash
open()    → Open a file
read()    → Read data
write()   → Write data
fork()    → Create a process
execve()  → Execute a program
socket()  → Create a network socket
```

البته باید دقت کنیم که اسم‌هایی مثل()open یا()read می‌ توانند در سطح Library هم به شکل function در اختیار برنامه باشند. چیزی که در نهایت مهم است ، **ورود درخواست به Kernel از طریق System Call Interface** است.

#### 💎 تفاوت Library و System Call

به ساده‌ ترین شکل ، Library یک ابزار و واسط در User Space است و System Call راه ورود درخواست از User Space به Kernel است بنابراین Library قبل از مرز Kernel قرار دارد ، ولی System Call مرز ارتباط با Kernel را طی می‌ کند.

```
User Space
──────────────────────────────

Application
     ↓
Library Function
     ↓
System Call Interface
     
──────────────────────────────
        Kernel Boundary
──────────────────────────────

     ↓
Kernel Subsystem
     ↓
Kernel Data
```

به عنوان مثال فرض کنیم یک برنامه می‌ خواهد یک File را بخواند و جریان کلی می‌ تواند این‌ طور باشد :
```bash
Application
     ↓
fopen() / fread()
     ↓
glibc
     ↓
openat() / read()
     ↓
System Call
     ↓
Kernel
     ↓
VFS / Filesystem
     ↓
Storage Device
```

اینجا glibc در User Space است. برنامه از function های glibc استفاده می‌ کند و glibc در صورت نیاز System Call مناسب را انجام می‌ دهد. بعد درخواست وارد Kernel می‌ شود و Kernel عملیات مربوط به File System و Storage را انجام می‌ دهد. پس مرز دقیقاً کجاست؟ می‌توانیم تصویر را این‌ طور ببینیم :

```bash
USER SPACE
─────────────────────────────────────
User
↓
Shell
↓
Command / Application
↓
Library (e.g., glibc)
↓
System Call Wrapper
│
│
│  ← **Still User Space here**
│
═════════════════════════════════════
SYSTEM CALL BOUNDARY
═════════════════════════════════════
│
│  ← **Entry into Kernel**
↓
System Call Handler
↓
Kernel Subsystem
↓
Kernel Data Structures
↓
Hardware / Resources
─────────────────────────────────────
KERNEL SPACE
```

یعنی خود Library در User Space اجرا می‌ شود. وقتی Library یا برنامه System Call را درخواست می‌ کند ، CPU یک transition از User Mode به Kernel Mode انجام می‌ دهد و Kernel کنترل اجرای عملیات را به دست می‌گیرد. بعد از اینکه Kernel کار را انجام داد ، نتیجه را برمی‌ گرداند و اجرای برنامه دوباره در User Space ادامه پیدا می‌ کند :
```bash
User Space
    │
    │ System Call
    ↓
Kernel Space
    │
    │ return
    ↓
User Space
```

💡 یک نکته خیلی مهم اینکه هر function که اسمش شبیه System Call است ، لزوماً خودش مستقیماً وارد Kernel نمی‌ شود مثلاً()printf اصلاً System Call نیست در واقع()printf یک function در Library است و در User Space اجرا می‌ شود. در نهایت برای نمایش Data ممکن است از مسیرهایی مثل :
```bash
printf()
   ↓
glibc
   ↓
write()
   ↓
System Call
   ↓
Kernel
```
استفاده کند بنابراین همیشه این سه مورد Library Function ≠ System Call ≠ Kernel Function را از هم جدا کنید. 

#### 💎 تفاوت Interface و Execution of operations

وقتی می‌گوییم Interface ، منظورمان خود عملیات نیست بلکه منظور راهی است که از طریق آن درخواست یک عملیات را مطرح می‌ کنیم مثلاً فرض کن یک برنامه می‌ خواهد اطلاعات یک File را بخواند.
```bash
Application
    ↓
Library Function
    ↓
System Call Interface
    ↓
Kernel
    ↓
Execution of operations
```
در اینجا Library یک Interface در User Space در اختیار برنامه قرار می‌ دهد و System Call Interface راه رسمی درخواست سرویس از Kernel است و Kernel جایی است که درخواست واقعاً پردازش می‌ شود پس Interface می‌ گوید چطور درخواست بده ، اما اجرای واقعی داخل Kernel انجام می‌ شود.

##### 🔹 یک مثال ساده برای درک Interface 

فرض کنیم در Terminal این دستور ```cat /etc/hostname```  را اجرا کنیم و ما فقط ```server01``` را می‌بینیم ، اما پشت صحنه اتفاقات بیشتری رخ داده است. cat باید File را باز کند و محتوای آن را بخواند به‌ صورت ساده :

```bash
cat →
 ↓
Library / User-Space Interface
 ↓
System Call
 ↓
Kernel
 ↓
Filesystem
 ↓
Disk / Page Cache
 ↓
Data
 ↓
cat
 ↓
Terminal
```
مثلاً برای خواندن فایل ، Kernel باید عملیات‌ هایی مثل باز کردن File و خواندن Data را انجام دهد.

#### 2️⃣ -  استفاده از Virtual Filesystem
```bash
User
  ↓
Shell
  ↓
Command / Executable
  ↓
File I/O
  ↓
VFS
  ↓
procfs (/proc) / sysfs (/sys)
  ↓
Kernel Subsystem
  ↓
Kernel Data Structures / Device Information
  ↓
User-Space Output
```

این مسیر برای ابزارهایی که اطلاعات خود را از proc/ یا sys/ دریافت می‌ کنند اهمیت زیادی دارد. نکته مهم این است که مسیر واقعی هر Command باید بر اساس Implementation همان Command و Version مورد نظر بررسی شود و نباید یک مسیر ثابت را برای همه ابزارها فرض کرد.

##### 🔹 شبه‌ فایل‌ سیستم‌ های proc/ و sys/

فایل سیستم procfs یک Virtual Filesystem است که Kernel اطلاعات مختلفی درباره Process ها و وضعیت Runtime سیستم را از طریق آن در اختیار User Space قرار می‌ دهد ، برخی فایل‌ های مهم عبارت‌ اند از :

```bash
/proc/cpuinfo
/proc/meminfo
/proc/uptime
/proc/loadavg
/proc/version
/proc/sys/kernel/
```
این اطلاعات معمولاً مستقیماً روی یک Disk File معمولی ذخیره نشده‌ اند ، بلکه هنگام دسترسی به Interface مربوطه توسط Kernel ارائه می‌ شوند ، به همین دلیل بسیاری از اطلاعات موجود در proc/ وضعیت جاری سیستم را نشان می‌ دهند.

فایل سیستم sysfs یک Virtual Filesystem برای نمایش ساختار و روابط Object های Kernel است و اطلاعات مربوط به Device ها و Driver ها و Bus ها و CPU ها و Memory و بسیاری از بخش‌ های دیگر Kernel را در اختیار User Space قرار می‌ دهد برای مثال :

```bash
/sys/devices/system/cpu/
/sys/devices/system/memory/
```

در نتیجه ابزارهایی مانند lscpu و lsmem می‌ توانند برای جمع‌ آوری بخشی از اطلاعات خود از sysfs استفاده کنند.

##### 🔹 ساختار UTS Namespace

برخی اطلاعات مربوط به هویت سیستم ، مانند Hostname و مشخصات Kernel در Linux با UTS Namespace مرتبط هستند. Kernel اطلاعات مربوط به UTS را در ساختارهای داخلی مرتبط با uts_namespace نگهداری می‌ کند و Process ها در یک UTS Namespace مشخص ، این اطلاعات را مشاهده می‌ کنند. این موضوع در محیط‌ های Container اهمیت زیادی دارد ، زیرا Container می‌ تواند UTS Namespace جداگانه‌ ای داشته باشد و در نتیجه Hostname متفاوتی نسبت به Host مشاهده کند. این جداسازی به این معنا نیست که کل اطلاعات سیستم برای Container کاملاً مستقل است بلکه فقط Namespace هایی که جدا شده‌اند ، View متفاوتی از منابع مربوطه ارائه می‌ کنند.


## 🐬 بررسی تفکیکی و عمیق Command ها

### 🔧 Unix Name command (uname)


دستور uname یکی از command های پایه در سیستم‌ های Unix و Linux است. کار اصلی آن نمایش اطلاعات مربوط به Kernel و معماری سخت‌ افزار و Hostname سیستم است. برخلاف بعضی ابزارها که اطلاعات را از config file های موجود روی Disk می‌ خوانند ، uname اطلاعات را مستقیماً از Kernel در حال اجرا می‌ گیرد و برای این کار از System Call به نام ()uname استفاده می‌ کند.

**خروجی** : اگر uname را بدون هیچ option اجرا کنید ، فقط نام Kernel را نشان می‌ دهد مثلاً :
```bash
:~$ uname
Linux
```

#### 🧰 Under the Hood

وقتی این Command را اجرا می‌ کنیم، از لحظه اجرا تا نمایش Output دقیقاً چه اتفاقی می‌ افتد ؟

🔸 اجرا در User Space : کاربر دستور uname را در Shell وارد می‌ کند. Shell با ایجاد یک Process جدید ، باینری مربوطه (معمولاً در /usr/bin/uname از مجموعه GNU Coreutils) را بارگذاری کرده و تابع ()main را اجرا می‌ کند.

🔸 فراخوانی Interface : ابزار uname برای دریافت داده به جای خواندن فایل از دیسک ، مستقیماً از تابع کتابخانه‌ ای ()uname  در C Standard Library (glibc) استفاده می‌ کند. این تابع یک System Call به نام sys_uname (یا sys_newuname) صادر می‌ کند.

🔸 لایه Kernel و Data Structure : با وقوع Context Switch و انتقال کنترل به Kernel Space ، هسته به Metadata management subsystem و ساختار داده struct uts_namespace مراجعه می‌ کند. در سیستم‌ های لینوکس مدرن، Metadata هویت سیستم درون ساختار struct new_utsname که درون uts_namespace قرار دارد نگهداری می‌ شود. این ساختار شامل فیلد های متنی زیر است :

  - sysname: Kernel name (default: Linux).
  - nodename: Network node name (Hostname).
  - release: Kernel release version.
  - version: Kernel build version and compilation date.
  - machine: Hardware architecture (e.g., x86_64).

🔸 تفاوت Interface و Data Source در اینکه Interface برای فراخوانی سیستمی ()uname است و Data Source هم ساختار داده struct uts_namespace در حافظه RAM هسته که در زمان ساخت هسته و بوت مقداردهی شده است.  شبه‌ فایل‌ های /proc/sys/kernel/ostype و /proc/sys/kernel/osrelease نیز رابط‌ های procfs برای نمایش همین داده‌ های موجود در uts_namespace هستند.

🔸 بازگشت داده و پردازش خروجی : هسته داده‌ های موجود در struct new_utsname را به آرگومان اشاره‌ گر حافظه در User Space کپی می‌ کند. سپس دستور uname بر اساس Option های ورودی مانند a- یا r- ، سوئیچ‌ های مربوطه را بررسی کرده ، رشته‌ ها را کنار هم قرار داده و خروجی را در stdout چاپ می‌ کند.

🔸 بخشی از اطلاعات مرتبط با uname در Interface های proc/ نیز قابل مشاهده هستند اما نباید فرض کرد که uname صرفاً با خواندن همین فایل‌ ها کار می‌ کند. uname در Linux Interface مستقیمی برای دریافت UTS Information دارد ، برای مثال :
- /proc/sys/kernel/ostype
- /proc/sys/kernel/osrelease
- /proc/version

#### ✅ Linux Administration

یکی از کاربرد های مهم uname -r بررسی Kernel Release فعلی است ، این مقدار معمولاً هنگام بررسی Kernel Modules نیز اهمیت دارد :
```bash
:~$ ls -l /lib/modules/$(uname -r)
total 8076
lrwxrwxrwx  1 root root      39 Sep  9 19:31 build -> /usr/src/linux-headers-7.0.0-38-generic
drwxr-xr-x  2 root root    4096 Sep  9 19:31 initrd
drwxr-xr-x 22 root root    4096 Oct  2 17:58 kernel
drwxr-xr-x  2 root root    4096 Oct  2 17:39 misc
-rw-r--r--  1 root root 1817568 Oct  2 17:58 modules.alias
-rw-r--r--  1 root root 1760629 Oct  2 17:58 modules.alias.bin
-rw-r--r--  1 root root   10431 Sep  9 19:31 modules.builtin
-rw-r--r--  1 root root   73581 Oct  2 17:58 modules.builtin.alias.bin
-rw-r--r--  1 root root   12411 Oct  2 17:58 modules.builtin.bin
-rw-r--r--  1 root root  159067 Sep  9 19:31 modules.builtin.modinfo
-rw-r--r--  1 root root  963682 Oct  2 17:58 modules.dep
-rw-r--r--  1 root root 1263280 Oct  2 17:58 modules.dep.bin
-rw-r--r--  1 root root     353 Oct  2 17:58 modules.devname
-rw-r--r--  1 root root  282013 Sep  9 19:31 modules.order
-rw-r--r--  1 root root    2384 Oct  2 17:58 modules.softdep
-rw-r--r--  1 root root  852538 Oct  2 17:58 modules.symbols
-rw-r--r--  1 root root 1024198 Oct  2 17:58 modules.symbols.bin
drwxr-xr-x  3 root root    4096 Oct  2 00:53 ubuntu
drwxr-xr-x  3 root root    4096 Oct  2 00:53 vdso
```

برای مثال ، Administrator می‌ تواند بررسی کند آیا Kernel جدید واقعاً بعد از Reboot در حال اجرا است یا سیستم هنوز با Kernel قبلی Boot شده است.

#### ⚙️ uname Options


- ##### uname -a / --all

تقریباً تمام اطلاعات مهم سیستم را نمایش می‌دهد. مثل نام Kernel و Hostname و Kernel Release و نسخه و زمان Build شدن Kernel و معماری سخت‌افزار و نوع Processor و نام سیستم‌ عامل مثل GNU/Linux.

```bash
:~$ uname -a
Linux computerName 7.0.0-38-generic #38~24.04.4-Ubuntu SMP PREEMPT_DYNAMIC Mon Sep 14 16:37:11 UTC 2 x86_64 x86_64 x86_64 GNU/Linux
```

- ##### uname -r / --kernel-release

نسخه Kernel Release فعلی را نشان می‌ دهد.
```bash
:~$ uname -r
7.0.0-38-generic
or
3.10.0-862.2.3.el7.x86_64
```

- ##### uname -n / --nodename

نام Hostname سیستم را نشان می‌ دهد.
```bash
:~$ uname -n
orcanestlab
```

- ##### uname -s / --kernel-name

نام Kernel را نشان می‌ دهد.
```bash
:~$ uname -s
Linux
```

- ##### uname -v / --kernel-version

اطلاعات مربوط به نسخه و زمان Build شدن Kernel را نمایش می‌ دهد.
```bash
:~$ uname -v
#38~24.04.4-Ubuntu SMP PREEMPT_DYNAMIC Mon Sep 14 16:37:11 UTC 2
```

- ##### uname -m / --machine

معماری سخت‌ افزار سیستم را نشان می‌ دهد :
```bash
:~$ uname -m
x86_64
```

- ##### uname -p / --processor

نوع Processor سیستم را نمایش می‌ دهد.
```bash
:~$ uname -p
x86_64
```

- ##### uname -i / --hardware-platform

اطلاعات Hardware Platform سیستم را نمایش می‌ دهد.
```bash
:~$ uname -i
x86_64
```

- ##### uname -o / --operating-system

نام سیستم‌ عامل را نمایش می‌ دهد.
```bash
:~$ uname -o
GNU/Linux
```

#### 💡 Tips

- دستور uname فقط مخصوص Linux نیست و در سیستم‌ های Unix و سیستم‌ هایی مثل FreeBSD ، OpenBSD و macOS هم وجود دارد.
- از خروجی uname -r می‌ توان داخل Shell Script هم استفاده کرد. مثلاً برای دسترسی پویا به مسیر Module های  Kernel :
```bash
:~$ ls /lib/modules/`uname -r`
build   kernel  modules.alias      modules.builtin            modules.builtin.bin      modules.dep      modules.devname  modules.softdep  modules.symbols.bin  vdso
initrd  misc    modules.alias.bin  modules.builtin.alias.bin  modules.builtin.modinfo  modules.dep.bin  modules.order    modules.symbols  ubuntu
```

---

### 🔧 hostname command

دستور hostname برای نمایش یا تغییر Runtime Hostname سیستم استفاده می‌ شود. اگر hostname را بدون گزینه اجرا کنید ، Hostname فعلی سیستم را نشان می‌ دهد :
```bash
:~$ hostname
myhost.example.com
```

#### 🧰 Under the Hood

وقتی این Command را اجرا می‌ کنیم ، از لحظه اجرا تا نمایش Output دقیقاً چه اتفاقی می‌ افتد؟

🔸 اجرا در User Space : Shell پردازش فرعی برای /bin/hostname اجرا می‌کند.

🔸 فراخوانی Interface : ابزار hostname از فراخوانی‌ های سیستمی POSIX شامل ()gethostname (برای خواندن) یا()sethostname (برای تنظیم) استفاده می‌ کند.

🔸 لایه Kernel و Data Source : اول Interface فراخوانی سیستمی ()gethostname می کند و Data  Source  فیلد nodename درون ساختار struct uts_namespace جاری پردازش می کند. هنگام اجرا ، kernel مقدار nodename را از حافظه استخراج کرده و به User Space بازمی‌ گرداند. اگر دستور همراه با Option هایی مانند f- برای FQDN اجرا شود ، ابزار hostname از طریق کتابخانه NSS (Name Service Switch) و تابع ()getaddrinfo ، فایل‌ های /etc/hosts و /etc/nsswitch.conf یا DNS را جستجو می‌ کند تا نام کامل دامنه را استخراج کند.

🔸ذخیره‌ سازی و پایداری : تغییر نام با دستور hostname فقط متغیر حافظه کرنل (Runtime) را تغییر می‌ دهد و پس از Reboot پاک می‌ شود. برای پایداری ، تغییرات باید در فایل /etc/hostname بنویسد.

#### 🔩 Configuration

در بسیاری از Linux Distribution ها ، Static Hostname در فایل /etc/hostname نگهداری می‌شود ، اما مدیریت Hostname در Distribution های مختلف می‌ تواند توسط ابزارها و سرویس‌ های مختلف انجام شود.

#### ⚙️ hostname Options

- ##### hostname -f

نام کامل سیستم به همراه Domain را نمایش می‌ دهد.
```bash
:~$ hostname -f
server01.example.com
```

- ##### hostname -i

میتواند IP مربوط به Hostname را نمایش می‌ دهد.
```bash
:~$ hostname -i
127.0.1.1 / 192.168.1.10
```

- ##### hostname -I

آدرس‌ های IP اختصاص داده‌ شده به Interface های سیستم را نمایش می‌ دهد.
```bash
$ hostname -I
192.168.1.10 10.0.0.5
```

- ##### hostname -A

تمام FQDN هایی را که به سیستم مربوط هستند نمایش می‌ دهد.
```bash
:~$ hostname -A
orcanestlab.bbrouter
```

#### 🪛 Changing hostname

با دستور ```sudo hostname new_name``` می‌ توان Hostname را تغییر داد ، این تغییر معمولاً موقتی است و بعد از Reboot باقی نمی‌ ماند. برای اینکه تغییر دائمی باشد ، بسته به توزیع Linux باید تنظیمات مربوط به Hostname ، مثل ```etc/hostname/``` و ``` etc/hosts/```  یا تنظیمات شبکه ، به‌ درستی تغییر کنند.
```bash
:~$ sudo hostname orange
```

این تغییر Runtime است و نحوه حفظ آن پس از Reboot به روش مدیریت Hostname در Distribution بستگی دارد. برای Configuration دائمی ، بهتر است از ابزار مدیریت Hostname همان Distribution مانند hostnamectl در سیستم‌ های مبتنی بر systemd استفاده شود.

---

### 🔧 hostnamectl commmand

یک ابزار مربوط به سیستم‌ های مبتنی بر Systemd است که برای مشاهده و مدیریت Hostname و بعضی اطلاعات هویتی سیستم استفاده می‌ شود.

- **خروجی :** اگر hostnamectl را بدون آرگومان اجرا کنید ، اطلاعات مختلفی از سیستم نمایش داده می‌ شود ، از جمله :
  - Static Hostname
  - Icon Name
  - Chassis
  - Machine ID
  - Boot ID
  - Virtualization
  - Operating System
  - Kernel
  - Architecture
  - Hardware Vendor
