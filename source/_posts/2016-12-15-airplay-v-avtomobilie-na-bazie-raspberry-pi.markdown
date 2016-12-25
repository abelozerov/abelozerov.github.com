---
layout: post
title: "AirPlay в автомобиле на базе Raspberry Pi Model B"
date: 2016-12-25 14:49
comments: true
categories:
- AirPlay
- Raspberry Pi
- автомобиль
---

Возникло желание получить возможность слушать Apple Music с телефона на аудиосистеме автомобиля. Для реализации своего желания решил сделать AirPlay устройство на базе Raspberry Pi, которое будет выводить звук в порт AUX

**Приборы и материалы:**

- Raspberry Pi Model B + SD карта
- USB WiFi карточка Edimax EW-7811UN
- Зарядка в прикуриватель Philips DLP2357V/51
- USB - micro USB кабель для зарядки
- 3.5 minijack - 3.5 minijack кабель для звука

{% img /images/2016/12/25/pi.jpg Awesome! %}

**Задача:**

Для работы AirPlay, источник и приемник сигнала должны находиться в одной WiFi сети. Поэтому iPhone должен видеть Raspberry Pi как WiFi точку доступа, при подключении к которой в списке AirPlay устройств в телефоне должно появиться новое устройство. При этом WiFi сеть обязана использоваться исключительно для передачи данных на AirPlay, в интернет телефон должен по-прежнему ходить через мобильную сеть

**Поехали:**

<!-- more -->

- Скачиваем последний [Raspbian Jessie Lite](https://www.raspberrypi.org/downloads/raspbian/), заливаем на SD карту
- У меня нет монитора, мышки и клавиатуры, поэтому всю настройку Raspberry Pi я будут проводить через SSH. Для того, чтобы включить возможность SSH подключения, нужно создать пустой файл с названием `ssh` в корне SD карты. [Источник](https://www.raspberrypi.org/documentation/remote-access/ssh/)

	Hint: первый раз включите Rabpberry Pi без WiFi карты и дождитесь полной загрузки. У меня при установленной WiFi карте перепутались имена сетевых интерфейсов eth0 и wlan0

Подключаю Raspberry Pi через Ethernet провод к домашней сети, и захожу через SSH:

	ssh pi@192.168.x.x

Стандартный пароль `raspberry`

- AirPlay реализую с помощью [Shairport Sync](https://github.com/mikebrady/shairport-sync), это актуальный поддерживаемый форк оригинального Shairport

У него достаточно сложная инструкция по установке, обязательно прочитайте, выложу команды, которые вводил я (включая команду, которой я определял, `systemd` или `System V` используется в моем дистрибутиве Linux):

	sudo apt-get install git
	sudo apt-get install autoconf libtool libdaemon-dev libasound2-dev libpopt-dev libconfig-dev
	sudo apt-get install avahi-daemon libavahi-client-dev libssl-dev
	mkdir ~/airplay
	cd ~/airplay
	git clone https://github.com/mikebrady/shairport-sync.git
	cd shairport-sync
	autoreconf -i -f
	ps aux | grep systemd | grep -v grep
	./configure --sysconfdir=/etc --with-alsa --with-avahi --with-ssl=openssl --with-metadata --with-systemd
	make
	sudo getent group shairport-sync &>/dev/null || sudo groupadd -r shairport-sync >/dev/null
	getent passwd shairport-sync &> /dev/null || sudo useradd -r -M -g shairport-sync -s /usr/bin/nologin -G audio shairport-sync >/dev/null
	sudo make install
	sudo systemctl enable shairport-sync


Поменял настройки в `/etc/shairport-sync.conf` по рекомендации инструкции по установке, для получения лучших результатов от штатной звуковой карточки в Raspberry Pi:

	general
		volume_range_db=30
		drift=352

- Далее настраиваем WiFi карточку

Устанавливаем ISC DHCP Server:

	sudo apt-get install hostapd isc-dhcp-server

Также установим удобный iptables менеджер (во время установки на оба вопроса отвечаем Yes):

	sudo apt-get install iptables-persistent

Далее отредактируем `/etc/dhcp/dhcpd.conf`:

	sudo nano /etc/dhcp/dhcpd.conf

Найдем строчки:

	option domain-name "example.org";
	option domain-name-servers ns1.example.org, ns2.example.org;

И закомментируем их:

	#option domain-name "example.org";
	#option domain-name-servers ns1.example.org, ns2.example.org;

Найдем:

	# If this DHCP server is the official DHCP server for the local
	# network, the authoritative directive should be uncommented.
	#authoritative;

И раскомментируем:

	# If this DHCP server is the official DHCP server for the local
	# network, the authoritative directive should be uncommented.
	authoritative;

Добавляем в конец файла:

	subnet 192.168.42.0 netmask 255.255.255.0 {
		range 192.168.42.10 192.168.42.50;
		option broadcast-address 192.168.42.255;
		option routers 192.168.42.1;
		default-lease-time 600;
		max-lease-time 7200;
	}

Здесь краеугольный камень в том, что мы не добавляем адреса DNS-серверов (например `option domain-name-servers 8.8.8.8, 8.8.4.4`), в результате телефон не будет использовать эту WiFi сеть для доступа в интернет

Редактируем `sudo nano /etc/default/isc-dhcp-server`

	sudo nano /etc/default/isc-dhcp-server

Находим `INTERFACES=""` и меняем на 

	INTERFACES="wlan0"

Далее на время редактирование выключаем `wlan0`

	sudo ifdown wlan0

Редактируем `/etc/network/interfaces`

	sudo nano /etc/network/interfaces

Комментируем все строки, относящиеся к `wlan0`, и добавляем свои:

	allow-hotplug wlan0
	iface wlan0 inet static
		address 192.168.42.1
		netmask 255.255.255.0

Задаем WiFi адаптеру статический IP:

	sudo ifconfig wlan0 192.168.42.1

- Кофигурируем WiFi карточку как Access Point

Редактируем `/etc/hostapd/hostapd.conf`

	sudo nano /etc/hostapd/hostapd.conf

Добавляем:

	interface=wlan0
	ssid=Pi_AP
	country_code=US
	hw_mode=g
	channel=6
	macaddr_acl=0
	auth_algs=1
	ignore_broadcast_ssid=0
	wpa=2
	wpa_passphrase=Raspberry
	wpa_key_mgmt=WPA-PSK
	wpa_pairwise=CCMP
	wpa_group_rekey=86400
	ieee80211n=1
	wme_enabled=1

Редактируем `/etc/default/hostapd`

	sudo nano /etc/default/hostapd

Меняем строку `#DAEMON_CONF=""` на

	DAEMON_CONF="/etc/hostapd/hostapd.conf"

Далее аналогично в `/etc/init.d/hostapd`

	sudo nano /etc/init.d/hostapd

Меняем `DAEMON_CONF=` на

	DAEMON_CONF=/etc/hostapd/hostapd.conf

Готово! Проверяем:

	sudo /usr/sbin/hostapd /etc/hostapd/hostapd.conf

**Итоги:**

Работает как запланировано!
Правда качеством звука встроенного в Pi аналогового выхода я не доволен - много шумов, заказал I2S звуковую карточку :)

**FAQ**

*** Как проверить звук на аудиовыходе? ***

	speaker-test -t sine

*** Как принудительно выбрать аналоговый выход для вывода звука? ***

	amixer cset numid=3 1

(0=auto, 1=headphones, 2=hdmi)