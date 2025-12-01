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
