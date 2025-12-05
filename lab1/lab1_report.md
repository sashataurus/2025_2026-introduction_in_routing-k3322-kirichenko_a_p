- University: [ITMO University](https://itmo.ru/ru/)
- Faculty: [FICT](https://fict.itmo.ru)
- Course: [Introduction in routing](https://github.com/itmo-ict-faculty/introduction-in-routing)
- Year: 2025/2026
- Group: K3322
- Author: Kirichenko Aleksandra Petrovna
- Lab: Lab1
- Date of create: 01.12.2025
- Date of finished: 01.12.2025

## Задание

- Вам необходимо сделать трехуровневую сеть связи классического предприятия изображенную на рисунке 1 в ContainerLab. Необходимо создать все устройства указанные на схеме и соединения между ними, правила работы с СontainerLab можно изучить по [ссылке](https://containerlab.dev/quickstart/).

<img width="431" height="311" alt="image" src="https://github.com/user-attachments/assets/72797011-72ec-43a7-aaf6-22c2ae444e67" />

Рисунок 1 - Трехуровневая сеть связи классического предприятия

**Подсказка №1** Не забудьте создать mgmt сеть, чтобы можно было зайти на CHR

**Подсказка №2** Для mgmt_ipv4 не выбирайте первый и последний адрес в выделенной сети, ходить на CHR можно используя SSH и Telnet (admin/admin)

- Помимо этого вам необходимо настроить IP адреса на интерфейсах и 2 VLAN-a для PC1 и PC2, номера VLAN-ов вы вольны выбрать самостоятельно.

- Также вам необходимо создать 2 DHCP сервера на центральном роутере в ранее созданных VLAN-ах для раздачи IP адресов в них. PC1 и PC2 должны получить по 1 IP адресу из своих подсетей.

- Настроить имена устройств, сменить логины и пароли.

## Подготовка к лабораторной работе

У меня Мас с процессором Apple Silicon, поэтому у меня возникли трудности - не получилось выполнить работу на Ubuntu на UTM, посколько там невозможна вложеннная виртуализация. В итоге пришлось обратиться за помощью к одногруппнику и попросить его компьютер на Windows.

Для начала нужно было установить Docker, и в конфигурации `lxc` контейнера включить `security.nesting`.

Затем нужно установить `make`, склонировать `hellt/vrnetlab`, перейти в папку `routeros`, загрузить в эту папку `chr-6.47.9.vmdk` и с помощью `make docker-image` собрать образ.

Затем установим ContainerLab с помощью команды `bash -c "$(curl -sL https://get.containerlab.dev)"`.

## Схема сети

<img width="542" height="557" alt="image" src="https://github.com/user-attachments/assets/dc74a49c-8728-41c5-955e-6b0ee9e28eec" />

## Сеть управления

Я создала файл `lab1.clab.yaml`, в котором была создана базовая топология и задана сеть управления:

```
name: lab1

mgmt:
  network: custom-mgmt
  ipv4-subnet: 172.31.0.0/24

topology:
  kinds:
    vr-ros:
      image: vrnetlab/mikrotik_routeros:6.47.9

  nodes:
    R01.TEST:
      kind: vr-ros
      mgmt-ipv4: 172.31.0.10

    SW01.L3.01.TEST:
      kind: vr-ros
      mgmt-ipv4: 172.31.0.11

    SW02.L3.01.TEST:
      kind: vr-ros
      mgmt-ipv4: 172.31.0.12

    SW02.L3.02.TEST:
      kind: vr-ros
      mgmt-ipv4: 172.31.0.13

    PC1:
      kind: linux
      image: ubuntu:latest
      mgmt-ipv4: 172.31.0.2

    PC2:
      kind: linux
      image: ubuntu:latest
      mgmt-ipv4: 172.31.0.3

  links:
    - endpoints: ["R01.TEST:eth1", "SW01.L3.01.TEST:eth1"]
    - endpoints: ["SW01.L3.01.TEST:eth2", "SW02.L3.01.TEST:eth1"]
    - endpoints: ["SW01.L3.01.TEST:eth3", "SW02.L3.02.TEST:eth1"]
    - endpoints: ["SW02.L3.01.TEST:eth2", "PC1:eth1"]
    - endpoints: ["SW02.L3.02.TEST:eth2", "PC2:eth1"]
```
Затем деплоим с помощью команды `clab deploy -t lab1.clab.yaml`:

<img width="813" height="354" alt="image" src="https://github.com/user-attachments/assets/bc624a5b-2ba1-4dd7-b0bf-f6f65578ab19" />

Строим граф топологии с помощью команды `clab graph -t lab1.clab.yaml`:

<img width="455" height="408" alt="image" src="https://github.com/user-attachments/assets/89cec55a-815d-46e6-bb58-7f0c6fa9e78a" />

## Написание конфигов

### Роутер R01

Настроиваем сначала роутер - сздаем 2 VLAN'а на `ether2` (важно, что `Ether1` выделяется под `ether0`, поэтому начинаем брать интерфейсы с `ether2`). Затем задаем ip адрес и пул адресов внутри каждого VLAN и настраиваем DHCP сервера, а также настраиваем имена устройств, меняем логины и пароли:

```
/interface vlan
add name=vlan10 vlan-id=10 interface=ether2
add name=vlan20 vlan-id=20 interface=ether2
/ip address
add address=10.10.0.1/24 interface=vlan10
add address=10.20.0.1/24 interface=vlan20
/ip pool
add name=pool10 ranges=10.10.0.10-10.10.0.254
add name=pool20 ranges=10.20.0.10-10.20.0.254
/ip dhcp-server
add address-pool=pool10 disabled=no interface=vlan10 name=dhcp-server10
add address-pool=pool20 disabled=no interface=vlan20 name=dhcp-server20
/ip dhcp-server network
add address=10.10.0.0/24 gateway=10.10.0.1
add address=10.20.0.0/24 gateway=10.20.0.1
/user
add name=fe0fanov password=1234 group=full
remove admin
/system identity
set name=R01
```

### Свитч SW01

Настраиваем первый свитч. Так как через него идут 2 VLAN'а, то необходимо фильтровать пакеты. Делается это с помощью мостов. Настраиваем интерфейс моста, включаем `vlan-filtering` и указываем имя моста для каждого VLAN. Затем указываем все порты устройства. Далее указываем id VLAN и в `tagged` сам мост и trunk-порты. Далее настраиваем ip, имя устройства, логин и пароль.

```
/interface bridge
add name=bridge1 vlan-filtering=yes
/interface vlan
add name=vlan10 vlan-id=10 interface=bridge1
add name=vlan20 vlan-id=20 interface=bridge1
/interface bridge port
add bridge=bridge1 interface=ether2
add bridge=bridge1 interface=ether3
add bridge=bridge1 interface=ether4
/interface bridge vlan
add bridge=bridge1 tagged=bridge1,ether2,ether3 vlan-ids=10
add bridge=bridge1 tagged=bridge1,ether2,ether4 vlan-ids=20
/ip address
add address=10.10.0.2/24 interface=vlan10
add address=10.20.0.2/24 interface=vlan20
/user
add name=fe0fanov password=1234 group=full
remove admin
/system identity
set name=SW01
```

### Свитчи SW02.01 и SW02.02

Рассмотрим на примере SW02.01. Настраиваем интерфейс моста и VLAN, порты. ЗатемнНастраиваем trunk- и access-порты (`tagged` для trunk, `untagged` для access) и указываем id VLAN. Настраиваем ip, имя, логин и пароль.

```
/interface bridge
add name=bridge1
/interface vlan
add name=vlan10 vlan-id=10 interface=bridge1
/interface bridge port
add bridge=bridge1 interface=ether2
add bridge=bridge1 interface=ether3 pvid=10
/interface bridge vlan
add bridge=bridge1 tagged=bridge1,ether2 untagged=ether3 vlan-ids=10
/ip address
add address=10.10.0.3/24 interface=vlan10
/user
add name=fe0fanov password=1234 group=full
remove admin
/system identity
set name=SW02-01
```

### Компьютеры PC1 и PC2

Компьютеры я решила настроить вручную - зайти в каждый ПК, установить необходимые утилиты (настраиваем VLAN и запрашиваем ip у сервера) и затем настроить. 

```
#!/bin/sh
ip link add link eth1 name vlan10 type vlan id 10
ip link set vlan10 up
udhcpc -i vlan10
ip route add 10.20.0.0/24 via 10.10.0.1 dev vlan10
```

## Ping

Заходим внутрь роутера через команду `ssh username@clab-lab1-R01.TEST` и проверяем работоспособность системы:

## Вывод

В ходе работы были создана трёхуровневая сеть для классического предприятия. Все устройства успешно соединены, были настроены 2 VLAN'а и DHCP-серверы внутри них для раздачи ip компьютерам.
