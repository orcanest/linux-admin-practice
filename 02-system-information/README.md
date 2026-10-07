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

ابزار مدیریت Hostname و برخی اطلاعات شناسایی سیستم در محیط‌ های مبتنی بر systemd است. اگر hostnamectl را بدون آرگومان اجرا کنید ، اطلاعات مختلفی از سیستم نمایش داده می‌ شود ، از جمله :
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

```bach
:~$ hostnamectl
 Static hostname: ASUS
       Icon name: computer-laptop
         Chassis: laptop 💻
      Machine ID: 46eda155dc23417aa5e3feaa19329sce
         Boot ID: 48f9909c43fc49ed9b903c17daopuj28
Operating System: Ubuntu 24.04.5 LTS                  
          Kernel: Linux 7.0.0-38-generic
    Architecture: x86-64
 Hardware Vendor: ASUS
  Hardware Model: ASUS vivobook Laptop 15-Sca0xxx
Firmware Version: F.29
   Firmware Date: Tue 2025-04-22
    Firmware Age: 1y 5month 2w 1d
```

#### 🧰 Under the Hood

وقتی این Command را اجرا می‌ کنیم ، از لحظه اجرا تا نمایش Output دقیقاً چه اتفاقی می‌ افتد ؟

- **اجرا در User Space :** اول دستور ```/usr/bin/hostnamectl``` اجرا می شود. این دستور بخشی از ابزارهای systemd هست و در User Space اجرا می شود.
- **ارتباط با systemd-hostnamed از طریق D-Bus :** اینجا یک نکته مهم وجود دارد ، برخلاف روش‌ های قدیمی تغییر یا دریافت hostname ، خود hostnamectl مستقیماً System Call خاصی برای انجام این کار صدا نمی‌ زند. در عوض، از طریق IPC و روی D-Bus ، یک پیام برای سرویس systemd-hostnamed.service ارسال می‌ کند.
- **پردازش درخواست توسط systemd-hostnamed :** سرویس systemd-hostnamed در پشت صحنه درخواست را دریافت می‌ کند و اطلاعات مختلف مربوط به سیستم را از منابع مختلف جمع‌ آوری می‌کند. مهم‌ ترین این منابع عبارت‌ اند از :

  - **اطلاعات Static Hostname** از فایل /etc/hostname خوانده می‌ شود.
  - **اطلاعات Pretty Hostname و Location و Chassis** از فایل /etc/machine-info  گرفته می‌شوند.
  - **اطلاعات Machine ID** از فایل /etc/machine-id  خوانده می‌ شود.
  - **اطلاعات Boot ID** از فایل مجازی /proc/sys/kernel/random/boot_id دریافت می‌ شود.
  - **اطلاعات Virtualization** که برای تشخیص اینکه سیستم روی چه نوع Hypervisor اجرا می‌ شود ، مثل DMI/SMBIOS یا مسیر /sys/hypervisor بررسی می‌ شوند.
  - **اطلاعات Operating System** مربوط به توزیع لینوکس از فایل /etc/os-release خوانده می‌ شود.
  - **اطلاعات Kernel و Architecture** مربوط به نسخه کرنل و معماری سیستم هم از طریق تابع()uname به دست می آید.

در نتیجه ، systemd-hostnamed فقط مسئول hostname نیست بلکه اطلاعات مختلفی از وضعیت و مشخصات سیستم جمع آوری می‌کند.

-  **بازگردانی اطلاعات به hostnamectl :** بعد از اینکه systemd-hostnamed اطلاعات مورد نیاز را جمع‌ آوری کرد ، نتیجه را دوباره از طریق D-Bus به hostnamectl برمی‌ گرداند.

در نهایت ، hostnamectl این اطلاعات رو به شکل **Key-Value** مرتب و روی ترمینال نمایش می‌ دهد.

```bash
User
│
│ Execution of hostnamectl
▼
/usr/bin/hostnamectl
│
│ D-Bus / IPC
▼
systemd-hostnamed.service
│
├── /etc/hostname
├── /etc/machine-info
├── /etc/machine-id
├── /etc/os-release
├── /proc/sys/kernel/random/boot_id
├── /sys/hypervisor
├── DMI / SMBIOS
└── uname()
│
│ Collected information
▼
systemd-hostnamed
│
│ D-Bus
▼
hostnamectl
│
▼
Display information in the terminal
```

#### 🔩 Configuration

اطلاعات مرتبط می‌ تواند شامل موارد زیر باشد :
```bash
/etc/hostname
/etc/machine-info
/etc/machine-id
/etc/os-release
/proc/sys/kernel/random/boot_id
```

مفاهیم مهم :

- **مفهوم Static Hostname :** نامی که به‌ صورت دائمی Configuration می‌ شود.
- **مفهوم Transient Hostname :** نامی که Runtime و معمولاً توسط سرویس‌ های مدیریتی شبکه یا مکانیزم‌ های دیگر تعیین می‌ شود.
- **مفهوم Pretty Hostname :** نام قابل نمایش و آزاد تر برای معرفی سیستم.
- **مفهوم Machine ID :** شناسه‌ ای که سیستم برای شناسایی Instance سیستم‌ عامل استفاده می‌ کند.
- **مفهوم Boot ID :** شناسه مربوط به Boot جاری.
- **مفهوم Operating System Information :** اطلاعاتی که معمولاً از /etc/os-release قابل مشاهده است.

#### ⚙️ hostnamectl Options

- ##### hostnamectl status

اطلاعاتی مثل hostname ، سیستم‌ عامل ، Kernel ، معماری و Virtualization رو نمایش می‌ دهد.
```bash
:~$ hostnamectl status
Static hostname: ASUS
       Icon name: computer-laptop
         Chassis: laptop 💻
      Machine ID: 46eda155dc23417aa5e3feaa19329sce
         Boot ID: 48f9909c43fc49ed9b903c17daopuj28
Operating System: Ubuntu 24.04.5 LTS                  
          Kernel: Linux 7.0.0-38-generic
    Architecture: x86-64
 Hardware Vendor: ASUS
  Hardware Model: ASUS vivobook Laptop 15-Sca0xxx
Firmware Version: F.29
   Firmware Date: Tue 2025-04-22
    Firmware Age: 1y 5month 2w 1d
```

- ##### hostnamectl set-hostname

تغییر hostname سیستم مثلاً به web01.
```bash
:~$ sudo hostnamectl set-hostname web01
```

- ##### hostnamectl set-chassis

مشخص کردن نوع سیستم که به systemd می‌ گوید این سیستم از نوع server است.
```bash
:~$ sudo hostnamectl set-chassis server
```
مقادیر دیگر :
```bash
desktop
laptop
server
vm
container
```

- ##### hostnamectl set-deployment

مشخص می‌ کند سیستم در محیط production قرار دارد.
```bash
:~$ sudo hostnamectl set-deployment production
```
مثلاً :
```bash
production
development
testing
```

- ##### hostnamectl set-location 

موقعیت سیستم را تنظیم می‌ کند.
```bash
:~$ sudo hostnamectl set-location "Tehran"
```

- ##### hostnamectl --static

نمایش Static Hostname که web01 همان hostname اصلی است که معمولاً در /etc/hostname ذخیره می‌ شود.
```bash
:~$ hostnamectl --static
web01
```

- ##### hostnamectl --pretty

نمایش Pretty Hostname که اگر قبلاً تنظیم شده باشه Production Web Server این hostname می‌ تواند شامل فاصله و حروف خوانا تر باشه.
```bash
:~$ hostnamectl --pretty
Production Web Server
```

- ##### hostnamectl --transient

نمایش Transient Hostname که hostname موقتی سیستم را نمایش می‌ دهد این مقدار ممکنه توسط شبکه یا سرویس‌های دیگر تعیین شود و لزوماً در /etc/hostname ذخیره نشده باشد.
```bash
:~$ hostnamectl --transient
```

- ##### hostnamectl --json

خروجی JSON که اطلاعات رو به شکل JSON برمی‌ گرداند و برای اسکریپت‌ نویسی و پردازش خودکار خیلی کاربردی است.

```bash
:~$ hostnamectl --json=pretty status
```
مثلاً :
```JSON
{
  "Hostname": "web01",
  "OperatingSystem": "Ubuntu 24.04 LTS",
  "Kernel": "Linux 6.8.0",
  "Architecture": "x86-64"
}
```

> 💡 در نسخه‌ های مختلف systemd ممکنه بعضی subcommand ها یا option ها کمی متفاوت باشند. برای دیدن option های دقیق روی همان سیستم ، بهترین مرجع man hostnamectl و hostnamectl --help هستند.
#### ✅ Linux Administration

دستور hostnamectl برای مدیریت یکپارچه Hostname در سیستم‌ های مبتنی بر systemd بسیار کاربردی است و معمولاً انتخاب مناسبی نسبت به ویرایش دستی چند فایل مختلف است.

---

### 🔧 arch commmand

دستور arch نوع Machine Architecture را نمایش می‌ دهد.
```bash
:~$ arch
x86_64
```
این مقدار نشان می‌ دهد User Space در یک Architecture از نوع x86_64 اجرا می‌ شود. نکته مهم این است که Architecture گزارش‌ شده لزوماً به این معنا نیست که هر نرم‌ افزار نصب‌ شده روی سیستم حتماً با همان Architecture ساخته شده است بلکه سیستم می‌ تواند در شرایط خاص قابلیت اجرای Binary های Architecture های دیگر را نیز داشته باشد.

#### 🧰 Under the Hood

وقتی این Command را اجرا می‌ کنیم ، از لحظه اجرا تا نمایش Output دقیقاً چه اتفاقی می‌ افتد ؟

هنگام اجرای دستور arch، ابتدا Shell باینری /usr/bin/arch را در User Space اجرا می‌ کند. این ابزار از نظر عملکرد مشابه uname -m بوده و برای دریافت معماری سیستم ، فراخوانی سیستمی ()uname را انجام می‌ دهد. در سمت Kernel ، لینوکس مقدار معماری سخت‌افزار، مانند x86_64 یا aarch64 ، را از فیلد machine در ساختار struct new_utsname که تحت uts_namespace قرار دارد ، استخراج می‌ کند. در نهایت ، ابزار arch این مقدار را بدون هیچ متن اضافی در stdout چاپ کرده و با Exit Code 0 خاتمه می‌ یابد.

#### ✅ Linux Administration

این Command در Script های install و Automation برای انتخاب Package یا Binary مناسب کاربرد دارد.

---

### 🔧 lscpu commmand

دستور lscpu اطلاعات مربوط به CPU Architecture و Topology سیستم را نمایش می‌ دهد ، از جمله :

- Number of logical CPUs
- Cores
- Sockets
- Threads
- NUMA
- Cache
- CPU Model
- CPU Flags
- CPU Architecture

```bash
:~$ lscpu
Architecture:        x86_64
CPU op-mode(s):      32-bit, 64-bit
Byte Order:          Little Endian
CPU(s):              6
On-line CPU(s) list: 154
Thread(s) per core:  2
Core(s) per socket:  8
Socket(s):           1
NUMA node(s):        1
Vendor ID:           GenuineIntel
Model name:          12th Gen Intel(R) Core(TM) i5-12450H
L1d cache:           32K
L1i cache:           32K
L2 cache:            256K
L3 cache:            3072K
Flags:               fpu vme de pse tsc msr pae mce cx8 apic sep mtrr pge mca...
```

#### 🧰 Under the Hood

وقتی این Command را اجرا می‌ کنیم ، از لحظه اجرا تا نمایش Output دقیقاً چه اتفاقی می‌ افتد ؟

هنگام اجرای دستور lscpu ، ابتدا ابزار /usr/bin/lscpu که بخشی از بسته util-linux است ، در User Space اجرا می‌ شود. این ابزار برای جمع‌ آوری اطلاعات پردازنده ، با استفاده از عملیات عادی I/O فایل ، دو فایل‌ سیستم مجازی procfs و sysfs را می‌ خواند. از طریق /proc/cpuinfo اطلاعات عمومی پردازنده  مانند Vendor ID و CPU Family و Model و Stepping و CPU MHz و Flags استخراج می‌ شود و از طریق /sys/devices/system/cpu/ اطلاعات مربوط به توپولوژی پیشرفته CPU از جمله تعداد core ها ، socket ها ، threads ها و ساختار حافظه Cache از مسیرهایی مانند /sys/devices/system/cpu/cpuN/topology/ و /*sys/devices/system/cpu/cpuN/cache/index/ خوانده می‌ شود. داده‌ های موجود در این فایل‌ های مجازی در اصل بر پایه اطلاعاتی هستند که هسته لینوکس از جداول سخت‌افزاری CPUID پردازنده و جداول DMI/SMBIOS مربوط به Firmware سیستم در زمان بوت استخراج کرده و در زیرسیستم‌ های مربوط به مدیریت CPU نگهداری می‌ کند. در نهایت ، lscpu این اطلاعات پراکنده را parse و پردازش کرده و توپولوژی پردازنده را محاسبه می‌ کند و نتیجه را در قالب یک جدول خوانا در خروجی نمایش می‌ دهد.

#### ⚙️ lscpu Options

- ##### lscpu -e

اطلاعات CPU ها را به‌ صورت جدولی و خوانا نشان می‌ دهد.
```bash
:~$ lscpu -e
CPU NODE SOCKET CORE L1d:L1i:L2:L3 ONLINE    MAXMHZ   MINMHZ       MHZ
  0    0      0    0 0:0:0:0          yes 4400.0000 400.0000  400.0000
  1    0      0    0 0:0:0:0          yes 4400.0000 400.0000  400.0000
  2    0      0    1 4:4:1:0          yes 4400.0000 400.0000  400.0000
  3    0      0    1 4:4:1:0          yes 4400.0000 400.0000 1169.6121
  4    0      0    2 8:8:2:0          yes 4400.0000 400.0000 1276.9930
  5    0      0    2 8:8:2:0          yes 4400.0000 400.0000  400.0000
  6    0      0    3 12:12:3:0        yes 4400.0000 400.0000 1400.0000
  7    0      0    3 12:12:3:0        yes 4400.0000 400.0000  957.1790
  8    0      0    4 20:20:5:0        yes 3300.0000 400.0000 1135.9080
  9    0      0    5 21:21:5:0        yes 3300.0000 400.0000 1399.9860
 10    0      0    6 22:22:5:0        yes 3300.0000 400.0000 1342.8090
 11    0      0    7 23:23:5:0        yes 3300.0000 400.0000  400.0000
```

- ##### lscpu -p

خروجی را در قالبی می‌ دهد که برای اسکریپت‌ نویسی و پردازش توسط ابزارها مناسب‌ تر باشد.

```bash
:~$ lscpu -p
# The following is the parsable format, which can be fed to other
# programs. Each different item in every column has an unique ID
# starting usually from zero.
# CPU,Core,Socket,Node,,L1d,L1i,L2,L3
0,0,0,0,,0,0,0,0
1,0,0,0,,0,0,0,0
2,1,0,0,,4,4,1,0
3,1,0,0,,4,4,1,0
4,2,0,0,,8,8,2,0
5,2,0,0,,8,8,2,0
6,3,0,0,,12,12,3,0
7,3,0,0,,12,12,3,0
8,4,0,0,,20,20,5,0
9,5,0,0,,21,21,5,0
10,6,0,0,,22,22,5,0
11,7,0,0,,23,23,5,0
```

- ##### lscpu --online

فقط CPU هایی را نشان می‌ دهد که در حال حاضر Online / فعال هستند.

```bash
:~$ lscpu --online
CPU(s) online:                       0-7

CPU 0  → online
CPU 1  → online
CPU 2  → online
CPU 3  → online
CPU 4  → online
CPU 5  → online
CPU 6  → online
CPU 7  → online
```

- ##### lscpu --offline

فقط CPU هایی را نشان می‌ دهد که در حال حاضر Offline / فعال هستند.

```bash
:~$ lscpu --online
CPU(s) offline:                      4-5

CPU 0 → Online
CPU 1 → Online
CPU 2 → Online
CPU 3 → Online
CPU 4 → Offline
CPU 5 → Offline
CPU 6 → Online
CPU 7 → Online
```

#### ✅ Linux Administration

دستور lscpu برای بررسی موارد زیر بسیار کاربردی است :
- CPU Capacity
- CPU Topology
- NUMA
- CPU Affinity Planning
- Virtualization Capability
- Performance Troubleshooting

در سیستم‌ های Multi-Socket و NUMA ، اطلاعات Topology می‌ تواند برای تصمیم‌ گیری درباره CPU Affinity و Memory Locality اهمیت داشته باشد.

---

### 🔧 lsmem commmand

دستور lsmem اطلاعات مربوط به Physical Memory و Memory Block های Kernel را نمایش می‌ دهد ، این اطلاعات می‌ تواند شامل Memory Block ها و Range آدرس‌ ها و اندازه Block ها و Online / Offline بودن Memory و Summary حافظه باشد.
```bash
:~$ lsmem
RANGE                                  SIZE  STATE REMOVABLE  BLOCK
0x0000000000000000-0x0000000077ffffff  1.9G online       yes   0-14
0x0000000100000000-0x000000047fffffff   14G online       yes 32-143

Memory block size:       128M
Total online memory:    15.9G
Total offline memory:      0B
```

#### 🧰 Under the Hood

وقتی این Command را اجرا می‌ کنیم ، از لحظه اجرا تا نمایش Output دقیقاً چه اتفاقی می‌ افتد ؟

هنگام اجرای دستور lsmem ، ابتدا ابزار /usr/bin/lsmem که بخشی از بسته util-linux است ، در User Space اجرا می‌ شود. این ابزار برای جمع‌ آوری اطلاعات مربوط به حافظه فیزیکی ، فایل‌ سیستم مجازی sysfs را در مسیر /sys/devices/system/memory/ پیمایش می‌ کند. در این مسیر، هر دایرکتوری memoryN نماینده یک Memory Block از حافظه فیزیکی سیستم است و lsmem با خواندن فایل‌ هایی مانند state که وضعیت بلاک را به‌ صورت online یا offline مشخص می‌ کنند ، اطلاعات مورد نیاز را دریافت می‌ کند. این داده‌ ها توسط Memory Hotplug Subsystem هسته لینوکس مدیریت و از طریق رابط sysfs در اختیار فضای کاربر قرار می‌ گیرند. در نهایت ، lsmem بلاک‌ های حافظه متوالی با وضعیت یکسان را شناسایی و با یکدیگر ترکیب می‌ کند و سپس بازه‌ های آدرس‌ دهی فیزیکی RAM ، اندازه حافظه و وضعیت آنلاین یا آفلاین بودن آن را در قالب خروجی نهایی نمایش می‌ دهد.

#### ⚙️ lsmem Options

- ##### lsmem -e

این گزینه در انتهای خروجی ، یک خلاصه (Summary) از وضعیت حافظه ارائه می‌ دهد.
```bash
:~$ lsmem --summary
Memory block size:       128M
Total online memory:    15.9G
Total offline memory:      0B
```

- ##### lsmem -e

به‌ صورت پیش‌ فرض ، lsmem معمولاً Memory Block ها را به شکل Range های متوالی نمایش می‌ دهد.
```bash
:~$ lsmem --all
RANGE                                  SIZE  STATE REMOVABLE BLOCK
0x0000000000000000-0x0000000007ffffff  128M online       yes     0
0x0000000008000000-0x000000000fffffff  128M online       yes     1
...
0x0000000478000000-0x000000047fffffff  128M online       yes   143

Memory block size:       128M
Total online memory:    15.9G
Total offline memory:      0B
```

#### ✅ Linux Administration

دستور lsmem مخصوصاً در سیستم‌ هایی که قابلیت‌ هایی مانند Memory Hotplug دارند مفید است. برای مثال ، Administrator می‌ تواند بررسی کند Memory Block های جدید توسط Kernel شناسایی و Online شده‌اند یا خیر.

---

### 🔧 free commmand

دستور free وضعیت Memory سیستم و Swap را نمایش می‌ دهد.
```bash
:~$ free
               total        used        free      shared  buff/cache   available
Mem:        16026356     5110452     6307308      149824     5084212    10915904
Swap:       15625212           0    15625212
```

##### 🔻 total
کل Memory قابل مشاهده در MemTotal.

##### 🔻 free
به Memory موجود در MemFree که در حال حاضر به‌ عنوان حافظه آزاد گزارش می‌ شود.

##### 🔻 used
در خروجی معمول free ، این مقدار بر اساس تعریف ابزار محاسبه می‌ شود و نباید آن را صرفاً Memory مصرف‌ شده توسط Process ها در نظر گرفت.

##### 🔻 shared
در خروجی‌ های جدید free معمولاً با اطلاعاتی مانند Shmem مرتبط است و می‌ تواند شامل Memory مورد استفاده توسط tmpfs باشد.

##### 🔻 buff/cache
ترکیبی از Buffer و Cache های مربوط به Kernel ، از جمله Page Cache است.

##### 🔻 available
همچنین MemAvailable تخمینی از Memory است که بدون ایجاد Memory Pressure شدید می‌ تواند برای Application های جدید در دسترس قرار گیرد بنابراین free متضاد available است و available معمولاً برای ارزیابی وضعیت Memory از free اطلاعات مفید تری ارائه می‌ کند اما MemAvailable یک estimation است و نباید آن را با مقدار واقعی RAM آزاد یکی دانست.

#### 🧰 Under the Hood

وقتی این Command را اجرا می‌ کنیم ، از لحظه اجرا تا نمایش Output دقیقاً چه اتفاقی می‌ افتد ؟

هنگام اجرای دستور free ، ابتدا Shell ابزار /usr/bin/free را که بخشی از بسته procps است ، در User Space اجرا می‌ کند. سپس free با استفاده از System Call هایی مانند()open و()read ، شبه‌ فایل /proc/meminfo را باز کرده و اطلاعات آن را می‌ خواند. این فایل توسط Memory Management Subsystem هسته لینوکس تولید می‌ شود و اطلاعاتی از بخش‌ هایی مانند Buddy System ، Page Cache و Slab Allocator را در اختیار فضای کاربر قرار می‌ دهد. هنگام خواندن /proc/meminfo ، هسته مقادیر متغیر های مختلفی مانند MemTotal برای کل RAM فیزیکی ، MemFree برای حافظه آزاد ، MemAvailable برای حافظه قابل تخصیص ، Buffers برای بافر های مربوط به I/O ، Cached برای Page Cache ، Shmem برای حافظه مورد استفاده tmpfs و Slab و SReclaimable برای حافظه اختصاص‌ یافته به ساختار های داخلی Kernel را محاسبه کرده و به‌ صورت متن در اختیار free قرار می‌ دهد. در مرحله بعد free این مقادیر متنی در User Space را parse کرده و بر اساس آن مقادیر نهایی را محاسبه می‌ کند برای مثال buff/cache از مجموع Buffers + Cached + SReclaimable و used از رابطه total - free - buff/cache به دست می‌ آید. در نهایت ، free مقادیر محاسبه‌ شده را بر اساس Option انتخاب‌شده ، مانند h- برای نمایش خوانا یا m- برای نمایش برحسب مگابایت، مقیاس‌ دهی کرده و نتیجه را در قالب جدول در خروجی نمایش می‌ دهد.

#### ⚙️ free Options

- ##### free -h

منظور از h- یعنی human-readable و مقادیر را با واحد های قابل‌ خواندن مثل Gi, Mi, Ki نمایش می‌ دهد.
```bash
:~$ free -h
               total        used        free      shared  buff/cache   available
Mem:            15Gi       4.9Gi       6.0Gi       147Mi       4.9Gi        10Gi
Swap:           14Gi          0B        14Gi
```

- ##### free -m

منظور از -m یعنی مقادیر را بر اساس Megabytes نمایش بدهد.
```bash
:~$ free -m
               total        used        free      shared  buff/cache   available
Mem:           15650        5126        6012         146        4976       10524
Swap:          15258           0       15258
```

- ##### free -g

منظور از -g یعنی مقادیر را بر اساس Gigabytes نمایش بدهد.
```bash
:~$ free -g
               total        used        free      shared  buff/cache   available
Mem:              15           4           5           0           4          10
Swap:             14           0          14
```

- ##### free -s 2

اینجا s- یعنی seconds و عدد 2 یعنی خروجی را هر ۲ ثانیه یک بار دوباره نمایش بدهد.
```bash
:~$ free -s 2
               total        used        free      shared  buff/cache   available
Mem:        16026356     5185924     6218256      150448     5096368    10840432
Swap:       15625212           0    15625212

               total        used        free      shared  buff/cache   available
Mem:        16026356     5182460     6221720      150392     5096312    10843896
Swap:       15625212           0    15625212

               total        used        free      shared  buff/cache   available
Mem:        16026356     5182400     6221720      150392     5096384    10843956
Swap:       15625212           0    15625212


               total        used        free      shared  buff/cache   available
Mem:        16026356     5182148     6221972      150392     5096392    10844208
Swap:       15625212           0    15625212
```

- ##### free -t

منظور از -t یعنی Total که یک ردیف اضافه به نام Total نمایش می‌ دهد که RAM + Swap را با هم نشان می‌ دهد.
```bash
:~$ free -t
               total        used        free      shared  buff/cache   available
Mem:        16026356     5192684     6209724      150400     5098100    10833672
Swap:       15625212           0    15625212
Total:      31651568     5192684    21834936
```

#### ✅ Linux Administration

دستور free یکی از اولین ابزارها برای بررسی Memory Pressure و Swap Usage و RAM Capacity و احتمال OOM و وضعیت کلی Memory است.

---

### 🔧 uptime commmand

دستورuptime اطلاعاتی مانند Current Time و Uptime و تعداد User های Login شده و Load Average را نمایش می‌ دهد.
```bash
:~$ uptime 
 17:20:43 up  7:23,  1 user,  load average: 0.25, 0.27, 0.45
```

در لینوکس Load Average فقط به CPU Utilization محدود نیست. به‌ طور کلی ، Load Average میانگینی از تعداد  Taskهایی است که در وضعیت‌ های مرتبط با اجرای CPU و برخی حالت‌ های غیرقابل وقفه مانند انتظار برای I/O  قرار دارند.

Load Average :
1 minute  → 0.25
5 minutes → 0.27
15 minutes → 0.45

مقایسه Load Average با تعداد CPU های logical می‌ تواند یک نشانه اولیه برای بررسی وضعیت سیستم باشد ، اما به‌ تنهایی Diagnosis محسوب نمی‌ شود. برای مثال ، Load Average بالاتر از تعداد CPU های منطقی می‌ تواند نشان‌ دهنده وجود تعداد زیادی Task در حال رقابت برای CPU یا Task هایی در وضعیت‌ های مرتبط با I/O باشد. برای مشخص کردن Root Cause باید ابزارهای دیگری مانند top و vmstat و iostat و pidstat و ps نیز بررسی شوند.

#### 🧰 Under the Hood

وقتی این Command را اجرا می‌ کنیم ، از لحظه اجرا تا نمایش Output دقیقاً چه اتفاقی می‌ افتد ؟

هنگام اجرای دستور uptime ، ابتدا Shell ابزار /usr/bin/uptime را که بخشی از بسته procps است ، در User Space اجرا می‌ کند. این ابزار برای دریافت اطلاعات مورد نیاز، بسته به پیاده‌ سازی، می‌ تواند از فراخوانی سیستمی()sysinfo استفاده کند یا اطلاعات را مستقیماً از شبه‌ فایل‌ های /proc/uptime و /proc/loadavg بخواند. برای محاسبه تعداد کاربران وارد شده به سیستم نیز فایل /var/run/utmp را باز کرده و رکورد های موجود در آن را پردازش می‌ کند. مقدار System Uptime توسط زیرسیستم Kernel Timekeeping و بر اساس زمان سپری‌ شده از لحظه بوت سیستم نگهداری شده و از طریق /proc/uptime در اختیار فضای کاربر قرار می‌ گیرد. مقدار Load Average نیز توسط Scheduler در kernel محاسبه می‌ شود و میانگین بار سیستم در بازه‌ های ۱ ، ۵ و ۱۵ دقیقه‌ ای را در /proc/loadavg ارائه می‌ کند. همچنین اطلاعات مربوط به کاربران وارد شده از رکورد های USER_PROCESS موجود در /var/run/utmp استخراج می‌ شود. در نهایت ، uptime زمان جاری ، مدت زمان سپری‌ شده از بوت ، تعداد کاربران وارد شده و سه مقدار Load Average را در کنار یکد یگر قرار داده و خروجی نهایی را در قالب یک خط نمایش می‌ دهد.

#### ⚙️ uptime Options

- ##### uptime -p

مدت زمان روشن بودن سیستم را به شکل خوانا نمایش می‌ دهد و  کاربرد آن برای زمانی که می خواهید بدانید که سیستم چند وقت است Restart نشده است.
```bash
:~$ uptime -p
up 7 hours, 38 minutes
```

- ##### uptime -s

به‌ جای اینکه بگوید سیستم چقدر روشن است ، می‌ گوید سیستم از چه تاریخی و چه ساعتی روشن شده است.
```bash
:~$ uptime -s
2026-10-07 09:57:17
```

#### ✅ Linux Administration

دستور uptime برای بررسی سریع وضعیت کلی سیستم و تشخیص اولیه مشکلات Performance بسیار مفید است ، اما نباید Load Average را به‌ تنهایی معادل CPU Saturation یا I/O Wait در نظر گرفت.

---

### 🔧 date commmand

دستور date برای نمایش و در برخی موارد تنظیم System Time و همچنین Format کردن  Timestampها استفاده می‌ شود.
```bash
:~$ date
Wed Oct  7 05:45:28 PM +0330 2026

Wed        → Day of the week
Oct        → Month
7          → Day of the month
05:45:28   → Time
+0330      → Timezone
2026       → Year
```

##### 🔻 Timezone
نمایش زمان محلی به Timezone Configuration سیستم وابسته است. در بسیاری از سیستم‌ها مسیر /etc/localtime به Timezone Database مربوطه اشاره می‌ کند.

##### 🔻RTC
مفاهیم System Clock و Hardware Clock یا RTC دو مفهوم متفاوت هستند. date عمدتاً با System Clock سروکار دارد. مدیریت و Synchronization بین System Clock و RTC می‌ تواند توسط ابزارهایی مانند hwclock و سرویس‌ های Time Synchronization انجام شود.

#### 🧰 Under the Hood

وقتی این Command را اجرا می‌ کنیم ، از لحظه اجرا تا نمایش Output دقیقاً چه اتفاقی می‌ افتد ؟

هنگام اجرای دستور date ، ابتدا Shell باینری /bin/date را در User Space اجرا می‌ کند. این ابزار برای دریافت زمان فعلی ، از رابط clock_gettime(CLOCK_REALTIME) استفاده می‌ کند و در صورت داشتن دسترسی لازم ، برای تنظیم ساعت سیستم از ()clock_settime بهره می‌ گیرد. منبع اصلی زمان ، Timekeeping Subsystem هسته لینوکس است که زمان سیستم را با استفاده از منابعی مانند ساعت سخت‌افزاری RTC (Real-Time Clock) و منابع زمان‌ سنج داخلی مدیریت می‌ کند. برای تبدیل زمان سیستم به زمان محلی ، date فایل /etc/localtime را بررسی می‌ کند که معمولاً به یکی از فایل‌ های مربوط به منطقه زمانی در /usr/share/zoneinfo/ اشاره دارد. سپس مقدار زمان دریافت‌ شده از Kernel را در فضای کاربر و از طریق توابع glibc مانند ()localtime_r به اجزای قابل‌ خواندن شامل سال ،  ماه ، روز، ساعت ، دقیقه و ثانیه تبدیل می‌ کند. در نهایت، date این اطلاعات را بر اساس الگوی تعیین‌ شده توسط کاربر، مانند +FORMAT قالب‌ بندی کرده و نتیجه را در خروجی استاندارد نمایش می‌ دهد.

#### ⚙️ date Options and Formats

- ##### date -u

به‌ جای Timezone محلی ، زمان را بر اساس UTC نمایش می‌ دهد.
```bash
:~$ date -u
Wed Oct  7 02:23:53 PM UTC 2026
```

- ##### date --date

گزینه‌ ی date-- یا به شکل کوتاه‌ ترd- اجازه می‌دهد یک تاریخ / زمان مشخص را به date بدهیم و آن را پردازش کنیم.
```bash
:~$ date --date="tomorrow"
Thu Oct  8 05:56:18 PM +0330 2026

:~$ date --date="yesterday"
Sun Oct  4 12:30:45 +0330 2026

:~$ date --date="2026-10-05 + 10 days"
Thu Oct 15 00:00:00 +0330 2026
```

- ##### date +FORMAT

علامت + به date می‌ گوید خروجی را با این فرمت مشخص بدهد.
```bash
:~$ date +%Y-%m-%d
2026-10-05

date +%Y → Year
2026

date +%m → Month
10

date +%d → Day of the month
7

date +%H → Hour
18

date +%M → Minute
12

date +%S → Seconde
32

date '+%Y-%m-%d %H:%M:%S' → Date & time
2026-10-05 12:30:45
```

- ##### date +%s

این format خیلی مهم است ، %s یعنی تعداد ثانیه‌ های گذشته از Unix Epoch برابر است با :
```bash
1970-01-01 00:00:00 UTC 
```
به عبارتی Unix Timestamp که این مقدار را در برنامه‌ نویسی ، اسکریپت‌ها ، لاگ‌ ها و سیستم‌ های مختلف زیاد می‌ بینید.

```bash
:~$ date +%s
1791196245
```

- ##### date -s

گزینه‌ ی s- یعنی set ، تاریخ و ساعت سیستم را تغییر بدهد.
```bash
:~$ sudo date -s "2026-10-05 12:00:00"
Mon Oct  5 12:00:00 +0330 2026
```

#### ✅ Linux Administration

دستور date در مواردی مانند Timestamp در Script ها و Log Analysis و Backup Naming و بررسی اختلاف زمانی همچنین Troubleshooting Time-related Issues کاربرد دارد. برای سیستم‌ های Distributed ، اختلاف زمان بین Server ها می‌ تواند باعث مشکلاتی در Log Correlation ، Authentication و Application های حساس به زمان شود.

---

### 🔧 cal commmand


