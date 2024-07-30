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
   - Инициализация PKI:
   ```shell
   cd /etc/openvpn
   /usr/share/easy-rsa/easyrsa init pki
   ```
   - Генерирование необходимых серитификатов и ключей для сервера:
   ```shell
   echo 'rasvpn' | /usr/share/easy-rsa/easyrsa build-ca nopass
   echo 'rasvpn' | /usr/share/easy-rsa/easyrsa gen-req server nopass
   echo 'yes' | /usr/share/easy-rsa/easyrsa sign-req server server
   /usr/share/easy-rsa/easyrsa gen-dh
   openvpn --genkey --secret ta.key
   ```
   - Генерирование сертификатов для клиента:
   ```shell
   echo 'client' | /usr/share/easy-rsa/easyrsa gen-req client nopass
   echo 'yes' | /usr/share/easy-rsa/easyrsa sign-req client client
   ```
   - Подготовка на сервере конфигурационного файла сервиса OpenVPN:
   ```shell
   nano /etc/openvpn/server.conf
   ...
   cat /etc/openvpn/server.conf
   port 1207
   proto udp
   dev tun
   ca /etc/openvpn/pki/ca.crt
   cert /etc/openvpn/pki/issued/server.crt
   key /etc/openvpn/pki/private/server.key
   dh /etc/openvpn/pki/dh.pem
   server 10.10.10.0 255.255.255.0
   push "route 192.168.56.0 255.255.255.0"
   ifconfig-pool-persist ipp.txt
   client-to-client
   client-config-dir /etc/openvpn/client
   keepalive 10 120
   comp-lzo
   persist-key
   persist-tun
   status /var/log/openvpn-status.log
   log /var/log/openvpn.log
   verb 3
   ```
   - Задание параметра iroute для клиента:
   ```shell
   echo 'iroute 192.168.56.0 255.255.255.0' > /etc/openvpn/client/client
   ```
   - Копирование указанных ниже файлов сертификатов и ключа на клентскую машину:
   ```shell
   /etc/openvpn/pki/ca.crt
   /etc/openvpn/pki/issued/client.crt
   /etc/openvpn/pki/private/client.key

   calculate dem # ls -al /etc/openvpn
   итого 64
   drwxr-xr-x 1 root root   144 июл 28 12:58 .
   drwxr-xr-x 1 root root  3948 июл 30 11:51 ..
   -rw------- 1 dem  guest 1184 июл 28 12:38 ca.crt
   -rw-r--r-- 1 root root   160 июл 28 12:58 client.conf
   -rw------- 1 dem  guest 4471 июл 28 12:45 client.crt
   -rw------- 1 dem  guest 1704 июл 28 12:44 client.key
   -rwxr-xr-x 1 root root   943 июн  6 13:07 down.sh
   -rw-r--r-- 1 root root     0 июн  6 13:07 .keep_net-vpn_openvpn-0
   -rwxr-xr-x 1 root root  2865 июн  6 13:07 up.sh
   ```
   - Подготовка на клиенте конфигурационного файла сервиса OpenVPN:
   ```shell
   nano /etc/openvpn/client.conf
   ...
   cat /etc/openvpn/client.conf
   dev tun
   proto udp
   remote 192.168.56.10 1207
   client
   resolv-retry infinite
   ca ./ca.crt
   cert ./client.crt
   key ./client.key
   persist-key
   persist-tun
   comp-lzo
   verb 3
   ```
   - Подлючение к серверу OpenVPN c хостовой машины:
   ```shell
   calculate dem # cd /etc/openvpn/
   calculate openvpn # openvpn --config client.conf
   2024-07-30 13:00:18 WARNING: Compression for receiving enabled. Compression has been used in the past to break encryption. Sent packets are not compressed unless "allow-compression yes" is also set.
   2024-07-30 13:00:18 Note: --cipher is not set. OpenVPN versions before 2.5 defaulted to BF-CBC as fallback when cipher negotiation failed in this case. If you need this fallback please add '--data-ciphers-fallback BF-CBC' to your configuration and/or add BF-CBC to --data-ciphers.
   2024-07-30 13:00:18 OpenVPN 2.6.9 x86_64-pc-linux-gnu [SSL (OpenSSL)] [LZO] [LZ4] [EPOLL] [MH/PKTINFO] [AEAD]
   2024-07-30 13:00:18 library versions: OpenSSL 3.0.13 30 Jan 2024, LZO 2.10
   2024-07-30 13:00:18 WARNING: No server certificate verification method has been enabled.  See http://openvpn.net/howto.html#mitm for more info.
   2024-07-30 13:00:18 TCP/UDP: Preserving recently used remote address: [AF_INET]192.168.56.10:1207
   2024-07-30 13:00:18 Socket Buffers: R=[212992->212992] S=[212992->212992]
   2024-07-30 13:00:18 UDPv4 link local: (not bound)
   2024-07-30 13:00:18 UDPv4 link remote: [AF_INET]192.168.56.10:1207
   2024-07-30 13:00:18 TLS: Initial packet from [AF_INET]192.168.56.10:1207, sid=c5707599 e1e34c53
   2024-07-30 13:00:18 VERIFY OK: depth=1, CN=rasvpn
   2024-07-30 13:00:18 VERIFY OK: depth=0, CN=rasvpn
   2024-07-30 13:00:18 Control Channel: TLSv1.3, cipher TLSv1.3 TLS_AES_256_GCM_SHA384, peer certificate: 2048 bits RSA, signature: RSA-SHA256, peer temporary key: 253 bits X25519
   2024-07-30 13:00:18 [rasvpn] Peer Connection Initiated with [AF_INET]192.168.56.10:1207
   2024-07-30 13:00:18 TLS: move_session: dest=TM_ACTIVE src=TM_INITIAL reinit_src=1
   2024-07-30 13:00:18 TLS: tls_multi_process: initial untrusted session promoted to trusted
   2024-07-30 13:00:18 PUSH: Received control message: 'PUSH_REPLY,route 10.10.10.0 255.255.255.0,topology net30,ping 10,ping-restart 120,ifconfig 10.10.10.6 10.10.10.5,peer-id 0,cipher AES-256-GCM'
   2024-07-30 13:00:18 OPTIONS IMPORT: --ifconfig/up options modified
   2024-07-30 13:00:18 OPTIONS IMPORT: route options modified
   2024-07-30 13:00:18 net_route_v4_best_gw query: dst 0.0.0.0
   2024-07-30 13:00:18 net_route_v4_best_gw result: via 192.168.55.1 dev wlp2s0
   2024-07-30 13:00:18 ROUTE_GATEWAY 192.168.55.1/255.255.255.0 IFACE=wlp2s0 HWADDR=4c:d5:77:7b:e2:87
   2024-07-30 13:00:18 TUN/TAP device tun0 opened
   2024-07-30 13:00:18 net_iface_mtu_set: mtu 1500 for tun0
   2024-07-30 13:00:18 net_iface_up: set tun0 up
   2024-07-30 13:00:18 net_addr_ptp_v4_add: 10.10.10.6 peer 10.10.10.5 dev tun0
   2024-07-30 13:00:18 net_route_v4_add: 10.10.10.0/24 via 10.10.10.5 dev [NULL] table 0 metric -1
   2024-07-30 13:00:18 Initialization Sequence Completed
   2024-07-30 13:00:18 Data Channel: cipher 'AES-256-GCM', peer-id: 0, compression: 'lzo'
   2024-07-30 13:00:18 Timers: ping 10, ping-restart 120
   ...
   calculate dem # ip r
   default via 192.168.55.1 dev wlp2s0 proto dhcp src 192.168.55.251 metric 600 
   10.10.10.0/24 via 10.10.10.5 dev tun0 
   10.10.10.5 dev tun0 proto kernel scope link src 10.10.10.6 
   127.0.0.0/8 via 127.0.0.1 dev lo 
   192.168.55.0/24 dev wlp2s0 proto kernel scope link src 192.168.55.251 metric 600 
   192.168.56.0/24 dev vboxnet0 proto kernel scope link src 192.168.56.1 

   calculate dem # ping -c 4 10.10.10.1
   PING 10.10.10.1 (10.10.10.1) 56(84) bytes of data.
   64 bytes from 10.10.10.1: icmp_seq=1 ttl=64 time=0.389 ms
   64 bytes from 10.10.10.1: icmp_seq=2 ttl=64 time=0.616 ms
   64 bytes from 10.10.10.1: icmp_seq=3 ttl=64 time=0.327 ms
   64 bytes from 10.10.10.1: icmp_seq=4 ttl=64 time=0.577 ms

   --- 10.10.10.1 ping statistics ---
   4 packets transmitted, 4 received, 0% packet loss, time 3030ms
   rtt min/avg/max/mdev = 0.327/0.477/0.616/0.122 ms
   ```
