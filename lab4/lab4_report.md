- University: [ITMO University](https://itmo.ru/ru/)
- Faculty: [FICT](https://fict.itmo.ru)
- Course: [Introduction in routing](https://github.com/itmo-ict-faculty/introduction-in-routing)
- Year: 2025/2026
- Group: K3322
- Author: Kirichenko Aleksandra Petrovna
- Lab: Lab4
- Date of create: 14.12.2025
- Date of finished: 15.12.2025

# Задание

Вам необходимо сделать IP/MPLS сеть связи для "RogaIKopita Games" изображенную на рисунке 1 в ContainerLab. Необходимо создать все устройства указанные на схеме и соединения между ними.

<img width="1478" height="728" alt="image" src="https://github.com/user-attachments/assets/c3af52c6-d9c6-4717-8fd1-6c6390f26a60" />

Рисунок 1 - Схема связи IP/MPLS сети компании "RogaIKopita Games" с внедренным BGPv4

- Помимо этого вам необходимо настроить IP адреса на интерфейсах.
- Настроить OSPF и MPLS.
- Настроить iBGP с route reflector кластером

И вот тут лабораторная работа работа разделяется на 2 части, в первой части вам надо настроить L3VPN, во второй настроить VPLS, но при этом менять топологию не требуется. Вы можете просто разобрать VRF и на их месте собрать VPLS.

**Первая часть:**
- Настроить iBGP RR Cluster.
- Настроить VRF на 3 роутерах.
- Настроить RD и RT на 3 роутерах.
- Настроить IP адреса в VRF.
- Проверить связность между VRF
- Настроить имена устройств, сменить логины и пароли.

**Вторая часть:**
- Разобрать VRF на 3 роутерах (или отвязать их от интерфейсов).
- Настроить VPLS на 3 роутерах.
- Настроить IP адресацию на PC1,2,3 в одной сети.
- Проверить связность.

# Схема сети

<img width="1461" height="899" alt="image" src="https://github.com/user-attachments/assets/00a1b9af-db75-4cbe-b785-4e0c64763f08" />

# Часть 1

## Сеть управления

Ниже файл `lab4-1.clab.yaml`:

```
name: lab4-1
mgmt:
  network: custom_mgmt
  ipv4-subnet: 172.16.16.0/24

topology:
  kinds:
    vr-ros:
      image: vrnetlab/mikrotik_routeros:6.47.9

  nodes:
    R01.SPB:
      kind: vr-ros
      mgmt-ipv4: 172.16.16.101
      startup-config: config/part1/r01-spb.rsc
    R01.HKI:
      kind: vr-ros
      mgmt-ipv4: 172.16.16.102
      startup-config: config/part1/r01-hki.rsc
    R01.SVL:
      kind: vr-ros
      mgmt-ipv4: 172.16.16.103
      startup-config: config/part1/r01-svl.rsc
    R01.LND:
      kind: vr-ros
      mgmt-ipv4: 172.16.16.104
      startup-config: config/part1/r01-lnd.rsc
    R01.LBN:
      kind: vr-ros
      mgmt-ipv4: 172.16.16.105
      startup-config: config/part1/r01-lbn.rsc
    R01.NY:
      kind: vr-ros
      mgmt-ipv4: 172.16.16.106
      startup-config: config/part1/r01-ny.rsc
    PC1:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.16.16.2
      binds:
        - ./config:/config
      exec:
        - sh /config/pc.sh
    PC2:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.16.16.3
      binds:
        - ./config:/config
      exec:
        - sh /config/pc.sh
    PC3:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.16.16.4
      binds:
        - ./config:/config
      exec:
        - sh /config/pc.sh

  links:
    - endpoints: ["R01.SPB:eth1","R01.HKI:eth1"]
    - endpoints: ["R01.NY:eth1","R01.LND:eth1"]
    - endpoints: ["R01.SVL:eth1","R01.LBN:eth1"]
    - endpoints: ["R01.HKI:eth2","R01.LND:eth3"]
    - endpoints: ["R01.HKI:eth3","R01.LBN:eth2"]
    - endpoints: ["R01.LND:eth2","R01.LBN:eth3"]
    - endpoints: ["R01.SPB:eth2","PC1:eth1"]
    - endpoints: ["R01.NY:eth2","PC2:eth1"]
    - endpoints: ["R01.SVL:eth2","PC3:eth1"]
```

Затем деплоим с помощью команды `clab deploy -t lab4-1.clab.yaml`:

<img width="972" height="656" alt="image" src="https://github.com/user-attachments/assets/11121411-e3a6-4396-b27f-23b4d6eb3b2f" />

Строим граф топологии с помощью команды `clab graph -t lab4-1.clab.yaml`:

<img width="837" height="604" alt="image" src="https://github.com/user-attachments/assets/6a98cc67-626a-4b37-aacf-ce6796a4b933" />

## Написание конфигов

### Роутеры R01

*OSPF:*

Используем loopback, потому что это интерфейс с IP-адресом, который сам по себе никогда не упадёт без вмешательства. Указываем в router-id адрес loopback интерфейса. Нужна только 1 зона для всех устройств - пишем её имя и все физические подключения.

*MPLS:*

Также используем адрес loopback для transport-address. Ограничиваем, какие префиксы сетей будут получать ярлыки и указываем все интерфейсы роутеров.

*iBGP c Route Reflector Cluster:*

Autonomous System Number (система сетей под управлением единственной административной зоны) выбираем 65000 (можно любое из диапазона). В конфигах в зоне конфигурируемого роутера и у соседей мы будем указывать этот номер. В `instance` указываем ID роутера такой же, что и у loopback. В `peer` указываем роутер HKI, поскольку они соседние. В `address-families` - `l2vpn, vpnv4`. Привязываем к loopback.

*VRF на внешних роутерах:*

В `interface bridge` привязываем рут VRF и свой адрес к мосту. В `ip address` адрес для моста. Далее в `ip route vrf` параметры с тем же ASN и указываем `routing-mark=VRF_DEVOPS`.

Файл `r01-spb.rsc`:
 
```
/user
add name=sasha password=1234 group=full
remove admin
/system identity
set name=R01.SPB

/ip address
add address=10.20.1.1/30 interface=ether2
add address=192.168.10.1/24 interface=ether3

/ip pool
add name=dhcp-pool ranges=192.168.10.10-192.168.10.100
/ip dhcp-server
add address-pool=dhcp-pool disabled=no interface=ether3 name=dhcp-server
/ip dhcp-server network
add address=192.168.10.0/24 gateway=192.168.10.1

/interface bridge
add name=loopback
/ip address 
add address=10.255.255.1/32 interface=loopback network=10.255.255.1

/routing ospf instance
add name=inst router-id=10.255.255.1
/routing ospf area
add name=backbonev2 area-id=0.0.0.0 instance=inst
/routing ospf network
add area=backbonev2 network=10.20.1.0/30
add area=backbonev2 network=192.168.10.0/24
add area=backbonev2 network=10.255.255.1/32

/mpls ldp
set lsr-id=10.255.255.1
set enabled=yes transport-address=10.255.255.1
/mpls ldp interface
add interface=ether2

/routing bgp instance
set default as=65000 router-id=10.255.255.1
/routing bgp peer
add name=peerHKI remote-address=10.255.255.2 address-families=l2vpn,vpnv4 remote-as=65000 update-source=loopback route-reflect=no 
/routing bgp network
add network=10.255.255.0/24

/interface bridge 
add name=br100
/ip address
add address=10.100.1.1/32 interface=br100
/ip route vrf
add export-route-targets=65000:100 import-route-targets=65000:100 interfaces=br100 route-distinguisher=65000:100 routing-mark=VRF_DEVOPS
/routing bgp instance vrf
add redistribute-connected=yes routing-mark=VRF_DEVOPS
```

### Компьютеры

Аналогично предыдущим лабораторным. Содержание `pc.sh`:

```
#!/bin/sh
ip route del default via 172.16.18.1 dev eth0
udhcpc -i eth1
```

## Проверка

### Роутеры R01

Заходим с помощью команды `ssh sasha@clab-lab3-R01.HKI`

### *OSPF:*

<img width="726" height="490" alt="image" src="https://github.com/user-attachments/assets/25154a02-8329-4f42-b6c6-fc3e36cb8023" />

<img width="729" height="490" alt="image" src="https://github.com/user-attachments/assets/dbe15ece-3a91-420c-9f15-e9c7d9ab1504" />

<img width="723" height="488" alt="image" src="https://github.com/user-attachments/assets/c9d0485e-da94-4a65-8829-f3eeb084581e" />

<img width="742" height="535" alt="image" src="https://github.com/user-attachments/assets/4120372e-6f3b-4179-b780-bea1c63d5910" />

<img width="723" height="534" alt="image" src="https://github.com/user-attachments/assets/753f8348-8a6f-41db-adaa-e720c126de3d" />

<img width="720" height="534" alt="image" src="https://github.com/user-attachments/assets/76601bce-6f6b-44e4-90b0-0c74cdc56446" />

### *MPLS:*

<img width="1002" height="792" alt="image" src="https://github.com/user-attachments/assets/4cca891f-bfb0-4894-a482-94552e3a1abb" />

<img width="1002" height="641" alt="image" src="https://github.com/user-attachments/assets/f6a2ea49-21b5-486d-bc1e-9df363c87e50" />

<img width="998" height="774" alt="image" src="https://github.com/user-attachments/assets/046f2834-5664-4732-a413-ef3472d00d00" />

<img width="1000" height="667" alt="image" src="https://github.com/user-attachments/assets/c6ed33f9-18de-493a-9ac1-5a762aa3fbf9" />

<img width="999" height="665" alt="image" src="https://github.com/user-attachments/assets/c94eb8f2-df95-486f-a077-53264911d8c3" />

<img width="997" height="507" alt="image" src="https://github.com/user-attachments/assets/38ea0853-9538-4a84-9f45-b0619729c100" />

### *iBGP:*

<img width="1002" height="687" alt="image" src="https://github.com/user-attachments/assets/7c080bcd-4a06-442b-9569-30b827a4687e" />

<img width="1002" height="684" alt="image" src="https://github.com/user-attachments/assets/cdb1205d-8abc-481a-9186-40e9dc4afdca" />

<img width="998" height="690" alt="image" src="https://github.com/user-attachments/assets/44a0d883-971d-4f93-9231-8bb201cb2590" />

<img width="1000" height="447" alt="image" src="https://github.com/user-attachments/assets/906fda74-9f1d-44a2-a0c1-95d70bbcef4f" />

<img width="999" height="444" alt="image" src="https://github.com/user-attachments/assets/89520877-8173-401f-a44d-6c6fd4ecf6db" />

<img width="997" height="423" alt="image" src="https://github.com/user-attachments/assets/1a19bc6d-57d1-4c78-b5de-c34074cb10cf" />

### *VRF:*

Маршруты на внешних роутерах:

<img width="794" height="295" alt="image" src="https://github.com/user-attachments/assets/823c4b9b-aade-4907-920e-f97a14d11453" />

<img width="748" height="180" alt="image" src="https://github.com/user-attachments/assets/2f7baf3b-bc2d-4dfd-984d-620a591ed748" />

<img width="728" height="177" alt="image" src="https://github.com/user-attachments/assets/a4bd69aa-cf66-47f5-8578-293f808dd99d" />

Ping между роутерами:

<img width="794" height="295" alt="image" src="https://github.com/user-attachments/assets/28f1a5fa-9ad5-4203-81d4-815ddca31b45" />

<img width="784" height="302" alt="image" src="https://github.com/user-attachments/assets/d4db23c2-1da5-4eb2-a1d5-4b01cc32f819" />

<img width="789" height="287" alt="image" src="https://github.com/user-attachments/assets/c2f3ed66-82b2-4cf0-9605-dd67ee9b6664" />

Выходим с помощью команды `quit`.

# Часть 2

## Написание конфигов

### Роутеры R01

*VPLS:*

В `mpls ldp interface` нужно указать интерфейс, направленный на компьютер. На всех 3-х внешних роутерах настраиваем `bridge port`, который направлен на компьютер, руты как у `ip route vrf`, а вместо `routing-mark` указываем `site-id` с уникальным ID для каждого роутера.

Чтобы задать IP компьютерам в одной сети VPN, нужно на одном из роутеров поставить dhcp-сервер, я выбрала SPB. Убираем раздачу dhcp-адресов с роутеров из 1-й части, на SPB создаём новый пул из сети VPN и подключаем его к нему.

Часть, которая изменилась в файле `r01-spb.rsc` во 2 части:

```
/mpls ldp
set lsr-id=10.255.255.1
set enabled=yes transport-address=10.255.255.1
/mpls ldp interface
add interface=ether2
add interface=ether3

/interface bridge
add name=vpn
/interface bridge port
add interface=ether3 bridge=vpn
/interface vpls bgp-vpls
add bridge=vpn export-route-targets=65000:100 import-route-targets=65000:100 name=vpn route-distinguisher=65000:100 site-id=1
/ip address
add address=10.100.1.1/24 interface=vpn

/ip pool
add name=vpn-dhcp-pool ranges=10.100.1.100-10.100.1.254
/ip dhcp-server
add address-pool=vpn-dhcp-pool disabled=no interface=vpn name=dhcp-vpls
/ip dhcp-server network
add address=10.100.1.0/24 gateway=10.100.1.1
```

## Проверка

### Роутеры R01

Заходим с помощью команды `ssh sasha@clab-lab3-R01.SPB`.

### *VPLS:*

Раздаем IP-адреса через DHCP-сервер на роутере SPB:

<img width="860" height="140" alt="image" src="https://github.com/user-attachments/assets/a565352a-c87f-4e09-8e31-fb48aec785cd" />

Выходим с помощью команды `quit`.

### Компьютеры

Заходим с помощью команды `docker exec -it clab-lab4-2-PC1 sh`.

<img width="596" height="422" alt="image" src="https://github.com/user-attachments/assets/4f4ffdf8-ce1f-4de2-8575-eaef57a10876" />

<img width="596" height="422" alt="image" src="https://github.com/user-attachments/assets/d8914aed-8859-46aa-8c93-5f2b0b6ce9c8" />

<img width="596" height="422" alt="image" src="https://github.com/user-attachments/assets/800a7f7e-1b13-44e2-8681-43819cd8249f" />

Выходим с помощью команды `exit`.

## Вывод

В ходе работы была создана IP/MPLS сеть. На ней были настроены протоколы OSPF, MPLS и iBGP с Route Reflector-кластером. В 1 части был настроен L3VPN, VRF, во 2 части был настроен VPLS.
