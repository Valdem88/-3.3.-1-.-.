# Домашнее задание к занятию "3.3. Операционные системы, лекция 1" - Yakovlev V.S.

#### 1. Какой системный вызов делает команда `cd`? В прошлом ДЗ мы выяснили, что `cd` не является самостоятельной  программой, это `shell builtin`, поэтому запустить `strace` непосредственно на `cd` не получится. Тем не менее, вы можете запустить `strace` на `/bin/bash -c 'cd /tmp'`. В этом случае вы увидите полный список системных вызовов, которые делает сам `bash` при старте. Вам нужно найти тот единственный, который относится именно к `cd`. Обратите внимание, что `strace` выдаёт результат своей работы в поток stderr, а не в stdout.
Решение 
```bash
chdir("/tmp") = 0
```
chdir — системная функция (системный вызов). Используется для изменения текущего рабочего каталога.
```bash
[root@Git-SentOS-8 tmp]# strace -o cd_log /bin/bash -c 'cd /tmp' && grep tmp cd_log
execve("/bin/bash", ["/bin/bash", "-c", "cd /tmp"], 0x7fffa2096b70 /* 33 vars */) = 0
stat("/tmp", {st_mode=S_IFDIR|S_ISVTX|0777, st_size=4096, ...}) = 0
stat("/tmp", {st_mode=S_IFDIR|S_ISVTX|0777, st_size=4096, ...}) = 0
stat("/tmp", {st_mode=S_IFDIR|S_ISVTX|0777, st_size=4096, ...}) = 0
chdir("/tmp")                           = 0
```
```bash
chdir("/tmp") = 0
```
#### 2. Попробуйте использовать команду `file` на объекты разных типов на файловой системе. Например:
```bash
    vagrant@netology1:~$ file /dev/tty
    /dev/tty: character special (5/0)
    vagrant@netology1:~$ file /dev/sda
    /dev/sda: block special (8/0)
    vagrant@netology1:~$ file /bin/bash
    /bin/bash: ELF 64-bit LSB shared object, x86-64
```
    Используя `strace` выясните, где находится база данных `file` на основании которой она делает свои догадки.
Решение 
```bash
[root@Git-SentOS-8 ~]# strace file
...
close(3)                                = 0
openat(AT_FDCWD, "/usr/share/misc/magic.mgc", O_RDONLY) = 3
fstat(3, {st_mode=S_IFREG|0644, st_size=5194744, ...}) = 0
mmap(NULL, 5194744, PROT_READ|PROT_WRITE, MAP_PRIVATE, 3, 0) = 0x7eff3a294000
close(3)
...
[root@Git-SentOS-8 ~]# file /usr/share/misc/magic.mgc
/usr/share/misc/magic.mgc: magic binary file for file(1) cmd (version 14) (little endian)
```
Команда `file` обращается к `"/usr/share/misc/magic.mgc"`
#### 3. Предположим, приложение пишет лог в текстовый файл. Этот файл оказался удален (deleted в lsof), однако возможности сигналом сказать приложению переоткрыть файлы или просто перезапустить приложение – нет. Так как приложение продолжает писать в удаленный файл, место на диске постепенно заканчивается. Основываясь на знаниях о перенаправлении потоков предложите способ обнуления открытого удаленного файла (чтобы освободить место на файловой системе).
Решение

Проверить можно таким образом. Команда делает пинг, выводит результат в файл ping_log, далее файл удаляется. С помощью `ps` проверяем процессы.

```bash
[root@Git-SentOS-8 ~]# ping google.com > ping_log & rm ping_log
[3] 7578
rm: удалить обычный файл 'ping_log'? y
[root@Git-SentOS-8 ~]# ps
    PID TTY          TIME CMD
   4510 pts/0    00:00:00 bash
   7578 pts/0    00:00:00 ping
   7580 pts/0    00:00:00 ps
```
Процесс `ping` выполняется. Файл удален, но размер растет
```bash
[root@Git-SentOS-8 ~]# lsof -p 7578 | grep deleted
ping    7578 root    1w   REG  253,0    28729 36196690 /root/ping_log (deleted)
[root@Git-SentOS-8 ~]# lsof -p 7578 | grep deleted
ping    7578 root    1w   REG  253,0    30367 36196690 /root/ping_log (deleted)
[root@Git-SentOS-8 ~]# lsof -p 7578 | grep deleted
ping    7578 root    1w   REG  253,0    31231 36196690 /root/ping_log (deleted)
[root@Git-SentOS-8 ~]# lsof -p 7578 | grep deleted
ping    7578 root    1w   REG  253,0    31748 36196690 /root/ping_log (deleted)
```
зная `pid` процесса можем ограничить размер файла или очистить его.
```bash
[root@Git-SentOS-8 ~]# truncate -s 1MB /proc/7578/fd/1
[root@Git-SentOS-8 ~]# lsof -p 7578 | grep deleted
ping    7578 root    1w   REG  253,0  1000000 36196690 /root/ping_log (deleted)

...
truncate -s 0 /proc/7578/fd/1
```

#### 4. Занимают ли зомби-процессы какие-то ресурсы в ОС (CPU, RAM, IO)?
Решение

Зомби-процессы не занимают память, но могут привести к тому что пользователь под которым они запущены не сможет запустить новые процессы. Зомби-процессы блокируют записи в таблице процессора и при достижении лимита это может стать проблемой и даже привести к невозможности пользователя подключиться к системе. Дисковая подсистема будет также нагружена в зависимости от размера файлов зомби-процесса.
#### 5. В iovisor BCC есть утилита `opensnoop`:
    
```bash
    root@vagrant:~# dpkg -L bpfcc-tools | grep sbin/opensnoop
    /usr/sbin/opensnoop-bpfcc
```
На какие файлы вы увидели вызовы группы `open` за первую секунду работы утилиты? Воспользуйтесь пакетом `bpfcc-tools` для Ubuntu 20.04. Дополнительные [сведения по установке](https://github.com/iovisor/bcc/blob/master/INSTALL.md).

Решение
```bash
root@debian-srv:~# opensnoop-bpfcc -d 1
In file included from <built-in>:2:
In file included from /virtual/include/bcc/bpf.h:12:
In file included from include/linux/types.h:6:
In file included from include/uapi/linux/types.h:14:
In file included from include/uapi/linux/posix_types.h:5:
In file included from include/linux/stddef.h:5:
In file included from include/uapi/linux/stddef.h:2:
In file included from include/linux/compiler_types.h:69:
include/linux/compiler-clang.h:51:9: warning: '__HAVE_BUILTIN_BSWAP32__' macro redefined [-Wmacro-redefined]
#define __HAVE_BUILTIN_BSWAP32__
        ^
<command line>:4:9: note: previous definition is here
#define __HAVE_BUILTIN_BSWAP32__ 1
        ^
In file included from <built-in>:2:
In file included from /virtual/include/bcc/bpf.h:12:
In file included from include/linux/types.h:6:
In file included from include/uapi/linux/types.h:14:
In file included from include/uapi/linux/posix_types.h:5:
In file included from include/linux/stddef.h:5:
In file included from include/uapi/linux/stddef.h:2:
In file included from include/linux/compiler_types.h:69:
include/linux/compiler-clang.h:52:9: warning: '__HAVE_BUILTIN_BSWAP64__' macro redefined [-Wmacro-redefined]
#define __HAVE_BUILTIN_BSWAP64__
        ^
<command line>:5:9: note: previous definition is here
#define __HAVE_BUILTIN_BSWAP64__ 1
        ^
In file included from <built-in>:2:
In file included from /virtual/include/bcc/bpf.h:12:
In file included from include/linux/types.h:6:
In file included from include/uapi/linux/types.h:14:
In file included from include/uapi/linux/posix_types.h:5:
In file included from include/linux/stddef.h:5:
In file included from include/uapi/linux/stddef.h:2:
In file included from include/linux/compiler_types.h:69:
include/linux/compiler-clang.h:53:9: warning: '__HAVE_BUILTIN_BSWAP16__' macro redefined [-Wmacro-redefined]
#define __HAVE_BUILTIN_BSWAP16__
        ^
<command line>:3:9: note: previous definition is here
#define __HAVE_BUILTIN_BSWAP16__ 1
        ^
3 warnings generated.
In file included from <built-in>:2:
In file included from /virtual/include/bcc/bpf.h:12:
In file included from include/linux/types.h:6:
In file included from include/uapi/linux/types.h:14:
In file included from include/uapi/linux/posix_types.h:5:
In file included from include/linux/stddef.h:5:
In file included from include/uapi/linux/stddef.h:2:
In file included from include/linux/compiler_types.h:69:
include/linux/compiler-clang.h:51:9: warning: '__HAVE_BUILTIN_BSWAP32__' macro redefined [-Wmacro-redefined]
#define __HAVE_BUILTIN_BSWAP32__
        ^
<command line>:4:9: note: previous definition is here
#define __HAVE_BUILTIN_BSWAP32__ 1
        ^
In file included from <built-in>:2:
In file included from /virtual/include/bcc/bpf.h:12:
In file included from include/linux/types.h:6:
In file included from include/uapi/linux/types.h:14:
In file included from include/uapi/linux/posix_types.h:5:
In file included from include/linux/stddef.h:5:
In file included from include/uapi/linux/stddef.h:2:
In file included from include/linux/compiler_types.h:69:
include/linux/compiler-clang.h:52:9: warning: '__HAVE_BUILTIN_BSWAP64__' macro redefined [-Wmacro-redefined]
#define __HAVE_BUILTIN_BSWAP64__
        ^
<command line>:5:9: note: previous definition is here
#define __HAVE_BUILTIN_BSWAP64__ 1
        ^
In file included from <built-in>:2:
In file included from /virtual/include/bcc/bpf.h:12:
In file included from include/linux/types.h:6:
In file included from include/uapi/linux/types.h:14:
In file included from include/uapi/linux/posix_types.h:5:
In file included from include/linux/stddef.h:5:
In file included from include/uapi/linux/stddef.h:2:
In file included from include/linux/compiler_types.h:69:
include/linux/compiler-clang.h:53:9: warning: '__HAVE_BUILTIN_BSWAP16__' macro redefined [-Wmacro-redefined]
#define __HAVE_BUILTIN_BSWAP16__
        ^
<command line>:3:9: note: previous definition is here
#define __HAVE_BUILTIN_BSWAP16__ 1
        ^
3 warnings generated.
PID    COMM               FD ERR PATH
1      systemd            24   0 /proc/259/cgroup
root@debian-srv:~#
```

#### 6. Какой системный вызов использует `uname -a`? Приведите цитату из man по этому системному вызову, где описывается альтернативное местоположение в `/proc`, где можно узнать версию ядра и релиз ОС.
Решение
```bash
[root@Git-SentOS-8 ~]# strace uname -a > out.log 2>&1
[root@Git-SentOS-8 ~]# cat out.log | grep uname
execve("/usr/bin/uname", ["uname", "-a"], 0x7ffcbe75a7a8 /* 32 vars */) = 0
uname({sysname="Linux", nodename="Git-SentOS-8.local", ...}) = 0
uname({sysname="Linux", nodename="Git-SentOS-8.local", ...}) = 0
uname({sysname="Linux", nodename="Git-SentOS-8.local", ...}) = 0
[root@Git-SentOS-8 ~]#
```
- это системный вызов
```bash
[root@Git-SentOS-8 ~]# cat /proc/version
Linux version 4.18.0-365.el8.x86_64 (mockbuild@kbuilder.bsys.centos.org) (gcc version 8.5.0 20210514 (Red Hat 8.5.0-10) (GCC)) #1 SMP Thu Feb 10 16:11:23 UTC 2022
```
- Из man 2 uname: Part of the utsname information is also accessible via /proc/sys/kernel/{ostype, hostname, osrelease, version, domainname}.

#### 7. Чем отличается последовательность команд через `;` и через `&&` в bash? Например:
```bash
    root@netology1:~# test -d /tmp/some_dir; echo Hi
    Hi
    root@netology1:~# test -d /tmp/some_dir && echo Hi
    root@netology1:~#
```
    Есть ли смысл использовать в bash `&&`, если применить `set -e`?

Решение

- `;` выполнит все команды последовательно, даже если какая-то завершится ошибкой.
- `&&` остановится при завершении какой-то команды в последовательности ошибкой.
- с параметром -e оболочка завершится только при ненулевом коде возврата команды. 
Если ошибочно завершится одна из команд, разделённых &&, то выхода из шелла не произойдёт. Смысл есть.

#### 8. Из каких опций состоит режим bash `set -euxo pipefail` и почему его хорошо было бы использовать в сценариях?
Решение

- `-e` прерывает выполнение исполнения при ошибке любой команды кроме последней в последовательности; 
- `-x` вывод трейса простых команд;
- `-u` неустановленные/не заданные параметры и переменные считаются как ошибки, с выводом в stderr текста ошибки и выполнит завершение не интерактивного вызова;
- `-o pipefail` возвращает код возврата набора/последовательности команд, ненулевой при последней команды или 0 для успешного выполнения команд.

Повышает детализацию вывода ошибок и завершит сценарий при наличии ошибок, на любом этапе выполнения сценария, кроме последней завершающей команды.

#### 9. Используя `-o stat` для `ps`, определите, какой наиболее часто встречающийся статус у процессов в системе. В `man ps` ознакомьтесь (`/PROCESS STATE CODES`) что значат дополнительные к основной заглавной буквы статуса процессов. Его можно не учитывать при расчете (считать S, Ss или Ssl равнозначными).
Решение
```bash
[root@Git-SentOS-8 ~]# ps -Ao stat  | sort | uniq -c | sort -h
      1 D+
      1 R+
      1 S<
      1 SNs
      1 SNsl
      1 S<sl
      1 Ssl+
      1 STAT
      2 R
      2 SN
      4 I
      4 S+
      8 Sl
     18 Ssl
     22 Sl+
     29 Ss
     44 I<
     51 S
```
Наиболее часто встречаются процессы - S(считать S, Ss или Ssl равнозначными) спящие процессы, находятся в режиме ожидания.
```bash
PROCESS STATE CODES
       Here are the different values that the s, stat and state output specifiers (header "STAT" or "S") will display to
       describe the state of a process:
               D    uninterruptible sleep (usually IO)
               I    Idle kernel thread
               R    running or runnable (on run queue)
               S    interruptible sleep (waiting for an event to complete)
               T    stopped by job control signal
               t    stopped by debugger during the tracing
               W    paging (not valid since the 2.6.xx kernel)
               X    dead (should never be seen)
               Z    defunct ("zombie") process, terminated but not reaped by its parent
```