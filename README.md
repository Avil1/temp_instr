# temp_instr
Схема: доступ в интернет, esr-21,к нему подключен коммутатор mes 21 и к нему 2 клиента на разных vlan
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

Настройка межсетевых экранов
Задание: ПК1 и ПК2 имеют доступ к esr-21,подключение по ssh и пинг
ПК1 может подключится к ПК2 по ssh и аналогично, но они не видят друг друга (запрет по icmp, он же пинг)

Действия:
  1) заходим по ssh: ssh admin@192.168.10.1 (физическое подключение через 8 порт. Пароль - toor)
  2) configure
  3) object-group service http
  4) port-range 80
  5) exit
  6) exit (6-8 - не обязательно)
  7) commit+confirm
  8) configure
  9) object-group service https
  10) port-range 443
  11) exit
  12) object-group service dns
  13) port-range 53
  14) exit
  15) exit
  16) commit+confirm (проверка: всё есть, show candidate-config)
  17) configure
  18) security zone WAN
  19) exit
  20) security zone vlan100
  21) exit
  22) security zone vlan200
  23) exit
  24) exit
  25) commit+confirm
  26) configure
  27) security zone-pair WAN self
  28) rule 1
  29) action permit
  30) match protocol udp
  31) match source-port dhcp_server
  32) match destination-port dhcp_client
  33) enable
  34) exit
  35) rule 10
  36) action permit
  37) match protocol udp
  38) match destination-port dns
  39) enable
  40) exit
  41) rule 20
  42) action permit
  43) match protocol tcp
  44) match destination-port dns
  45) enable
  46) exit
  47) exit
  48) commit+confirm
  49) configure
  50) security zone-pair vlan100 self
  51) rule 10
  52) action permit
  53) match protocol icmp
  54) enable
  55) exit
  56) rule 20
  57) action permit
  58) match protocol tcp
  59) match destination-port ssh
  60) enable
  61) exit
  62) rule 30
  63) action permit
  64) match protocol udp
  65) match source-port dhcp_client
  66) match destination-port dhcp_server
  67) enable
  68) exit
  69) rule 40
  70) action permit
  71) match protocol udp
  72) match destination-port dns
  73) enable
  74) exit
  75) rule 50
  76) action permit
  77) match protocol tcp
  78) match destination-port dns
  79) enable
  80) exit
  81) exit
  82) exit
  83) commit+confirm
  84) configure
  85) security zone-pair vlan200 self
  86) rule 10
  87) action permit
  88) match protocol icmp
  89) enable
  90) exit
  91) rule 20
  92) action permit
  93) match protocol tcp
  94) match destination-port ssh
  95) enable
  96) exit
  97) rule 30
  98) action permit
  99) match protocol udp
  100) match source-port dhcp_client
  101) match destination-port dhcp_server
  102) enable
  103) exit
  104) rule 40
  105) action permit
  106) match protocol udp
  107) match destinapion-port dns
  108) enable
  109) exit
  110) rule 50
  111) action permit
  112) match protocol tcp
  113) match destination-port dns
  114) enable
  115) exit
  116) exit
  117) exit
  118) commit+confirm
  119) configure
  120) security zone-pair vlan100 vlan200
  121) rule 10
  122) action permit
  123) match protocol tcp
  124) match destination-port ssh
  125) enable
  126) exit
  127) exit
  128) exit
  129) commit+confirm
  130) configure
  131) security zone-pair vlan200 vlan100
  132) action permit
  133) match protocol tcp
  134) match destination-port ssh
  135) enable
  136) exit
  137) exit
  138) exit
  139) commit+confirm
  140) configure
  141) security zone-pair vlan100 WAN
  142) rule 10
  143) action permit
  144) match protocol tcp
  145) match destination-port http
  146) enable
  147) exit
  148) rule 20
  149) action permit
  150) match protocol tcp
  151) match destination-port https
  152) enable
  153) exit
  154) rule 30
  155) action permit
  156) match protocol icmp
  157) enable
  158) exit
  159) rule 40
  160) action permit
  161) match protocol udp
  162) match destination-port dns
  163) enable
  164) exit
  165) rule 50
  166) action permit
  167) match protocol tcp
  168) match destination-port dns
  169) enable
  170) exit
  171) exit
  172) exit
  173) commit+confirm
  174) configure
  175) security zone-pair vlan200 WAN
  176) rule 10
  177) action permit
  178) match protocol icmp
  179) enable
  180) exit
  181) rule 20
  182) action permit
  183) match protocol tcp
  184) match destination-port http
  185) enable
  186) exit
  187) rule 30
  188) action permit
  189) match protocol tcp
  190) match destination-port https
  191) enable
  192) exit
  193) rule 40
  194) action permit
  195) match protocol udp
  196) match destination-port dns
  197) enable
  198) exit
  199) rule 50
  200) action permit
  201) match protocol tcp
  202) match destination-port dns
  203) enable
  204) exit
  205) exit
  206) exit
  207) commit+confirm
  208) configure
  209) interface gigabitethernet 1/0/3.100
  210) security-zone vlan100
  211) no ip firewall disable
  212) exit
  213) interface gigabitethernet 1/0/3.200
  214) security-zone vlan200
  215) no ip firewall disable
  216) exit
  217) bridge 2
  218) security-zone WAN
  219) no ip firewall disable
  220) exit
  221) exit
  222) commit+confirm

Модификация схемы: подмена MES на Alt JeOS (systemd) + OVS (Open VSwitch)
  1) ip -c a --> enp0s25, enp1s0, enp2s0 - все интерфейсы (в нашем случае)
  2) vim /etc/net/ifaces/enp0s25/options
  3) BOOTPROTO=dhcp + SYSTEMD_BOOTPROTO=dhcp4
  4) Shift+"Ж"-->qw
  5) systemctl restart network
  6) apt-get update
  7) apt-get install openvswitch
  8) systemctl enable --now openvswitch
  9) modprobe 8021q
  10) echo "8021q" > /etc/modules-load.d/8021q.conf
  11) lsmod | grep 8021q
  12) mc --> /etc/net/ifaces создать папки под каждый интерфейс (если таких нет. Можно и через терминал --> mkdir или f7)
  13) В каждом каталоге создать файл options (touch options, находясь в текущем каталоге через mc)
  14) в каждый из них через f4 (редактирование) добавить всего 3 строки:
TYPE=eth
BOOTPROTO=none
CONFIGURED=yes
  15) сохраняем (f2)
  16) выходим
  17) Ниже:
# Создаём мост
ovs-vsctl add-br ovs-br0

# Uplink к ESR-21 (trunk: только VLAN 100 и 200)
ovs-vsctl add-port ovs-br0 enp0s25 trunk=100,200

# Клиент 1 (access, VLAN 100)
ovs-vsctl add-port ovs-br0 enp1s0 tag=100

# Клиент 2 (access, VLAN 200)
ovs-vsctl add-port ovs-br0 enp2s0 tag=200

  18) Ниже:
cat << 'EOF' >> /etc/rc.local
ip link set enp0s25 up
ip link set enp1s0 up
ip link set enp2s0 up
EOF
chmod +x /etc/rc.local
  19) ВСЁ!!!! Собираем обновлённую схему
