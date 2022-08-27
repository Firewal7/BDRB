# Отказоустойчивость:Pacemaker BDRB
# Интерфейс на Вирт машинех
Сеть: 1) Сетевой мост Wifi 2) Внутреняя сеть 3) Внутреняя сеть

# /etc/network/interfaces
auto lo
iface lo inet loopback

auto enp0s3
iface enp0s3 inet dhcp

auto enp0s8
iface enp0s8 inet static
address 192.168.0.11/24

auto enp0s9
iface enp0s9 inet static
address 192.168.1.11/24

sudo ifdown; ifup enps08
sudo ifdown; ifup enp0s9

Интерфейс enp0s3 служит для доступа к отдельным нодам кластера.
enp0s8 — это heartbeat-интерфейс. 
enp0s9 используется для синхронизации DRBD-ресурсов.


# Установка sudo apt install drbd-utils

Подключаем DRBD к модулям ядра: sudo modprobe drbd
Добавляем в загрузки системы: echo “drbd” >> /etc/modules
Установка должна быть на двух машинах.
