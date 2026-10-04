## System Information

#### 📄 Command list :

- uname 
- hostname
- hostnamectl
- arch
- lscpu
- lsmem
- free
- uptime
- date
- cal
- whoami
- id

---

### 🔧 Unix Name command (uname)


**کار اصلی :** دستور uname یکی از command های پایه در سیستم‌ های Unix و Linux است. کار اصلی آن نمایش اطلاعات مربوط به Kernel و معماری سخت‌ افزار و Hostname سیستم است. برخلاف بعضی ابزارها که اطلاعات را از config file های موجود روی Disk می‌ خوانند ، uname اطلاعات را مستقیماً از Kernel در حال اجرا می‌ گیرد و برای این کار از System Call به نام ()uname استفاده می‌ کند.

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

- **کار اصلی :** برای دیدن یا تنظیم Hostname سیستم استفاده می‌ شود یعنی همان نامی که سیستم در شبکه با آن شناخته می‌ شود.
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

- **کار اصلی :** یک ابزار مربوط به سیستم‌ های مبتنی بر Systemd است که برای مشاهده و مدیریت Hostname و بعضی اطلاعات هویتی سیستم استفاده می‌ شود.
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

- **کار اصلی :** معماری سیستم را نمایش می‌ دهد :
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

- **کار اصلی :** اطلاعات کامل و مرتب‌ شده‌ ای درباره CPU و معماری آن نمایش می‌ دهد. اطلاعات را از منابعی مثل ```proc/cpuinfo/``` و ```sysfs``` می‌ گیرد. اطلاعاتی که می‌ توان از خروجی گرفت : 

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






