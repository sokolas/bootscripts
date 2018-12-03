# bootscripts
UEFI Linux boot scripts and instructions

Источник: https://github.com/slytomcat/UEFI-Boot/wiki
### Объединение ESP и boot в один раздел. ###
Вместо того, чтобы руками копировать/удалять ядра и initrd, можно заставить систему делать это автоматически. Для начала перенесем /boot на ESP-раздел. 
Эта идея - часть проекта UEFI-Boot.
> Есть один недостаток хранения ядер на ESP - там используется FAT, в которой не поддерживаются симлинки. Если при переустановке (re-installation) ядра симлинк не создается, нужно удалить и снова установить ядро.

## UEFI-Boot solution ##
> UEFI-Boot в основном предназначено для Ubuntu. Для других дистрибутивов, возможно, понадобится поменять какие-то дистрибутивозависимые настройки.

Сначала перенесем ядра и все, что для них нужно, на ESP-раздел.

`# mv /boot/*-generic* /boot/efi/`

Отмонтируем ESP.

`# umount /boot/efi`

ОЧистим /boot - теперь это будет просто директория для монтирования ESP. 

`# rm -rf /boot/*`

Затем необходимо поменять точку монтирования ESP в /etc/fstab

`# sed -i 's/\/boot\/efi/\/boot/' /etc/fstab`

Наконец, смонтируем ESP в /boot

`# mount /boot`

С этого момента все апдейты ядра и initrd будут проводиться прямо на ESP, а UEFI сможет загружать их напрямую.

### Установка EFI boot manager
(Взято с https://wiki.archlinux.org/index.php/systemd-boot)

Чтобы использовать EFI boot manager из systemd-boot, сначала нужно проверить, что система использует UEFI и UEFI-переменные доступны. Это можно сделать командой `efivar --list.`

Нужно заметить, что systemd-boot может загружать только ядра, собранные с EFISTUB, и только с ESP раздела. Чтобы обновлять ядра автоматически, рекомендуется смонтировать ESP в /boot. Если это не так, то ядра и initramfs нужно копировать на ESP. (See https://wiki.archlinux.org/index.php/EFI_system_partition#Alternative_mount_points for details.)

Точка монтирования ESP раздела далее будет обозначена как *esp*, в нашем случае это `/boot`.

Используйте bootctl(1), чтобы установить загрузчик systemd-boot на ESP:

`# bootctl --path=esp install`

Эта команда скопирует загрузчик systemd-boot на раздел EFI: на x64 два идентичных бинарника *esp*/EFI/systemd/systemd-bootx64.efi и *esp*/EFI/BOOT/BOOTX64.EFI будут записаны на ESP. Затем утилита установит загрузчик systemd-boot как опцию загрузки по умолчанию (default boot entry) 

### Начальная конфигурация systemd-boot
Настройка производится через /boot/loader. В файле /boot/loader/loader.conf содержатся общие настройки (таймаут и т.п.), а сами пункты меню загрузки - в файлах в /boot/loader/entries. Подробнее про настройку можно почитать тут https://wiki.archlinux.org/index.php/systemd-boot#Loader_configuration

### Скрипт обновления
(взято с https://help.ubuntu.ru/wiki/uefiboot)

Собственно скрипт утилиты располагается в `/usr/bin/uefiboot-update`

Код достаточно примитивный: после определения необходимых параметров, сначала мы вычищаем все пункты меню с названиями вида «Ubuntu….», а затем добавляем в порядке возрастания версии ядра новые пункты меню. Все это делается через создание/удаление файлов в /boot/loader/entries.

В скрипте предусмотрена инициализация переменных настройки ROOT, OPTINS и ROOTFLAGS, которая обеспечивает работу утилиты без дополнительных настроек в большинстве стандартных ситуаций. Но, например, для настройки загрузки системы в режиме SecureBoot нужно будет обязательно указать значение переменной K_SUFFIX в файле `/etc/uefiboot.conf`.

### Инициализация синхронизации пунктов загрузки

Самый простой способ инициализировать синхронизацию — воспользоваться теми же средствами, которыми пользуется grub для обновление своей конфигурации.

Ядро — это довольно важный компонент системы и на его установку/удаление завязано немало процессов. Для того, чтобы упростить процесс взаимодействия с процессом изменения версий ядер были предусмотрены триггеры ядра — это полная аналогия скриптов DEB-пакета: пред/постинсталляция и пред/постудаление, но лежат они в соответствующих папках /etc/kernel,а не в deb-пакете ядра. Таким образом, любое приложение, которому надо прореагировать на появление нового или удаление старого ядра могут сделать символьные ссылки на свои утилиты из соответствующих каталогов или просто разместить там свои скрипты (собственно именно так и поступает grub). Скрипты эти будут автоматически вызываться при наступлении событий связанных установкой или удалением ядер.

Нас будут интересовать каталоги /etc/kernel/postinst.d и /etc/kernel/postrm.d, именно в них мы создадим символьные ссылки на нашу утилиту.

`# ln -s /usr/bin/uefiboot-update /etc/kernel/postinst.d/uefiboot-update`
`# ln -s /usr/bin/uefiboot-update /etc/kernel/postrm.d/uefiboot-update`

Теперь пункты загрузки UEFI будут автоматически обновляться после установки нового ядра или после удаления одного из ранее установленных.
Подчистим систему

Т. к. мы настроили все для загрузки без GRUB/Shim, то необходимо вычистить все, что связано с этими загрузчиками из системы (при обновлении GRUB будет создавать свою запись в пунктах меню UEFI):

`# apt-get purge grub* shim os-prober`

Но оказывается, этого не всегда достаточно. Любое ядро в репозитории Ubuntu имеет grub в зависимостях (зависимость типа рекомендованные — recommends) и вот, в некоторых случаях, grub пытается установится вместе с ядром (причем ставится почему-то grub-pc). Чтобы этого не происходило, нужно apt дать четкую и постоянную инструкцию не ставить рекомендованные.

Сделать это можно, записав значение APT::Install-Recommends "false"; в файл (новый) /etc/apt/apt.conf.d/zz-no-recommends:

`# echo 'APT::Install-Recommends "false";' > /etc/apt/apt.conf.d/zz-no-recommends`

### Финальная настройка

Теперь осталось запустить утилиту обновления пунктов меню загрузки

`# uefiboot-update`
