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

#nano /etc/hosts 

127.0.0.1       localhost
192.168.0.11       msi
192.168.0.12       msi2
192.168.1.11    msi
192.168.1.12    msi2

# The following lines are desirable for IPv6 capable hosts
::1     localhost ip6-localhost ip6-loopback
ff02::1 ip6-allnodes
ff02::2 ip6-allrouters


# Установка 
sudo apt install drbd-utils
sudo apt install lvm2

Подключаем DRBD к модулям ядра: sudo modprobe drbd
Добавляем в загрузки системы: echo “drbd” >> /etc/modules
Установка должна быть на двух машинах.

sudo service drbd start
# Для выполнения самого накопителя требуется добавить новый диск к виртуальной машине
sudo fdisk /dev/sdb
n — создание диска — либо primary, либо extension.Создаем primary — ключ p.Остальное по вашему усмотрению — по умолчанию.Enter — Enter готово.
Создаем логические разделы: 
pvcreate /dev/sdb1
vgcreate vg0 /dev/sdb1
lvcreate -L3G -n www vg0
lvcreate -L3G -n mysql vg0
# Создаём файлы конфигурации /etc/drbd.d/www.res и /etc/drbd.d/mysql.res
esource www {
    protocol C;
    disk {
        fencing resource-only;
    }
    handlers {
        fence-peer
"/usr/lib/drbd/crm-fence-peer.sh";
        after-resync-target
"/usr/lib/drbd/crm-unfence-peer.sh";
    }
syncer {
       rate 110M;
    }
    on msi2
    {
       device /dev/drbd2;
       disk /dev/vg0/www;
       address 192.168.1.12:7794;
       meta-disk internal;
    }
    on msi
    {
       device /dev/drbd2;
       disk /dev/vg0/www;
       address 192.168.1.11:7794;
       meta-disk internal;
    }
}

и 

resource mysql {
    protocol A;
    disk {
        fencing resource-only;
    }
    handlers {
        fence-peer
"/usr/lib/drbd/crm-fence-peer.sh";
        after-resync-target
"/usr/lib/drbd/crm-unfence-peer.sh";
    }
syncer {
       rate 110M;
    }
    on msi2
    {
       device /dev/drbd1;
       disk /dev/vg0/mysql;
       address 192.168.1.12:7798;
       meta-disk internal;
    }
    on msi
    {
       device /dev/drbd1;
       disk /dev/vg0/mysql;
       address 192.168.1.11:7798;
       meta-disk internal;
    }
}

# После чего на обоих серверах выполняем:
drbdadm create-md www
drbdadm create-md mysql
drbdadm up all

на первой машине
drbdadm primary --force www
drbdadm primary --force mysql

на второй
drbdadm secondary  www 
sudo drbdadm down/up all
sudo drbdadm status
sudo drbdmon

# Форматируем диски 
sudo drbdadm primary www
sudo drbdadm primary mysql

sudo mkfs.ext4 /dev/drbd1
sudo mkfs.ext4 /dev/drbd2
mount /dev/drbd1 /mnt/www
mount /dev/drbd2 /mnt/mysql
