# Базовая работа с виртуальной лабораторией PNETLab

## Часть 1. Создание топологии

План топологии следующий:
![](pictures/lab1_net.png)

## Часть 2. Работа с сетевыми устройствами

> Взаимодействие с сетевыми устройствами Cisco IOL происходит с помощью CLI

### Настройка коммутатора

2.1 Зайдём в Switch7 и перейдём в привилегированный режим.

`Switch>enable`
![](pictures/lab1_2_1.png)

2.2 Просмотрим его таблицу коммутации.

`Switch#show mac address-table`
![](pictures/lab1_2_2.png)

2.3 Настроим имя устройства. Перейдём в режим конфигурации.

`Switch#configure terminal`
![](pictures/lab1_2_3.png)

2.4 Укажем имя устройства.

`Switch(config)#hostname Switch7`
![](pictures/lab1_2_4.png)

2.5 Настроим ip адрес для управления Switch7.
```
Switch7(config)#interface vlan 1
Switch7(config-if)#ip address 192.168.1.100 255.255.255.0
Switch7(config-if)#no shutdown
```
![](pictures/lab1_2_5.png)

2.6 Ставим пароль на консоль.
```
Switch7(config)#line console 0
Switch7(config-line)#password VeryStrongPasswd
Switch7(config-line)#login
```
> Надо понимать, что физическая безопасность устройства важный аспект защиты, так как, имея физический доступ к консольному порту, даже не зная пароля его можно сбросить. 

> Этот пароль и все пароли далее устанавливаются в демонстрационных целях. На практике нужно использовать более сильные варианты.

![](pictures/lab1_2_6.png)
![](pictures/lab1_2_6_.png)

2.7 Ставим пароль на привилегированный режим.

`Switch7(config)#enable secret VeryStrongPasswd`
![](pictures/lab1_2_7.png)
![](pictures/lab1_2_7_.png)

2.8 Пароль на консоль (и все остальные, которые могли быть поставлены с помощью `password`) хранятся в открытом виде, поэтому включаем службу шифрования паролей.

`Switch7(config)#service password-encryption`

![](pictures/lab1_2_8.png)

2.9 Создадим учетную запись для пользователя и включим модель аутентификации.
```
Switch7(config)#username user password Qq123456
Switch7(config)#aaa new-model
```
![](pictures/lab1_2_9.png)

2.10 Выйдем из режима конфигурации и сохраним конфиг.
```
Switch7(config)#exit
Switch7#write
```
![](pictures/lab1_2_10.png)

Остальные роутеры 

### Настройка маршрутизатора 

2.11 Зайдём в роутер. После нажатия Enter он спросит, желаете ли автонастройку. Отказываемся.
![](pictures/lab1_2_11.png)

2.12 Повторяем пункты 2.6-2.8 для роутера

2.13 Настроим адрес внутреннего интерфейса е0/1. Это будет адрес шлюза по умолчанию для наших устройств.
```
Router(config)#int e0/1
Router(config-if)#ip address 192.168.1.1 255.255.255.0
Router(config-if)#no shutdown
```
![](pictures/lab1_2_13.png)

2.14 Создаём ACL для будущего NAT. Это значит, что натить роутер будет только подсеть 192.168.1.0/24.
```
Router(config)# access-list 99 permit 192.168.1.0 0.0.0.255 
Router(config)# ip nat inside source list 99 interface e0/0 overload
Router(config)#int e0/0
Router(config-if)#ip nat outside
Router(config)#int e0/1
Router(config-if)#ip nat inside
```
![](pictures/lab1_2_14.png)

## Часть 3. Анализ трафика

Теперь с помощью Wireshark можем отследить, например, ARP трафик.

![](pictures/lab1_3.png)

