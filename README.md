# VPN
1. Настроить VPN между двумя виртуальными машинами (ВМ) в tun/tap режимах, замерить скорость в туннелях, сделать вывод об отличающихся показателях.
2. Поднять RAS на базе OpenVPN с клиентскими сертификатами, подключиться с локальной машины на ВМ.
### Исходные данные ###
&ensp;&ensp;ПК на Linux c 8 ГБ ОЗУ или виртуальная машина (ВМ) с включенной Nested Virtualization.<br/>
&ensp;&ensp;Предварительно установленное и настроенное ПО:<br/>
&ensp;&ensp;&ensp;Hashicorp Vagrant (https://www.vagrantup.com/downloads);<br/>
&ensp;&ensp;&ensp;Oracle VirtualBox (https://www.virtualbox.org/wiki/Linux_Downloads).<br/>
&ensp;&ensp;&ensp;Ansible (https://docs.ansible.com/ansible/latest/installation_guide/intro_installation.html).<br/>
&ensp;&ensp;Все действия проводились с использованием Vagrant 2.4.0, VirtualBox 7.0.18, Ansible 9.4.0 и образа ubuntu/jammy64 версии 20240301.0.0. <br/>
### Ход решения ###
1. Настройка VPN между двумя ВМ.
   - Установка необходимого ПО. В частности, на оба хоста устанавливается пакет OpenVPN и программа для измерения пропускной способности сети iperf3:
  ```shell
  apt update
  apt install -y openvpn iperf3
  ```
  - Генерирование на сервере ключа:
  ```shell
  openvpn --genkey secret /etc/openvpn/static.key
  ```
  - Подготовка на сервере конфигурационного файла сервиса OpenVPN:
  ```shell
  nano /etc/openvpn/server.conf
  ...
  cat /etc/openvpn/server.conf
  dev tap
  ifconfig 10.10.10.1 255.255.255.0
  topology subnet
  secret /etc/openvpn/static.key
  comp-lzo
  status /var/log/openvpn-status.log
  log /var/log/openvpn.log
  verb 3
  ```
  - Подготовка на сервере systemd модуля для запуска сервиса OpenVPN и его запуск:
  ```shell
  nano /etc/systemd/system/openvpn@.service
  ...
  cat /etc/systemd/system/openvpn@.service
  [Unit]
  Description=OpenVPN Tunneling Application On %I
  After=network.target
  [Service]
  Type=notify
  PrivateTmp=true
  ExecStart=/usr/sbin/openvpn --cd /etc/openvpn/ --config %i.conf
  [Install]
  WantedBy=multi-user.target

  systemctl start openvpn@server
  systemctl enable openvpn@server
  ```
  - Под
