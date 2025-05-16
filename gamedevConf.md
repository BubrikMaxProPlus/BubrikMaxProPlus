# Базовая настройка сети Game Dev студии в PNETLab 

## 1. Сетевая архитектура

![](1_netplan.png)

| Сегмент        | VLAN | Подсеть           | Доступные IP    | Назначение                    |
| -------------- | ---- | ----------------- | --------------- | ----------------------------- |
| Программисты   | 2    | `10.0.0.128/25`   | `.129` – `.254` | Рабочие станции               |
| Администраторы | 6    | `10.0.0.0/28`     | `.1` – `.14`    | Доступ к управлению           |
| Серверы        | 3    | `10.0.2.0/24`     | `.1` – `.254`   | DNS, DHCP, GIT, Syslog и т.д. |
| Интернет / WAN | —    | `192.168.10.0/24` | `.1` – `.254`   | Подключение к внешней сети    |

## 2. Конфигурация оборудования

- Подробнее про работу с оборудованием и конфигурацию оборудования в PNETLab можно посмотреть в вынесенном отдельно модуле **Базовая конфигурация**
- Также в приложение вынесены модули Атака-Защита сетевой инфраструктуры (упрощенного варианта вышеописанной сети), где более подробно разобраны:
	- **CAM-table overflow**
	- **ARP-Spoofing**
	- **VLAN Hopping**
	- **MAC-Spoofing**
 
### Switch7 (основной для доступа)

- Базовая настройка:
```
enable
conf t
hostname Switch7
no ip domain-lookup
enable secret strongpass
service password-encryption
banner motd ^WARNING: Unauthorized access is prohibited!^
```
- VLAN и trunk:
```
vtp domain gamedev
vtp mode transparent

vlan 2
 name Programmers
vlan 3
 name Servers
vlan 6
 name Admins

interface range e0/1, e0/2
 switchport mode trunk
```
- Настройка порта для Win10 (VLAN 2 + Port-Security):
```
interface e1/2
 description Подключение Win10 (VLAN 2 - Programmers)
 switchport mode access
 switchport access vlan 2
 switchport port-security
 switchport port-security maximum 1          ! Только 1 MAC-адрес разрешён
 switchport port-security mac-address sticky ! Автоматически запоминает MAC
 switchport port-security violation restrict ! Нарушения логгируются, но порт не отключается
```

### Switch8 (серверный)
```
hostname Switch8
vtp mode transparent
vtp domain gamedev

vlan 3
 name Servers

interface e1/0
 switchport mode access
 switchport access vlan 3

interface e1/1
 switchport mode access
 switchport access vlan 3

interface e1/2
 switchport mode access
 switchport access vlan 3

logging host 10.0.2.10
```
### Switch9 (дистрибутивный)
```
hostname Switch9
vtp mode transparent
vtp domain gamedev

interface e0/0
 switchport mode trunk

interface e0/1
 switchport mode trunk

interface e0/2
 switchport mode access
 switchport access vlan 6
```
### Switch (7-9)
```
logging buffered 4096 debugging             ! Логирование в буфер памяти
logging host 10.0.2.10                      ! Сетевой syslog-сервер
logging trap informational                  ! Уровень логгирования на удалённый сервер
service timestamps log datetime msec        ! Отметка времени с миллисекундами
```
### Router (межвлан-маршрутизация и NAT)
- Интерфейсы
``` 
interface e0/0
 ip address 192.168.10.254 255.255.255.0
 no shutdown

interface e0/1.2
 encapsulation dot1Q 2
 ip address 10.0.0.1 255.255.255.128

interface e0/1.3
 encapsulation dot1Q 3
 ip address 10.0.2.254 255.255.255.0

interface e0/1.6
 encapsulation dot1Q 6
 ip address 10.0.0.14 255.255.255.240
```
- NAT и доступ в интернет:
```
ip nat inside source list 1 interface e0/0 overload

access-list 1 permit 10.0.0.0 0.0.255.255

interface e0/0
 ip nat outside
interface e0/1
 ip nat inside
```
Дополнительно 
- DHCP (если не на сервере):
```
ip dhcp pool PROGRAMMERS
 network 10.0.0.128 255.255.255.128
 default-router 10.0.0.1
 dns-server 10.0.2.10

ip dhcp pool ADMINS
 network 10.0.0.0 255.255.255.240
 default-router 10.0.0.14
 dns-server 10.0.2.10
```
### pfSense (Firewall + WAN шлюз)

- WAN: e0 — подключение к интернету

- LAN: e1 — 192.168.10.1/24

- Базовые правила firewall:
	- Разрешить:
        - HTTP/HTTPS
		- DNS, SSH, ICMP
        - NAT к внешнему
    - Блокировать:
        - МежVLAN (кроме Admin → Server)
        - Входящие подключения к pfSense

- IDS / IPS:
	- Установить пакет через pfSense Package Manager
	-Настроить:
    	- LAN-интерфейс
    	- Правила: Emerging Threats, Snort GPL
    - Алерты:
        - Port Scan
		- Brute Force
        - Malware Activity
	- Включить blocking mode

- Логирование (настройка):
    - System > Logging -> Remote syslog
        - IP: 10.0.2.10
        - Facility: local0
        - Enable logging firewall events

## 3. Серверы — развертывание служб
Настройка серверов (пример):

### WinServer16 (10.0.2.10)  — SYSLOG + DNS + DHCP (если не на роутере)
#### RSyslog приёмник

- Установка NXLog:
```
Install-PackageProvider -Name NuGet -Force
Install-Module -Name nxlog-ce
```
Конфигурация (`nxlog.conf`):
```
<Input in>
  Module im_udp
  Host 0.0.0.0
  Port 514
</Input>

<Output out>
  Module om_file
  File "C:\\Logs\\switch_logs.log"
</Output>

<Route r>
  Path in => out
</Route>
```
#### DNS Server
- Зона: `gamedev.local`
- DNS-записи:
	- router.gamedev.local -> 10.0.0.1
	- git.gamedev.local -> 10.0.2.20
	- files.gamedev.local -> 10.0.2.30

### WinServer22 (10.0.2.20) — GIT + Jenkins
- GitLab Runner (Docker или EXE)
- Jenkins с интеграцией в AD (если настроено)
- SSH ключи деплоя
- Логирование действий разработчиков

### Linux-сервер (10.0.2.30) — Fail2Ban, rsyslog, rsync
- rsyslog (приём логов с Cisco/pfSense)
- Fail2Ban
- Git mirror / CI
- SFTP

## 4. Security Policy Checklist 

### A. Контроль доступа

| Группа         | VLAN | Доступ к                |
| -------------- | ---- | ----------------------- |
| Программисты   | 2    | Git, Jenkins, интернет  |
| Администраторы | 6    | Полный доступ           |
| Серверы        | 3    | Только по нужным портам |

#### ACL на роутере:
``` 
access-list 100 deny ip 10.0.0.128 0.0.0.127 10.0.2.0 0.0.0.255
access-list 100 permit ip any any

interface e0/1.2
 ip access-group 100 in
```
### B. Аудит и логгирование
- Все устройства логируют на 10.0.2.10
- Используется:
    - login on-success/failure
    - logging trap informational
	- pfSense -> syslog
- Анализ логов через ELK stack (опц.)

### C. Резервное копирование
- WinServer16: резерв DNS, DHCP и SYSLOG каждую ночь на NAS
- Linux-сервер:
	- rsnapshot / rsync на внешний диск
	- Журнал rsyslog сохраняется ежедневно
- Cisco config: `copy running-config tftp` по расписанию через cron или вручную

### D. Обновления
- Windows Server -> WSUS или вручную еженедельно
- Linux -> еженедельно вручную с ипользованием менеджера пакетов 
- Cisco -> регулярная проверка (минимум раз в месяц) и обновление образов
- pfSense -> ручное обновление через GUI

### E. Пароли и аутентификация
- Сложные пароли >= 15 символов
- Ротация каждые 90 дней (опц.)
- Двухфакторная авторизация
- SSH доступ (если есть) — по ключам

### F. IDS/IPS и защита от атак
- Suricata на pfSense
- Fail2Ban на Linux
- Ограничение по MAC-адресу на Switch7/8
- Блокировка неизвестных устройств через port-security

# Приложение