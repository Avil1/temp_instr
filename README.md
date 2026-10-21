# temp_instr
Схема: доступ в интернет, esr-21,к нему подключн коммутатор mes 21 и к нему 2 клиента на разных vlan
1. Настройка esr-21:
  1) подключаем один клиент к esr-21
  2) на момент подключения IP-клиента - 192.168.1.2, Ip esr - 192.168.1.1 (password->toor)
  3) commit+confirm
1.1 Настройка глобального DNS:
  1) domain lookup enable
  2) domain name-server 8.8.8.8 (1.1.1.1 - cloudflare)
  3) commit+confirm
1.2 Настрока бриджей (firewall)
  1) configure
  2) bridge 1
  3) ip firewall disable
  4) exit
  5) bridge 2
  6) ip firewall disable
  7) exit
  8) exit
  9) commit+confirm
1.3 Настройка DHCP (первый)
  1) configure
  2) ip dhcp-server pool lan_pool
  3) network 172.16.100.0/28
  4) dofault-router 172.16.100.1
  5) dns-server 172.16.100.1
  6) address-range 172.16.100.2-172.16.100.14
  7) exit
  8) exit
  9) commit+confirm
1.4 Настройка DHCP (второй)
  1) configure
  2) ip dhcp-server pool lan_pool2
  3) network 192.168.100.0/27
  4) dofault-router 192.168.100.1
  5) dns-server 192.168.100.1
  6) address-range 192.168.100.2-192.168.100.14
  7) exit
  8) exit
  9) commit+confirm
1.5 Настрока резервного порта (8)
  1) configure
  2) interface gigabitethernet 1/0/8
  3) mode routerport
  4) ip firewall disable
  5) ip address 192.168.10.1/26
  6) exit
  7) exit
  8) commit+confirm
1.6 Проверка порта (8)
  1) (клиент 1) ip addr flush dev enp0s31f6 //интерфейс
  2) ip addr add 192.168.10.2/26 dev enp0s31f6
  3) ip route add default via 192.168.10.1
  4) ping 192.168.10.1 //работает
  5) просто заходим в esr по тому же паролю
1.7 Настройка sub-interface'ов
  1) configure
  2) interface gigabitethernet 1/0/3
  3) mode routerport
  4) exit
  5) interface gigabitethernet 1/0/3.100
  6) description VLAN100_to_mes
  7) security-zone trusted
  8) ip address 172.16.100.1/28
  9) interface gigabitethernet 1/0/3.200
  10) description VLAN200_to_mes
  11) security-zone trusted
  12) ip address 192.168.100.1/27
  13) exit
  14) exit
  15) commit+confirm
1.8 Настрока NAT
  1) object-group network LAN
  2) ip prefix 172.16.100.0/28
  3) ip prefix 192.168.100.0/27
  4) ip prefix 192.168.10.0/26
  5) exit
  6) nat source
  7) ruleset SNAT
  8) to interface bridge 2
  9) rule 1
  10) match source-address LAN
  11) action source-nat interface
  12) enable
  13) exit
  14) exit
  15) exit
  16) commit+confirm
  17) configure
  18) interface gigabitethernet 1/0/3.100
  19) ip firewall disable
  20) exit
  21) interface gigabitethernet 1/0/3.200
  22) ip firewall disable
  23) exit
  24) commit+confirm
2. Настройка клиентов:
  1) etc/net/ifaces/ваш интерфейс //изменить на динамику
Options (DHCP)
TYPE=eth
SYSTEMD_BOOTPROTO=dhcp4
BOOTPROTO=dhcp
ONBOOT=yes
DISABLED=no
Файл Ipv4address очистить
НА ОБОИХ КЛИЕНТАХ
3. Настройка MES
   1) подключаемся через 1/0/1. Заходим в браузер на 192.168.1.239
   2) Вход admin/admin
   3) Настройки L2->VLAN->Параметры->добавить
  1/0/2: NAME=VLAN100 PVID=100 MODE=TRUNK
  1/0/3: NAME=VLAN100 PVID=100 MODE=ACCESS 
  1/0/4: NAME=VLAN200 PVID=200 MODE=ACCESS
   4) Настройки L2->VLAN->Членство
   5) для VLAN100:
  1/0/2 -> TAGGED
  1/0/3 -> UNTAGGED
  1/0/4 -> EXCLUDE
   5) для VLAN200:
  1/0/2 -> TAGGED
  1/0/3 -> EXCLUDE
  1/0/4 -> UNTAGGED

СОХРАНИТЬ ВСЁ И СОБРАТЬ СХЕМУ
