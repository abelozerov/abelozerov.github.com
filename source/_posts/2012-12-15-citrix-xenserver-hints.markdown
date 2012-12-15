---
layout: post
title: "Citrix XenServer and XenCenter hints"
date: 2012-12-15 19:44
comments: true
categories: 
- системное администрирование
- xen
- citrix xenserver
- citrix xencenter
---

Настраивал дома Citrix XenServer 6.1 через Citrix XenCenter и столкнулся с парой проблем. Пишу пост для того, чтобы самому второй раз долго не искать решения, и может помогу еще кому-нибудь сэкономить время.

### Как настроить автозапуск виртуальных машин при старте сервера?

Рекомендую сначала объединить виртуальные машины в vApp, чтобы было проще потом рулить автозапуском путем простого добавления или удаления виртуальной машины из соответствующего vApp.

После создания vApp необходимо узнать его UUID. Сделать это можно в консоли XenServer с помощью команды:

	xe appliance-list

Потом редактируете файл `/etc/rc.local`, добавляете в конец:

	sleep 30
	/opt/xensource/bin/xe appliance-start uuid=YOUR_UUID

Все. Теперь после запуска XenServer через 30 секунд стартуют виртуальные машины из вашего vApp.
(величину sleep подобрать опытным путем, если сделаете слишком мало, vApp не будет стартовать)

### Как приаттичить физический жесткий диск к виртуальной машине?

Из XenCenter очень легко присоединить к виртуальной машине removable накопители через меню Storage, к сожалению это не относится к внутренним жестким дискам. Но можно сделать так, чтобы внутренние HDD также оказались в списке removable накопителей :)

Редактируем файл `/etc/udev/rules.d/50-udev.rules`, добавляем строки:

	ACTION=="add", KERNEL=="sdb", SYMLINK+="xapi/block/%k", RUN+="/bin/sh -c '/opt/xensource/libexec/local-device-change %k 2>&1 >/dev/null&'"
	ACTION=="remove", KERNEL=="sdb", RUN+="/bin/sh -c '/opt/xensource/libexec/local-device-change %k 2>&1 >/dev/null&'"

Теперь внутренние HDD отображаются как removable SCSI диски, можно их присоединять к виртуальным машинам.