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




 

---

### 🔧 Unix Name command (uname)


دستور uname یکی از command های پایه در سیستم‌ های Unix و Linux است. کار اصلی آن نمایش اطلاعات مربوط به Kernel و معماری سخت‌ افزار و Hostname سیستم است. برخلاف بعضی ابزارها که اطلاعات را از config file های موجود روی Disk می‌ خوانند ، uname اطلاعات را مستقیماً از Kernel در حال اجرا می‌ گیرد و برای این کار از System Call به نام ()uname استفاده می‌ کند.

**خروجی** : اگر uname را بدون هیچ option اجرا کنید ، فقط نام Kernel را نشان می‌ دهد مثلاً :
```bash
:~$ uname
Linux
```

#### 🛠 Under the Hood

نحوه کارکرد uname در پشت صحنه به این صورت می باشد :

**فراخوانی سیستمی (System Call) :** وقتی uname را در User Space اجرا می‌ کنیم ، برنامه از طریق glibc درخواست خود را به Kernel می‌ فرستد و uname هم System Call را اجرا می‌ کند. در این مرحله پردازنده از User Mode (Ring 3) وارد Kernel Mode (Ring 0) می‌ شود.

**ساختار داده کرنل (struct new_utsname) :** کرنل اطلاعات سیستم را در ساختاری به نام struct new_utsname نگه می‌ دارد. این ساختار در <linux/utsname.h> تعریف شده و بخشی از UTS Namespace است :

```c
struct new_utsname {
    char sysname[65];    /* نام سیستم‌عامل */
    char nodename[65];   /* نام سیستم در شبکه */
    char release[65];    /* نسخه انتشار کرنل */
    char version[65];    /* تاریخ و نسخه ساخت کرنل */
    char machine[65];    /* معماری سخت‌افزار */
    char domainname[65]; /* نام دامنه */
};
```
**ارتباط با proc/  یا (Procfs Interface) :** بخشی از این اطلاعات را می‌ توان از طریق فایل‌ های شبه‌ سیستمی زیر هم مشاهده کرد :
```bash
/proc/sys/kernel/ostype/ ───> معادل sysname
/proc/sys/kernel/hostname/ ───> معادل nodename
/proc/sys/kernel/osrelease/ ───> معادل release
/proc/sys/kernel/version/ ───> معادل version
```

#### 📊 Data Flow Diagram

```
+-------------------------------------------------------------------+
|                     USER SPACE (Ring 3)                           |
| [ uname CLI Utility ]                                             |
|          │                                                        |
|          ▼ (glibc Wrapper)                                        |
| syscall: uname(&amp;buf)                                          |
+------------│------------------------------------------------------+
             │ Switch to Kernel Mode (Ring 0 via sysenter/syscall)
+------------v------------------------------------------------------+
|                   KERNEL SPACE (Ring 0)                           |
| [ sys\_uname() Kernel Function ]                                  |
|          │                                                        |
|          ▼                                                        |
| Reads from active UTS Namespace:                                  |
| current-&gt;nsproxy-&gt;uts\_ns-&gt;name (struct new\_utsname)    |
|          │                                                        |
|          ▼                                                        |
| Copy struct data back to User Space Buffer                        |
+-------------------------------------------------------------------+
```


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


#### 🔩 Practical Examples in Shell Scripting

- نصب خودکار Header های kernel متناظر با نسخه در حال اجرا :
```bash
sudo apt install linux-headers-$(uname -r)
```

- بررسی ۶۴ بیتی بودن معماری سیستم در اسکریپت :
```bash
if [ "$(uname -m)" = "x86\_64" ]; then
    echo "64-bit Architecture detected."
fi
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

برای دیدن یا تنظیم Hostname سیستم استفاده می‌ شود یعنی همان نامی که سیستم در شبکه با آن شناخته می‌ شود.

- **خروجی :** اگر hostname را بدون گزینه اجرا کنید ، Hostname فعلی سیستم را نشان می‌ دهد :
```bash
:~$ hostname
orcanestlab
```

#### ⚙️ hostname Options

- ##### hostname -f

نام کامل سیستم به همراه Domain را نمایش می‌ دهد.
```bash
:~$ hostname -f
orcanestlab
```

- ##### hostname -i

میتواند IP مربوط به Hostname را نمایش می‌ دهد.
```bash
:~$ hostname -i
127.0.1.1
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
:~$ sudo hostname alexAdmin
```

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

```bash
:~$ hostnamectl
 Static hostname: Asus
       Icon name: computer-laptop
         Chassis: laptop 💻
      Machine ID: 46eda155dc23417aa5e3feaa57482910
         Boot ID: db58f01f18f44cf2adae9b1b2847eks1
Operating System: Ubuntu 24.04.5 LTS                  
          Kernel: Linux 7.0.0-38-generic
    Architecture: x86-64
 Hardware Vendor: Asus
  Hardware Model: ASUS Vivobook S14 
Firmware Version: G.25
   Firmware Date: Tue 2025-04-22
    Firmware Age: 1y 5month 1w 6d
```

مثلاً اگر سیستم داخل Virtual Machine باشد ، در قسمت Virtualization ممکن است چیزی مثل ```KVM``` یا ```VirtualBox``` نمایش داده شود.

#### 🪛 Permanently change the hostname

برای تغییر دائمی Hostname از دستور ```sudo hostnamectl set-hostname new_hostname``` استفاده می شود.
```bash
:~$ sudo hostnamectl set-hostname new_hostname
```

#### 🪛 Pretty Hostname

دستور hostnamectl امکان تنظیم یک Pretty Hostname را هم دارد یعنی نامی خوانا و توصیفی که برای نمایش به کاربران مناسب‌ تر است.
```bash
:~$ sudo hostnamectl set-hostname "Your Pretty Name" --pretty
```

---

### 🔧 arch commmand

معماری سیستم را نمایش می‌ دهد :
```bash
:~$ arch 
x86_64
```

- ارتباط با uname: عملکرد arch عملاً همان uname -m است و معمولاً خروجی یکسانی دارند :
```bash
:~$ arch 
x86_64

:~$ uname -m
x86_64
```

--- 

### 🔧 lscpu command

اطلاعات کامل و مرتب‌ شده‌ ای درباره CPU و معماری آن نمایش می‌ دهد. اطلاعات را از منابعی مثل ```proc/cpuinfo/``` و ```sysfs``` می‌ گیرد. اطلاعاتی که می‌ توان از خروجی گرفت : 

```bash
:~$ lscpu 
Architecture:                x86_64
  CPU op-mode(s):            32-bit, 64-bit
  Address sizes:             39 bits physical, 48 bits virtual
  Byte Order:                Little Endian
CPU(s):                      12
  On-line CPU(s) list:       0-11
Vendor ID:                   GenuineIntel
  Model name:                12th Gen Intel(R) Core(TM) i5-12450H
    CPU family:              6
    Model:                   154
    Thread(s) per core:      2
    Core(s) per socket:      8
    Socket(s):               1
    Stepping:                3
    CPU(s) scaling MHz:      15%
    CPU max MHz:             4400.0000
    CPU min MHz:             400.0000
    BogoMIPS:                4992.00
...
```

---

### 🔧 






