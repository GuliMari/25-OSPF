# 25-OSPF:
1. Развернуть 3 виртуальные машины
2. Объединить их разными vlan
- настроить OSPF между машинами на базе Quagga;
- изобразить ассиметричный роутинг;
- сделать один из линков "дорогим", но что бы при этом роутинг был симметричным.

## настроить OSPF между машинами на базе Quagga

Следуя инструкциям методички развернем 3 сервера с помощью vagrant и ansible.
Зайдем на `router1` и проверим доступность сетей с хоста 192.168.30.1:

```bash
root@router1:~# ping 192.168.30.1
PING 192.168.30.1 (192.168.30.1) 56(84) bytes of data.
64 bytes from 192.168.30.1: icmp_seq=1 ttl=64 time=1.22 ms
64 bytes from 192.168.30.1: icmp_seq=2 ttl=64 time=0.926 ms
64 bytes from 192.168.30.1: icmp_seq=3 ttl=64 time=1.07 ms
64 bytes from 192.168.30.1: icmp_seq=4 ttl=64 time=0.659 ms
...
```

Запустим трассировку:

```bash
root@router1:~# traceroute 192.168.30.1
traceroute to 192.168.30.1 (192.168.30.1), 30 hops max, 60 byte packets
 1  192.168.30.1 (192.168.30.1)  1.833 ms  1.750 ms  1.695 ms
```
 
 Посмотрим маршруты из интерфейса `vtysh`:
 
```bash
router1# show ip route ospf
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/30 [110/100] is directly connected, enp0s8, weight 1, 00:00:17
O>* 10.0.11.0/30 [110/200] via 10.0.10.2, enp0s8, weight 1, 00:00:12
  *                        via 10.0.12.2, enp0s9, weight 1, 00:00:12
O   10.0.12.0/30 [110/100] is directly connected, enp0s9, weight 1, 00:00:52
O   192.168.10.0/24 [110/100] is directly connected, enp0s10, weight 1, 00:00:52
O>* 192.168.20.0/24 [110/200] via 10.0.10.2, enp0s8, weight 1, 00:00:12
O>* 192.168.30.0/24 [110/200] via 10.0.12.2, enp0s9, weight 1, 00:00:17
```
Отключаем прямое соединение между 1 и 3 роутером и смотрим, как изменится маршрут:

```bash
root@router1:~# ifconfig enp0s9 down
root@router1:~# traceroute 192.168.30.1
traceroute to 192.168.30.1 (192.168.30.1), 30 hops max, 60 byte packets
 1  10.0.10.2 (10.0.10.2)  0.537 ms  0.495 ms  0.470 ms
 2  192.168.30.1 (192.168.30.1)  1.797 ms  1.779 ms  1.703 ms
```
 Видно, что трафик пошел через 2 роутер.
 
 ## изобразить ассиметричный роутинг
 
 Поменяем стоимость интерфейса enp0s8 на router1:
 ```bash
 router1# show ip route ospf 
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/30 [110/300] via 10.0.12.2, enp0s9, weight 1, 00:00:14
O>* 10.0.11.0/30 [110/200] via 10.0.12.2, enp0s9, weight 1, 00:00:14
O   10.0.12.0/30 [110/100] is directly connected, enp0s9, weight 1, 00:00:14
O   192.168.10.0/24 [110/100] is directly connected, enp0s10, weight 1, 00:00:54
O>* 192.168.20.0/24 [110/300] via 10.0.12.2, enp0s9, weight 1, 00:00:14
O>* 192.168.30.0/24 [110/200] via 10.0.12.2, enp0s9, weight 1, 00:00:14


router2# show ip route ospf 
Codes: K - kernel route, C - connected, S - static, R - RIP,
       O - OSPF, I - IS-IS, B - BGP, E - EIGRP, N - NHRP,
       T - Table, v - VNC, V - VNC-Direct, A - Babel, F - PBR,
       f - OpenFabric,
       > - selected route, * - FIB route, q - queued, r - rejected, b - backup
       t - trapped, o - offload failure

O   10.0.10.0/30 [110/100] is directly connected, enp0s8, weight 1, 00:03:12
O   10.0.11.0/30 [110/100] is directly connected, enp0s9, weight 1, 00:03:12
O>* 10.0.12.0/30 [110/200] via 10.0.10.1, enp0s8, weight 1, 00:02:32
  *                        via 10.0.11.1, enp0s9, weight 1, 00:02:32
O>* 192.168.10.0/24 [110/200] via 10.0.10.1, enp0s8, weight 1, 00:02:37
O   192.168.20.0/24 [110/100] is directly connected, enp0s10, weight 1, 00:03:12
O>* 192.168.30.0/24 [110/200] via 10.0.11.1, enp0s9, weight 1, 00:02:32
 ```
На router1 запускаем пинг от 192.168.10.1 до 192.168.20.1, а на router2 запускаем tcpdump, который будет смотреть трафик только на
порту enp0s9 и на enp0s8:
```bash
root@router1:~#  ping -I 192.168.10.1 192.168.20.1
PING 192.168.20.1 (192.168.20.1) from 192.168.10.1 : 56(84) bytes of data.
64 bytes from 192.168.20.1: icmp_seq=1 ttl=64 time=2.79 ms
64 bytes from 192.168.20.1: icmp_seq=2 ttl=64 time=0.858 ms
64 bytes from 192.168.20.1: icmp_seq=3 ttl=64 time=1.15 ms
64 bytes from 192.168.20.1: icmp_seq=4 ttl=64 time=1.21 ms
64 bytes from 192.168.20.1: icmp_seq=5 ttl=64 time=1.12 ms
64 bytes from 192.168.20.1: icmp_seq=6 ttl=64 time=1.25 ms
64 bytes from 192.168.20.1: icmp_seq=7 ttl=64 time=1.14 ms
...
root@router2:~# tcpdump -i enp0s9
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s9, link-type EN10MB (Ethernet), capture size 262144 bytes
08:52:20.678465 IP 192.168.10.1 > router2: ICMP echo request, id 6, seq 20, length 64
08:52:21.679859 IP 192.168.10.1 > router2: ICMP echo request, id 6, seq 21, length 64
08:52:22.681016 IP 192.168.10.1 > router2: ICMP echo request, id 6, seq 22, length 64
08:52:23.325568 IP 10.0.11.1 > ospf-all.mcast.net: OSPFv2, Hello, length 48
08:52:23.325876 IP router2 > ospf-all.mcast.net: OSPFv2, Hello, length 48
08:52:23.682680 IP 192.168.10.1 > router2: ICMP echo request, id 6, seq 23, length 64
08:52:24.684970 IP 192.168.10.1 > router2: ICMP echo request, id 6, seq 24, length 64
08:52:25.685705 IP 192.168.10.1 > router2: ICMP echo request, id 6, seq 25, length 64
...
root@router2:~# tcpdump -i enp0s8
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s8, link-type EN10MB (Ethernet), capture size 262144 bytes
08:54:15.545563 IP router2 > 192.168.10.1: ICMP echo reply, id 7, seq 9, length 64
08:54:16.584293 IP router2 > 192.168.10.1: ICMP echo reply, id 7, seq 10, length 64
08:54:17.614885 IP router2 > 192.168.10.1: ICMP echo reply, id 7, seq 11, length 64
08:54:18.640211 IP router2 > 192.168.10.1: ICMP echo reply, id 7, seq 12, length 64
08:54:19.664690 IP router2 > 192.168.10.1: ICMP echo reply, id 7, seq 13, length 64
...
```
Видим, что enp0s9 только получает ICMP-трафик с адреса 192.168.10.1, а отправляет enp0s8, т.е. настроен ассиметричный роутинг.

## сделать один из линков "дорогим", но что бы при этом роутинг был симметричным

Чтобы сделать роутинг симметричным, поменяем стоимость интерфейса enp0s8 на router2, запустим ping c router1 и смотрим трафик на router2:
```bash
root@router2:~# tcpdump -i enp0s9
tcpdump: verbose output suppressed, use -v or -vv for full protocol decode
listening on enp0s9, link-type EN10MB (Ethernet), capture size 262144 bytes
09:15:09.832162 IP 10.0.11.1 > ospf-all.mcast.net: OSPFv2, Hello, length 48
09:15:09.878468 IP router2 > ospf-all.mcast.net: OSPFv2, Hello, length 48
09:15:10.032357 IP 192.168.10.1 > router2: ICMP echo request, id 9, seq 41, length 64
09:15:10.032422 IP router2 > 192.168.10.1: ICMP echo reply, id 9, seq 41, length 64
09:15:11.033824 IP 192.168.10.1 > router2: ICMP echo request, id 9, seq 42, length 64
09:15:11.033875 IP router2 > 192.168.10.1: ICMP echo reply, id 9, seq 42, length 64
09:15:12.035715 IP 192.168.10.1 > router2: ICMP echo request, id 9, seq 43, length 64
09:15:12.035743 IP router2 > 192.168.10.1: ICMP echo reply, id 9, seq 43, length 64
...
```
Видим, что теперь пакеты приходят и уходят с enp0s9, т.е. наситроен симметричный роутинг.
