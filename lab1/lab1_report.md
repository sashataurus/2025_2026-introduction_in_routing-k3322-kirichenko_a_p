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

<img width="828" height="613" alt="image" src="https://github.com/user-attachments/assets/dea19192-ebc4-43a1-828d-a2a96c54d1f8" />

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




