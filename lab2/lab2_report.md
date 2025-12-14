- University: [ITMO University](https://itmo.ru/ru/)
- Faculty: [FICT](https://fict.itmo.ru)
- Course: [Introduction in routing](https://github.com/itmo-ict-faculty/introduction-in-routing)
- Year: 2025/2026
- Group: K3322
- Author: Kirichenko Aleksandra Petrovna
- Lab: Lab2
- Date of create: 09.12.2025
- Date of finished: 15.12.2025

## Задание

- Вам необходимо сделать сеть связи в трех геораспределенных офисах "RogaIKopita Games" изображенную на рисунке 1 в ContainerLab. Необходимо создать все устройства указанные на схеме и соединения между ними.

<img width="548" height="365" alt="image" src="https://github.com/user-attachments/assets/2fe420d6-6631-4a8a-bf9a-33bc64021bee" />

Рисунок 1 - Схема связи трех геораспределенных офисов "RogaIKopita Games"

- Помимо этого вам необходимо настроить IP адреса на интерфейсах.
- Создать DHCP сервера на роутерах в сторону клиентских устройств.
- Настроить статическую маршрутизацию.
- Настроить имена устройств, сменить логины и пароли.

## Схема сети

<img width="908" height="780" alt="image" src="https://github.com/user-attachments/assets/b15882bc-6b56-4640-87d5-0c8272fa1178" />

## Сеть управления

Ниже файл `lab2.clab.yaml`:

```
name: lab2
mgmt:
  network: custom_mgmt
  ipv4-subnet: 10.20.30.0/24

topology:
  kinds:
    vr-ros:
      image: vrnetlab/mikrotik_routeros:6.47.9

  nodes:
    R01.BRL:
      kind: vr-ros
      mgmt-ipv4: 10.20.30.103
      startup-config: config/r01-brl.rsc
    R01.FRT:
      kind: vr-ros
      mgmt-ipv4: 10.20.30.102
      startup-config: config/r01-frt.rsc
    R01.MSK:
      kind: vr-ros
      mgmt-ipv4: 10.20.30.101
      startup-config: config/r01-msk.rsc
    PC1:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 10.20.30.2
      binds:
        - ./config:/config
      exec:
        - sh /config/pc1.sh
    PC2:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 10.20.30.3
      binds:
        - ./config:/config
      exec:
        - sh /config/pc2.sh
    PC3:
      kind: linux
      image: alpine:latest
      mgmt-ipv4: 10.20.30.4
      binds:
        - ./config:/config
      exec:
        - sh /config/pc3.sh

  links:
    - endpoints: ["R01.BRL:eth1","R01.MSK:eth2"]
    - endpoints: ["R01.BRL:eth2","R01.FRT:eth1"]
    - endpoints: ["R01.MSK:eth1","R01.FRT:eth2"]
    - endpoints: ["R01.BRL:eth3","PC3:eth1"]
    - endpoints: ["R01.FRT:eth3","PC2:eth1"]
    - endpoints: ["R01.MSK:eth3","PC1:eth1"]
```

Затем деплоим с помощью команды `clab deploy -t lab2.clab.yaml`:

<img width="865" height="461" alt="image" src="https://github.com/user-attachments/assets/b9af49ac-0880-431b-96fb-f60f88df8902" />

Строим граф топологии с помощью команды `clab graph -t lab2.clab.yaml`:

<img width="931" height="571" alt="image" src="https://github.com/user-attachments/assets/f12db3fb-5277-4e13-93ee-37f086774d65" />

## Написание конфигов

### Роутеры R01

Прописываем статические руты. DHCP аналогичен R01 из предыдущей работы.

```
/ip address
add address=10.10.11.1/30 interface=ether2
add address=10.10.13.2/30 interface=ether3
add address=10.1.0.1/16 interface=ether4

/ip pool
add name=dhcp-pool ranges=10.1.0.10-10.1.255.254

/ip dhcp-server
add address-pool=dhcp-pool disabled=no interface=ether4 name=dhcp-server

/ip dhcp-server network
add address=10.1.0.0/16 gateway=10.1.0.1

/ip route
add distance=1 dst-address=10.2.0.0/16 gateway=10.10.11.2
add distance=1 dst-address=10.3.0.0/16 gateway=10.10.13.1

/user
add name=sasha password=1234 group=full
remove admin

/system identity
set name=R01.MSK
```

### Компьютеры PC

IP-адреса выдаются компьютерам через DHCP, но в Containerlab у них выставляется дефолтный маршрут от сети управления в yaml-конфиге, из-за чего попытки послать запрос на передаются на роутер моего провайдера, где этот запрос теряется. Поэтому необходимо удалить этот маршрут.

```
#!/bin/sh
ip route del default via 10.20.30.1 dev eth0
udhcpc -i eth1
```

## Ping

### Роутеры R01

Заходим с помощью команды `ssh sasha@clab-lab2-R01.MSK`.

<img width="960" height="628" alt="image" src="https://github.com/user-attachments/assets/654b9079-a38b-49cd-a9a3-1fea0d385308" />

<img width="960" height="628" alt="image" src="https://github.com/user-attachments/assets/73896ae5-9016-4da8-a7bd-517bac2ebb14" />

<img width="960" height="628" alt="image" src="https://github.com/user-attachments/assets/03796481-2eb0-41f9-8a06-9df6832e3574" />

Выходим с помощью команды `quit`.

### Компьютеры PC

Заходим с помощью команды `docker exec -it clab-lab2-PC1 sh`.

<img width="586" height="511" alt="image" src="https://github.com/user-attachments/assets/0538af0b-f1a0-421f-9b0e-c2c434adf7b8" />

<img width="586" height="511" alt="image" src="https://github.com/user-attachments/assets/f0824434-9439-4bc5-ad4e-36b1ccd61ff0" />

<img width="586" height="511" alt="image" src="https://github.com/user-attachments/assets/269d2ad6-4293-46d9-987a-7af170429d77" />

Выходим с помощью команды `exit`.

## Вывод

В ходе работы были созданы:
- все необходимые устройства, которые соединены через настройку IP-адресов на интерфейсах
- DHCP-сервера для компьютеров
- настроена статическая маршрутизация
