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
   - Подготовка на клиенте конфигурационного файла сервиса OpenVPN:
   ```shell
   nano /etc/openvpn/server.conf
   ...
   cat /etc/openvpn/server.conf
   dev tap
   remote 192.168.56.10
   ifconfig 10.10.10.2 255.255.255.0
   topology subnet
   route 192.168.56.0 255.255.255.0
   secret /etc/openvpn/static.key 
   comp-lzo
   status /var/log/openvpn-status.log
   log /var/log/openvpn.log
   verb 3
   ```
   - На клиенте, в директорию /etc/openvpn/ копируется ключ static.key, сгенерированный ранее на сервере.
   - На клиенте создаётся аналогичный systemd unit для запуска сервиса OpenVPN и осуществляется его перезапуск.
   - Проведение замеров скорости в туннеле для режима TAP:<br/>
     На сервере запускается iperf3 в режиме сервера
     ```shell
     iperf3 -s &
     ```
     На клиенте запускается iperf в режиме клиента
     ```shell
     iperf3 -c 10.10.10.1 -t 40 -i 5
     Connecting to host 10.10.10.1, port 5201
     [  5] local 10.10.10.2 port 46056 connected to 10.10.10.1 port 5201
     [ ID] Interval           Transfer     Bitrate         Retr  Cwnd
     [  5]   0.00-5.00   sec   116 MBytes   195 Mbits/sec   22    179 KBytes       
     [  5]   5.00-10.00  sec   118 MBytes   198 Mbits/sec   26    204 KBytes       
     [  5]  10.00-15.00  sec   117 MBytes   196 Mbits/sec   22    170 KBytes       
     [  5]  15.00-20.00  sec   108 MBytes   181 Mbits/sec   40    125 KBytes       
     [  5]  20.00-25.00  sec   106 MBytes   177 Mbits/sec   42    114 KBytes       
     [  5]  25.00-30.00  sec   111 MBytes   186 Mbits/sec   14    206 KBytes       
     [  5]  30.00-35.00  sec   114 MBytes   192 Mbits/sec   10    219 KBytes       
     [  5]  35.00-40.00  sec   114 MBytes   190 Mbits/sec   19    194 KBytes       
     - - - - - - - - - - - - - - - - - - - - - - - - -
     [ ID] Interval           Transfer     Bitrate         Retr
     [  5]   0.00-40.00  sec   904 MBytes   190 Mbits/sec  195             sender
     [  5]   0.00-40.24  sec   903 MBytes   188 Mbits/sec                  receiver

     iperf Done.
     ```
     - В конфигурационных файлах сервера и клиента /etc/openvpn/server.conf директива dev устанавливается в tun. И далее, аналогично предыдущему пункту, осуществляются замеры скорости в туннеле при использовании режима TUN:
     ```shell
     iperf3 -c 10.10.10.1 -t 40 -i 5
     Connecting to host 10.10.10.1, port 5201
     [  5] local 10.10.10.2 port 57688 connected to 10.10.10.1 port 5201
     [ ID] Interval           Transfer     Bitrate         Retr  Cwnd
     [  5]   0.00-5.00   sec   119 MBytes   199 Mbits/sec   38    237 KBytes       
     [  5]   5.00-10.00  sec   117 MBytes   196 Mbits/sec   25    207 KBytes       
     [  5]  10.00-15.00  sec   115 MBytes   193 Mbits/sec   21    149 KBytes       
     [  5]  15.00-20.00  sec   115 MBytes   193 Mbits/sec   23    221 KBytes       
     [  5]  20.00-25.00  sec   114 MBytes   192 Mbits/sec   29    145 KBytes       
     [  5]  25.00-30.00  sec   114 MBytes   191 Mbits/sec   23    211 KBytes       
     [  5]  30.00-35.00  sec   114 MBytes   192 Mbits/sec   28    145 KBytes       
     [  5]  35.00-40.00  sec   113 MBytes   190 Mbits/sec   26    192 KBytes       
     - - - - - - - - - - - - - - - - - - - - - - - - -
     [ ID] Interval           Transfer     Bitrate         Retr
     [  5]   0.00-40.00  sec   921 MBytes   193 Mbits/sec  213             sender
     [  5]   0.00-40.18  sec   921 MBytes   192 Mbits/sec                  receiver

     iperf Done.
     ```
     - Результаты замеров скорости в туннелях с использованием режимов TAP и TUN оказались практически одинаковыми. Стоит отметить, что TUN и TAP являются драйверами виртуальных сетевых устройств. В режиме TAP эмулируется Ethernet-устройство и работа осуществляется на канальном уровне модели OSI с использованием ethernet-кадров, а в режиме TUN работа осуществляется на сетевом уровне модели OSI с использованием IP-пакетов. Логика работы указанных режимов говорит о том, что на практике TUN должен иметь более высокую производительность из-за отсутствия накладных расходов, таких как, например, обработка широковещательных адресов, характерных для режима TAP.
2. Настройка RAS на базе OpenVPN.
   - Установка необходимого ПО на сервер. В частности, пакет easy-rsa необходим для развёртывания центра серитификации:
   ```shell
   apt update
   apt install -y openvpn easy-rsa
   ```
   - Инициализация PKI, генерирование необходимых ключей и сертификатов:
   ```shell
   
   ```
