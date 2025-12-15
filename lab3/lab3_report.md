- University: [ITMO University](https://itmo.ru/ru/)
- Faculty: [FICT](https://fict.itmo.ru)
- Course: [Introduction in routing](https://github.com/itmo-ict-faculty/introduction-in-routing)
- Year: 2025/2026
- Group: K3322
- Author: Kirichenko Aleksandra Petrovna
- Lab: Lab3
- Date of create: 11.12.2025
- Date of finished: 15.12.2025

# Задание

Вам необходимо сделать IP/MPLS сеть связи для "RogaIKopita Games" изображенную на рисунке 1 в ContainerLab. Необходимо создать все устройства указанные на схеме и соединения между ними.

<img width="1034" height="516" alt="image" src="https://github.com/user-attachments/assets/1b07cf58-bb38-42e0-a03f-43c173160691" />

- Помимо этого вам необходимо настроить IP адреса на интерфейсах.
- Настроить OSPF и MPLS.
- Настроить EoMPLS.
- Назначить адресацию на контейнеры, связанные между собой EoMPLS.
- Настроить имена устройств, сменить логины и пароли.

# Схема сети

<img width="1290" height="564" alt="image" src="https://github.com/user-attachments/assets/94f058dc-9c81-4288-8dd5-cbaad4142a97" />

## Сеть управления

Ниже файл `lab3.clab.yaml`:

```
name: lab3
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
      startup-config: config/r01-spb.rsc
    R01.HKI:
      kind: vr-ros
      mgmt-ipv4: 172.16.16.102
      startup-config: config/r01-hki.rsc
    R01.MSK:
      kind: vr-ros
      mgmt-ipv4: 172.16.16.103
      startup-config: config/r01-msk.rsc
    R01.LND:
      kind: vr-ros
      mgmt-ipv4: 172.16.16.104
      startup-config: config/r01-lnd.rsc
    R01.LBN:
      kind: vr-ros
      mgmt-ipv4: 172.16.16.105
      startup-config: config/r01-lbn.rsc
    R01.NY:
      kind: vr-ros
      mgmt-ipv4: 172.16.16.106
      startup-config: config/r01-ny.rsc
    PC1:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.16.16.2
      binds:
        - ./config:/config
      exec:
        - sh /config/pc1.sh
    SGI-PRISM:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 172.16.16.3
      binds:
        - ./config:/config
      exec:
        - sh /config/sgi.sh

  links:
    - endpoints: ["R01.SPB:eth1","R01.HKI:eth1"]
    - endpoints: ["R01.SPB:eth2","R01.MSK:eth1"]
    - endpoints: ["R01.SPB:eth3","PC1:eth1"]
    - endpoints: ["R01.HKI:eth2","R01.LBN:eth2"]
    - endpoints: ["R01.HKI:eth3","R01.LND:eth1"]
    - endpoints: ["R01.MSK:eth2","R01.LBN:eth1"]
    - endpoints: ["R01.LND:eth2","R01.NY:eth1"]
    - endpoints: ["R01.LBN:eth3","R01.NY:eth2"]
    - endpoints: ["R01.NY:eth3", "SGI-PRISM:eth1"]
```

Затем деплоим с помощью команды `clab deploy -t lab3.clab.yaml`:

<img width="975" height="596" alt="image" src="https://github.com/user-attachments/assets/cbfe02da-dabc-4b99-9a25-50b1d1cdc37f" />

Строим граф топологии с помощью команды `clab graph -t lab3.clab.yaml`:

<img width="1147" height="514" alt="image" src="https://github.com/user-attachments/assets/8a28310c-57b4-4923-9888-57551f668b47" />

## Написание конфигов

### Роутеры R01

*OSPF:*

Используем loopback, потому что это интерфейс с IP-адресом, который сам по себе никогда не упадёт без вмешательства. Указываем в router-id адрес loopback интерфейса. Нужна только 1 зона для всех устройств - пишем её имя и все физические подключения.

*MPLS:*

Также используем адрес loopback для transport-address. Ограничиваем, какие префиксы сетей будут получать ярлыки и указываем все интерфейсы роутеров.

*EoMPLS (на роутерах NY и SPB):*

Указываем VPN, который соединяем с интерфейсом vpls и портом. В remote-peer пишем ip loopback-интерфейса другого роутера.

```
/user
add name=sasha password=1234 group=full
remove admin
/system identity
set name=R01.NY

/ip address
add address=10.20.6.2/30 interface=ether2
add address=10.20.7.2/30 interface=ether3
add address=192.168.11.1/24 interface=ether4

/ip pool
add name=dhcp-pool ranges=192.168.11.10-192.168.11.100
/ip dhcp-server
add address-pool=dhcp-pool disabled=no interface=ether4 name=dhcp-server
/ip dhcp-server network
add address=192.168.11.0/24 gateway=192.168.11.1

/interface bridge
add name=loopback
/ip address 
add address=10.255.255.6/32 interface=loopback network=10.255.255.6

/routing ospf instance
add name=inst router-id=10.255.255.6
/routing ospf area
add name=backbonev2 area-id=0.0.0.0 instance=inst
/routing ospf network
add area=backbonev2 network=10.20.6.0/30
add area=backbonev2 network=10.20.7.0/30
add area=backbonev2 network=192.168.11.0/24
add area=backbonev2 network=10.255.255.6/32

/mpls ldp
set lsr-id=10.255.255.6
set enabled=yes transport-address=10.255.255.6
/mpls ldp advertise-filter
add prefix=10.255.255.6/32 advertise=yes
/mpls ldp advertise-filter 
add prefix=10.255.255.0/24 advertise=yes
add advertise=no
/mpls ldp accept-filter 
add prefix=10.255.255.0/24 accept=yes
add accept=no
/mpls ldp interface
add interface=ether2
add interface=ether3

/interface bridge
add name=vpn
/interface vpls
add disabled=no name=SGIPC remote-peer=10.255.255.1 cisco-style=yes cisco-style-id=0
/interface bridge port
add interface=ether2 bridge=vpn
add interface=SGIPC bridge=vpn
```

### Компьютеры

IP-адреса выдаются компьютерам через DHCP, но в Containerlab у них выставляется дефолтный маршрут от сети управления в yaml-конфиге, из-за чего попытки послать запрос на передаются на роутер моего провайдера, где этот запрос теряется. Поэтому необходимо удалить этот маршрут.

```
#!/bin/sh
ip route del default via 172.16.16.1 dev eth0
udhcpc -i eth1
```

## Проверка

### Роутеры R01

Заходим с помощью команды `ssh sasha@clab-lab3-R01.NY`

### *OSPF:*

<img width="721" height="559" alt="image" src="https://github.com/user-attachments/assets/1e9dc572-f655-45ea-a150-03fcb62740c3" />

<img width="721" height="535" alt="image" src="https://github.com/user-attachments/assets/22127a87-fb15-443b-b0b2-635f11f0ca37" />

<img width="721" height="535" alt="image" src="https://github.com/user-attachments/assets/47c72ee9-57e2-4195-8454-78e46623ae07" />

<img width="721" height="535" alt="image" src="https://github.com/user-attachments/assets/a951db70-6ab0-471c-bd97-74a50ddf003e" />

<img width="721" height="535" alt="image" src="https://github.com/user-attachments/assets/13ac9bf8-3467-4c12-ba31-bf358f345f87" />

<img width="721" height="556" alt="image" src="https://github.com/user-attachments/assets/487bec0d-b232-40e0-91a1-9e0445ccbfc9" />

### *MPLS:*

<img width="965" height="625" alt="image" src="https://github.com/user-attachments/assets/2283a9ad-1505-47c4-80a3-689ccbfdb7d7" />

<img width="957" height="500" alt="image" src="https://github.com/user-attachments/assets/25cae11b-1014-495f-9ba9-b35c86271fa1" />

<img width="957" height="654" alt="image" src="https://github.com/user-attachments/assets/a32af9b6-1fd4-433b-bf82-f4dd12076e3a" />

<img width="957" height="565" alt="image" src="https://github.com/user-attachments/assets/1f9b8d4d-a47e-45b6-a584-870b08c0ae60" />

<img width="957" height="586" alt="image" src="https://github.com/user-attachments/assets/b25a89a9-db83-4eda-8316-e78ea929e69d" />

<img width="957" height="586" alt="image" src="https://github.com/user-attachments/assets/b8a89eb0-77a0-4ebb-acb0-3686b7a72242" />

### *VPLS:*

<img width="486" height="97" alt="image" src="https://github.com/user-attachments/assets/a1dc7a96-91fc-4e99-84ed-af0aa4fe80d8" />

<img width="482" height="103" alt="image" src="https://github.com/user-attachments/assets/2d8d2acf-fd2c-4662-b5e1-f4092f72fcf1" />

Выходим с помощью команды `quit`.

### Компьютеры

Заходим с помощью команды `docker exec -it clab-lab3-PC1 sh`.

<img width="582" height="230" alt="image" src="https://github.com/user-attachments/assets/83946bd4-75f7-44d6-9dcb-fb5bcec24df9" />

<img width="604" height="230" alt="image" src="https://github.com/user-attachments/assets/c814ef1d-434b-4056-be09-713feb470726" />

Выходим с помощью команды `exit`.

Перед выполнением следующей лабораторной нужно прописать команду `clab destroy -t lab3.clab.yaml`.

## Вывод

В ходе работы была настроена динамическая маршрутизация через OSFP, поверх которой положена сеть MPLS, и был проведён туннель VPLS между роутерами NY и SPB.
