# VPN-PPTP
Для установки понадобятся права root пользователя или sudo.

1. Устанавливаем необходимые пакеты:

		apt-get install ppp pptpd

После окончания установки, нам понадобится отредактировать несколько файлов. Для изменения файлов вы можете пользоваться любым удобным для вас способом, хоть nano, хоть визуальными редакторами и т.д.

Теперь вам необходимо определиться с локальной подсетью для клиентов VPN. Вы можете сделать себе подсеть из любой подсети, которая не маршутизируется в интернете:
		
		10.0.0.0/8
		172.16.0.0/12
		192.0.2.0/24
		192.88.99.0/24
		192.168.0.0/16
		198.18.0.0/15
		224.0.0.0/4
		240.0.0.0/4
		100.64.0.0/10

2. Открываем файл /etc/pptpd.conf, находим и раскоментируем (если вдруг закоментированы) строки localip и remoteip, затем прописываем свою ip адресацию.

		localip 172.16.0.1
		remoteip 172.16.0.2-254

    localip - ip адрес из выбранной вами подсети, который будет являться локальным шлюзом для клиентов VPN.
    remoteip - пул ip адресов для раздачи клиентам VPN.

Если на вашей машине несколько внешних IP адресов, то вы можете указать конкретный IP, по которому будет доступно подключение к VPN серверу. В конце файла добавьте:
	
	listen внешний_ip

3. Открываем файл /etc/ppp/pptpd-options и добавляем в конце файла:
	
		mtu 1400
		mru 1400 (В конце настройки может жаловаться на неизвестый параметр, можно убрать)
		auth
		require-mppe

В этом же файле при необходимости вы можете указать конкретные DNS, которые будут использоваться при подключении через VPN. Найдите и раскоментируйте строки ms-dns:

	ms-dns 8.8.8.8
	ms-dns 8.8.4.4

4. Открываем стандартный файл /etc/sysctl.conf, находим и раскоментируем строку:
	
		net.ipv4.ip_forward=1

5. Добавляем необходимые правила в iptables:
	
		iptables -A INPUT -p gre -j ACCEPT
		iptables -A INPUT -m tcp -p tcp --dport 1723 -j ACCEPT
		iptables -t nat -A POSTROUTING -o eth0 -j MASQUERADE
	
Обратите внимание, что eth0 - имя вашего сетевого интерфейса. Вы можете узнать его с помощью команды ifconfig

Если вам необходимо, чтобы была локальная сеть между клиентами, подключенными к VPN, добавьте следующие правила в iptables:

	
	iptables --table nat --append POSTROUTING --out-interface ppp0 -j MASQUERADE
	iptables -I INPUT -s 172.16.0.0/24 -i ppp0 -j ACCEPT
	iptables --append FORWARD --in-interface eth0 -j ACCEPT

Обратите внимание, что 172.16.0.0/24 - локальная подсеть, которую вы себе выбрали, а ppp0 - имя pptp интерфейса.

Для сохранения iptables, выполните команду:
	
	iptables-save 
(советую скачать утилиту убунту иногда шалит и не сохраняет: sudo apt-get install iptables-persistent; sudo netfilter-persistent save)

6. Пользователей для подключения к VPN серверу добавляем в файле /etc/ppp/chap-secrets
	
		user1	pptpd	password1	"*"
		user2	pptpd	password2	"172.16.0.2"

	user1 - имя пользователя
	password1 - пароль пользователя
	"*" - локальный ip будет выдаваться из пула, указанного в файле /etc/pptpd.conf
	"172.16.0.2" - пользователю будет присвоен указанный ip адрес.

7. Перезапускаем сервис pptpd для применения новых настроек:
	
		service pptpd restart

Ваш PPTP сервер на Ubuntu запущен!

P.S. Если вдруг вы столкнетесь с тем, что добавленные правила iptables пропадают после рестарта firewall или перезагрузки машины, то сделайте действия, описанные ниже.

1. Добавляем заново все правила, как описано в 5-м шаге.
2. Сохраняем правила в конфиг:
		
		iptables-save > /etc/iptables.conf

3. Открываем файл /etc/network/interfaces и добавляем в самый конец:
	
		pre-up /sbin/iptables-restore < /etc/iptables.conf
	
================================================================================
Настройка клиента
Install necessary packages:

		apt-get install pptp-linux pptpd ppp curl

Create PPTP configuration file:

		nano /etc/ppp/peers/tgvpn

Enter this as content of the "tgvpn" file:
(replace 124.123.123.123 is the IP of the PPTP server you want to connect to, and MYTGACCOUNTUSERNAME with your VPN username)

		pty "pptp 123.123.123.123 --nolaunchpppd"
		lock
		noauth
		nobsdcomp
		nodeflate
		name MYTGACCOUNTUSERNAME
		remotename tgvpn
		ipparam tgvpn
		require-mppe-128
		usepeerdns
		defaultroute
		persist

Enter VPN login credentials into chap-secrets file:
([tab] being replaced by a tab, username with your VPN account username and password with your PPTP password):

		nano /etc/ppp/chap-secrets
		username[tab]tgvpn[tab]password[tab]*

Create script to replace default routes - otherwise the VPN is not being used by your system:

		nano /etc/ppp/ip-up.local

Enter this as content of the "ip-up.local" file:

		#!/bin/bash
		H=`ps aux | grep 'pppd pty' | grep -v grep | awk '{print $14}'`
		DG=`route -n | grep UG | awk '{print $2}'`
		DEV=`route -n | grep UG | awk '{print $8}'`
		route add -host $H gw $DG dev $DEV
		route del default $DEV
		route add default dev ppp0

Make this script executable:

		chmod +x /etc/ppp/ip-up.local

To connect to the VPN:

		pon tgvpn

To disconnect from the VPN:

		poff tgvpn
