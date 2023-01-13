#   Загрузка системы 

Задание:

```text
1. Попасть в систему без пароля несколькими способами
2. Установить систему с LVM, после чего переименовать VG
3. Добавить модуль в initrd
```

Выполнение:
1.	Попасть в систему без пароля несколькими способами

Способ 1.

Запустил виртуальную машину и при выборе ядра для загрузки нажал e. Попал в окно где можно изменить параметры загрузки.
В конце строки начинающейся с linux16 добавил init=/bin/sh и нжал сtrl-x для загрузки в систему. Попали в систему. Перемонтировал систему в
режим Read-Write  командой:
```bash
mount -o remount,rw /
```
После чего можно убедился, записав данные в файл или прочитав вывод команды:
```bash
mount | grep root
```

Способ 2. rd.break
В конце строки начинающейся с linux16 добавил rd.break и нажимаем сtrl-x для загрузки в систему
Попал в emergency mode. Наша корневая файловая система смонтирована, опять же в режиме Read-Only, но мы не в ней. Далее будет пример как попасть в нее и поменять пароль администратора:
```bash
mount -o remount,rw /sysroot
chroot /sysroot
passwd root
touch /.autorelabel
```
После чего перезагрузился и зашёл в систему с новым паролем. 

Способ 3. rw init=/sysroot/bin/sh
В строке начинающейся с linux16 заменил ro на rw init=/sysroot/bin/sh и нажал сtrl-x для загрузки в систему
В целом то же самое что и в прошлом примере, но файловая система сразу смонтирована в режим Read-Write.

2.	Установить систему с LVM, после чего переименовать VG

1) Установил CentOS 7 с образа и создал LVM разделы 
2) Первым делом посмотрим текущее состояние системы, командой vgs
3) Переименовал VG командой vgrename centos centos_new
4) Переименовал vg с именем centos на centos_new в файлах
/etc/fstab
/etc/default/grub
/boot/grub2/grub.cfg

5) Собрал новый initramfs.img, чтобы он знал, что изменилось имя VG

mkinitrd -f -v /boot/initramfs-$(uname -r).img $(uname -r)

6) Перезагрузил систему.

3.	Добавить модуль в initrd

Скрипты модулей хранятся в каталоге /usr/lib/dracut/modules.d/. 
Для того чтобы добавить свой модуль создал там папку с именем 01test:
```bash
mkdir /usr/lib/dracut/modules.d/01test
```
И два файла со следующим содержимым:

Скрипт предназначен для установки модуля test.sh
module-setup.sh
```bash
#!/bin/bash

check() {
    return 0
}

depends() {
    return 0
}

install() {
    inst_hook cleanup 00 "${moddir}/test.sh"
}
```

Сам модуль
test.sh
```bash
#!/bin/bash

exec 0<>/dev/console 1<>/dev/console 2<>/dev/console
cat <<'msgend'
Hello! You are in dracut module!
 ___________________
< I'm dracut module >
 -------------------
   \
    \
        .--.
       |o_o |
       |:_/ |
      //   \ \
     (|     | )
    /'\_   _/`\
    \___)=(___/
msgend
sleep 10
echo " continuing...."
```

Пересоздал initrd командой mkinitrd -f -v -a test /boot/initramfs-$(uname -r).img $(uname -r)
Можно использовать так же  dracut -f -v

Вывод команды lsinitrd -m /boot/initramfs-$(uname -r).img

Image: /boot/initramfs-3.10.0-1160.el7.x86_64.img: 21M
========================================================================
Early CPIO image
========================================================================
drwxr-xr-x   3 root     root            0 Jan 13 12:13 .
-rw-r--r--   1 root     root            2 Jan 13 12:13 early_cpio
drwxr-xr-x   3 root     root            0 Jan 13 12:13 kernel
drwxr-xr-x   3 root     root            0 Jan 13 12:13 kernel/x86
drwxr-xr-x   2 root     root            0 Jan 13 12:13 kernel/x86/microcode
-rw-r--r--   1 root     root        19456 Jan 13 12:13 kernel/x86/microcode/GenuineIntel.bin
========================================================================
Version: dracut-033-572.el7

dracut modules:
bash
test
....
Так же вывод каманды lsinitrd -m /boot/initramfs-$(uname -r).img | grep test
[root@localhost ~]#  lsinitrd -m /boot/initramfs-$(uname -r).img | grep test
test


Что говорит о том, что наш кастомный модуль был загружен.

Сделаем reboot

При перезагрузке видим пингвина. 

