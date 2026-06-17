# demo
# ==Установка==

Выбрать ручную установку -> Создать файловую систему -> Файловая система Ext4

Включить "Дополнительные приложения":
- Поддержка клиентской инфраструктуры Samba AD
- Поддержка работы в виртуальных окружениях
- Поддержка управления через web-интерфейс

## ==После клонирования==

Адаптеры на "внутренняя сеть" выдавая "имя" по схеме.

# ==Задание 1. Выдача IP-адресов и имен устройств==

> [!info]
>[ipmeter.ru](https://ipmeter.ru/) - Калькулятор ip-адресов

Вход под sudo через пользователя производится через команду:
```
su
```

Также можно войти сразу через пользователя `root`.

Далее вводится пароль пользователя, которого мы создали при установке.
## ==Выдача hostname==

Необходимо выдать всем машинам имена.

```
hostnamectl set-hostname [Имя машины] 
exec bash # для применения и сохранения настроек
```

## ==Выдача IP-адресов==

> [!NOTE]
> Если вы зашли через `su`, то перед всей настройкой необходимо зайти в файл **`/etc/sudoers`** и раскомментировать строчку с правами root, выдав тем самым ему абсолютно все права и сохранить файл. (Используется позже, но лучше сделать заранее)

На всех машинах необходимо перейти в файл **`/etc/net/sysctl.conf`** и заменить строку и применить:

```
#Заменить строку
net.ipv4.ip_forward=1 
#Применить изменения
sysctl -p
```

Для выдачи ip-адресов необходимо перейти в каталог **`/etc/net/ifaces`** и создать необходимые каталоги интерфейсов, которые мы создали.

```
cd /etc/net/ifaces
mkdir [название интерфейса]
```

Далее в директориях создать файл **`options`** с содержанием:

```
DISABLED=no
TYPE=eth
BOOTPROTO=static
CONFIG_IPV4=yes
```

Создать файл **`ipv4address`** с содержанием:

```
[ip-адрес]/[маска подсети]
```

Далее на HQ-RTR, BR-RTR, HQ-SRV, HQ-CLI, BR-SRV, создаём файл **`ipv4route`** на интерфейсах, которые ведут в машину выше:

```
default via [ip-адрес машины выше]
```

Обязательно в конце настройки прописать:

```
systemctl restart network
```

# ==Задание 2. Получение доступа в интернет для машин HQ-RTR, BR-RTR==

Необходимо ввести правила для iptables на машине **`ISP`**:

```
iptables -t nat -A POSTROUTING -o enp0s3 -s [IP HQ]/[маска] -j MASQUERADE
iptables -t nat -A POSTROUTING -o enp0s3 -s [IP BR]/[маска] -j MASQUERADE
```

Далее необходимо сохранить правила и включить **`iptables`** в автозагрузку:

```
iptables-save > /etc/sysconfig/iptables
systemctl enable --now iptables
```

Далее, в идеале, так-же перезапустить **`systemctl restart network`** или перезапустить машину. Проверить работоспособность можно с помощью:

```
ping 8.8.8.8
```

По аналогии с настройкой выше так-же можно настроить интернет для остальных машин для последующих заданий.

# ==Задание 3. GRE-туннель на HQ-RTR и BR-RTR==

Создать директорию **`/etc/net/ifaces/gre1`**. 

```
mkdir /etc/net/ifaces/gre1 | cd /etc/net/ifaces/gre1
```

Далее в директории **`gre1`**. Создаём файл **`ipv4address`** туда записываем:

```
172.17.0.1 peer 172.17.0.2 
```

На второй машине в этом файле должно быть:

```
172.17.0.2 peer 172.17.0.1
```

После создаём файл **`options`** в директории **`gre1`** и прописываем туда конфиг:

```
TUNLOCAL=[IP HQ]
TUNREMOTE=[IP BR]
TUNTYPE=gre
TYPE=iptun
TUNTTL=64
TUNMTU=1476
TUNOPTIONS='ttl 64'
DISABLED=no
```

>[!Важно]
>IP HQ и IP BR прописываются те, которые ведут в сторону ISP. При настройке на второй машине необходимо поменять эти ip-адреса местами.

Как прописали всё в файлы, необходимо поменять правила фильтрации трафика:

```
iptables -I INPUT -i enp0s3 -p 47 -j ACCEPT
iptables -I OUTPUT -o enp0s3 -p 47 -j ACCEPT
```

Далее прописываем и проверяем правильность настройки:

```
ifup gre1
ifconfig gre1
ip tu sh
ip a
```

Далее настройка аналогичная. После настройки можно проверить всё командами(они должны пинговаться):

```
ping 172.17.0.1
ping 172.17.0.2
```

# ==Задание 4. SSH и пользователи на HQ-SRV и BR-SRV==

Также по заданию просят создать ещё пользователей для RTR машин, создать их можно по аналогии с тем, что описано ниже.

Для начала необходимо создать пользователя и назначить ему PID. Имя и PID пользователя даётся по заданию.  

```
useradd [Имя пользователя] -u [PID]
```

Задаем пароль:

```
passwd [Имя пользователя]
```

Добавляем в группу:

```
usermod -aG wheel [Имя пользователя]
```

Добавляем строку в **`/etc/sudoers`**, чтобы наш пользователь мог использовать команды **``sudo``** без ввода пароля админа:

```
[Имя пользователя] ALL=(ALL) NOPASSWD:ALL
```

И перед подключением по заданию необходимо настроить конфиг поменяв значения в строчках **`/etc/openssh/sshd_config`**:

```
Port [Порт по заданию]
MaxAuthTries [Количество попыток по заданию]
PasswordAuthentication yes
Banner /etc/openssh/bannermotd # расскоментировать строчку и прописать этот путь
AllowUsers  [Пользователь] # вместо пробела используется Tab
```

Далее создаём файл баннера, который мы прописали в конфиге:

```
nano /etc/openssh/bannermotd
```

И прописываем туда содержимое, требуемое по заданию, сохраняем и перезагружаем службу:

```
systemctl restart sshd
```

Можно подключаться. 

```
ssh -p [порт] [пользователь]@[ip-адрес]
```

# ==Задание 5. OSPF на HQ-RTR и BR-RTR==

Перед началом работы необходимо прописать:

```
apt-get update
```

После того, как всё прошло, необходимо установить **`frr`**, для работы с OSPF:

```
apt-get install frr -y
```

Далее необходимо включить демон **`ospfd`** поменяв в файле **`no`** на **`yes`**:

```
nano /etc/frr/daemons
```

Теперь перезапустим службу и включим OSPF в автозагрузку:

```
systemctl restart frr | systemctl enable --now frr
```

Теперь переходим к работе с ospf с помощью команды:

```
vtysh
```

Далее прописываем: 

```vtysh
conf t
router ospf
ospf router-id 1.1.1.1(2.2.2.2 для другой)
# необходимо брать полную подсеть, кроме gre, то есть пример: 172.16.4.0/28
network [ip-адрес]/[маска] area 0(повторить для всех ip-адресов на машине, даже gre)
exit
int tun0
ip ospf authentication
ip ospf authentication-key DEMO
exit
exit
write
```

Только после того, как все команды будут введены на обоих машинах необходимо будучи во **`vtysh`** проверить работу прописав:

```vtysh
sh ip ospf nei
```

Должны появиться строки обозначающие, что всё работает, если строки нету, значит появилась какая-то ошибка. Легче всего проверить будет через конфиг, который после настройки должен иметь вид:

```/etc/frr/frr.conf
frr version 10.2.2
frr defaults traditional
hostname [ваш домен]
log file /var/log/frr/frr.log
no ipv6 forwarding
!
interface tun0
 ip ospf authentication
 ip ospf authentication-key DEMO
exit
!
router ospf
 network [ip-адрес]/[маска] area 0
 network [ip-адрес]/[маска] area 0
 network 172.17.0.1/32 area 0
 network 172.17.0.2/32 area 0
exit
!
```

После переписывания конфига необходимо прописать: 

```
systemctl reload frr | systemctl restart frr
```

# ==Задание 6. Настройка DHCP на HQ-RTR==

Чтобы начать работать с DHCP необходимо установить его:

```
apt-get install dhcp-server -y
cp /etc/dhcp/dhcpd.conf.sample /etc/dhcp/dhcpd.conf
```

И дальнейшая настройка будет проходить в **`dhcpd.conf`**, который мы только что создали:

```
ddns-update-style none;

subnet [ip-адрес(вся подсеть)] netmask [маска подсети] {
    option routers                  [IP-адрес маршрутизатора]; 
    option subnet-mask              [маска подсети]; 
	
    option nis-domain               "[domain.org]"; 
    option domain-name              "[domain.org]"; 
    option domain-name-servers      [IP-адрес сервера];
	
    range dynamic-bootp 10.0.0.2 10.0.0.14; #диапазон DHCP-подсети
    default-lease-time 21600;
    max-lease-time 43200;
}
```

После прописывания конфига необходимо указать на каком интерфейсе будет работать наш DHCP. Прописывается это в файле **`/etc/sysconfig/dhcpd`**

```
DHCPDARGS=[Интерфейс HH-CLI]
```

Далее делаем автозапуск DHCP и стартуем его: 

```
sudo chkconfig dhcpd on | systemctl start dhcpd
```

Проверить полученный адрес можно с помощью команд:

```
ip -br a
cat /etc/resolv.conf # Должен быть домен
```

# ==Задание 7. Настройка часового пояса==

Необходимо просто ввести команду и проверить на всех устройствах:

```
timedatectl set-timezone Europe/Moscow
timedatectl status
```

# ==Задание 8. MariaDB & MediaWiki в Docker на BR-SRV==

Для начала работы с заданием необходимо установить сам **`Docker`** и добавить пользователя в группу, для работы с ним.

```
apt-get update
apt-get install docker-engine docker-compose-plugin
usermod -aG docker [пользователь] # Сюда вводить пользлователя под который настраиваете, даже если это root
systemctl enable --now docker
```

Далее необходимо проверить то, что всё установилось.

```
docker --version
docker compose --version
```

Дальше создаём и переходим в каталог в котором будет работа:

```
mkdir ~/mediawiki
cd ~/mediawiki
```

В этом каталоге создаём файл **`docker-compose.yml`** в который записываем:

```
services:
	db:
		image: mariadb
		container_name: mariadb
		environment:
			MYSQL_ROOT_PASSWORD: rootpass
			MYSQL_DATABASE: mediawiki
			MYSQL_USER: wiki
			MYSQL_PASSWORD: WikiP@ssw0rd
		volumes:
			- db: /var/lib/mysql
		restart: always
	mediawiki:
		image: mediawiki
		container_name: wiki
		ports:
			- 8080:80
#		volumes:
#			- ./LocalSettings.php: /var/www/html/LocalSettings.php
		restart: always
		links:
			- db
volumes:
	db_data:
	db:
```

Как прописали файл необходимо поднять контейнер.

```
docker compose up -d
```

Далее начнётся **`pulling`** компонентов, ведь их у нас нету локально. Если **`pulling`** начался, но ничего не происходит необходимо отменить и снова прописать ту же команду.
# ==Задание 9. Файловое хранилище на HQ-SRV== 

Необходимо выключить машину и в настройках подключить 3 виртуальных жёстких диска.

После подключения включаем машину, входим и проверяем с помощью команды. Должно появиться sdb, sdc, sdd. Могут быть и другие имена:

```
lsblk
```
 
Далее обнуляем суперблоки для добавления дисков и удаляем старые метаданные и подпись на дисках.

```
mdadm --zero-superblock --force /dev/sd{b,c,d}
wipefs --all --force /dev/sd{b,c,d}
```

Теперь создаём **`RAID`**

```
mdadm --create /dev/md0 -l 5 -n 3 /dev/sd{b,c,d}
```

Теперь необходимо проверить вошли ли диски в группу.

```
lsblk
```

В выводе команды, под записями о наших дисках должна появится подпись **`md0`**. 

Далее необходимо создать файловую систему из созданного массива.

```
mkfs -t ext4 /dev/md0
```

После необходимо создать файл-конфигурации и его директорию.

```
mkdir /etc/mdadm
echo "DEVICE partitions" > /etc/mdadm/mdadm.conf
mdadm --detail --scan | awk '/ARRAY/ {print}' >> /etc/mdadm/mdadm.conf 
```

После заполнения файла конфигурации необходимо создать файловую систему и монтировать массив.

```
mkdir /mnt/raid5
echo "/dev/md0 /mnt/raid5 ext4 defaults 0 0" >> /etc/fstab
mount -a
```

Когда мы ввели команды необходимо проверить результат.

```
df -h | grep raid5
```

Дальше устанавливаем необходимые пакеты.

```
apt-get install nfs-{server,utils} -y
```

После установки создаём директорию для общего доступа и выдаём ей права.

```
mkdir /mnt/raid5/nfs
chmod 766 /mnt/raid5/nfs
```

В файл `**/etc/exports**` добавляем строку:

```
/mnt/raid5/nfs [ip]/[маска](rw,no_root_squash)
```

Экспортируем файловую систему и добавляем в автозагрузку.

```
exportfs -arv
systemctl enable --now nfs-server
```

# ==Задание 10. chrony на HQ-RTR==

Для начала необходимо установить сам **`chrony`**.

```
apt-get install chrony -y
```

...
