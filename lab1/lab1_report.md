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

У меня Мас с процессором Apple Silicon, поэтому у меня возникли трудности и я выполняла работу на Ubuntu на UTM.

Для начала нужно было установить Docker, и в конфигурации lxc контейнера включить security.nesting.

<img width="2206" height="404" alt="telegram-cloud-document-2-5289549823307978141" src="https://github.com/user-attachments/assets/6960dc85-b3dc-46a8-ae45-e32fd0890c4c" />

Затем нужно установить `make`, склонировать `hellt/vrnetlab`, перейти в папку `routeros`, загрузить в эту папку `chr-6.47.9.vmdk` и с помощью `make docker-image` собрать образ.

<img width="1304" height="218" alt="telegram-cloud-document-2-5289549823307978286" src="https://github.com/user-attachments/assets/1055a59c-368a-44f5-be0d-89f92b0178ad" />

Затем установим ContainerLab.

<img width="1872" height="798" alt="telegram-cloud-document-2-5289549823307978312" src="https://github.com/user-attachments/assets/8d5139cb-fe0f-42bd-8fb4-86e4cbcf42c0" />

## Схема сети

<img width="828" height="613" alt="image" src="https://github.com/user-attachments/assets/dea19192-ebc4-43a1-828d-a2a96c54d1f8" />
