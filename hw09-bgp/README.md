# Основы eBPG

## Цель

Настроить BGP между автономными системами. Организовать доступность между офисами Москва и С.-Петербург

1. Настроите eBGP между офисом Москва и двумя провайдерами - Киторн и Ламас.
2. Настроите eBGP между провайдерами Киторн и Ламас.
3. Настроите eBGP между Ламас и Триада.
4. Настроите eBGP между офисом С.-Петербург и провайдером Триада.
5. Организуете IP доступность между пограничным роутерами офисами Москва и С.-Петербург.

![hw09-1](./images/hw09-1.png)

## Настройка BGP

### Описание

1. Временные статические маршруты на R22, R21, R24, R26 были удалены
2. Настроен eBGP согласно заданию
2. Для офисов Москвы и Санкт-Петербурга вышестоящими маршрутизаторами анонсируются маршруты по-умолчанию, ограниченные route-map
3. Во всех офисах и провайдерах настроена перераспределние маршрутов EGP в IGP (redistribute), и наоборот
4. В итоге любое устройство доступно с любого другого устройства

## Настройка

### Киторн

#### R22

Настроены маршруты по-умолчанию через Null0 и route-map для ограничения анонса маршрута по-умолчанию:

```
ip route 0.0.0.0 0.0.0.0 Null0
ipv6 route ::/0 Null0
!
ip prefix-list DEFAULT-ONLY seq 5 permit 0.0.0.0/0
ipv6 prefix-list DEFAULT-ONLY seq 5 permit ::/0
!
route-map SEND-DEFAULT permit 10
 match ip address prefix-list DEFAULT-ONLY
 match ipv6 address prefix-list DEFAULT-ONLY
```

Настроен BGP:

```
router bgp 101
 bgp router-id 172.16.254.22
 bgp log-neighbor-changes
 neighbor 3000:1234:1000:1::2 remote-as 1001
 neighbor 3000:1234:1000:2::6 remote-as 301
 neighbor 3000:1234:1000:3::10 remote-as 520
 neighbor 78.10.11.2 remote-as 1001
 neighbor 78.10.11.6 remote-as 301
 neighbor 78.10.11.10 remote-as 520
 !
 address-family ipv4
  redistribute connected
  no neighbor 3000:1234:1000:1::2 activate
  no neighbor 3000:1234:1000:2::6 activate
  no neighbor 3000:1234:1000:3::10 activate
  neighbor 78.10.11.2 activate
  neighbor 78.10.11.2 default-originate
  neighbor 78.10.11.2 route-map SEND-DEFAULT out
  neighbor 78.10.11.6 activate
  neighbor 78.10.11.10 activate
 exit-address-family
 !
 address-family ipv6
  redistribute connected
  neighbor 3000:1234:1000:1::2 activate
  neighbor 3000:1234:1000:1::2 default-originate
  neighbor 3000:1234:1000:1::2 route-map SEND-DEFAULT out
  neighbor 3000:1234:1000:2::6 activate
  neighbor 3000:1234:1000:3::10 activate
 exit-address-family
```

### Ламас

#### R21

Аналогично R21 настроены маршруты по-умолчанию через Null0 и route-map для ограничения анонса маршрута по-умолчанию.

Настроен BGP:

```
router bgp 301
 bgp router-id 172.17.254.21
 bgp log-neighbor-changes
 neighbor 2500:2345:2000:1::2 remote-as 1001
 neighbor 2500:2345:2000:2::6 remote-as 520
 neighbor 3000:1234:1000:2::5 remote-as 101
 neighbor 78.10.11.5 remote-as 101
 neighbor 158.43.5.2 remote-as 1001
 neighbor 158.43.5.6 remote-as 520
 !
 address-family ipv4
  redistribute connected
  no neighbor 2500:2345:2000:1::2 activate
  no neighbor 2500:2345:2000:2::6 activate
  no neighbor 3000:1234:1000:2::5 activate
  neighbor 78.10.11.5 activate
  neighbor 158.43.5.2 activate
  neighbor 158.43.5.2 default-originate
  neighbor 158.43.5.2 route-map SEND-DEFAULT out
  neighbor 158.43.5.6 activate
 exit-address-family
 !
 address-family ipv6
  redistribute connected
  neighbor 2500:2345:2000:1::2 activate
  neighbor 2500:2345:2000:1::2 default-originate
  neighbor 2500:2345:2000:1::2 route-map SEND-DEFAULT out
  neighbor 2500:2345:2000:2::6 activate
  neighbor 3000:1234:1000:2::5 activate
 exit-address-family
```

### Москва

#### R14

В OSPF настроен анонс маршрута по умолчанию для других IGP маршрутизаторов:

```
router ospfv3 1
 area 101 stub no-summary
 !
 address-family ipv4 unicast
  default-information originate
 exit-address-family
 !
 address-family ipv6 unicast
  default-information originate
 exit-address-family
```

Настроен BGP с перераспределнием маршрутов connected и ospfv3:

```
router bgp 1001
 bgp router-id 10.1.254.14
 bgp log-neighbor-changes
 neighbor 3000:1234:1000:1::1 remote-as 101
 neighbor 78.10.11.1 remote-as 101
 !
 address-family ipv4
  redistribute connected
  redistribute ospfv3 1
  no neighbor 3000:1234:1000:1::1 activate
  neighbor 78.10.11.1 activate
 exit-address-family
 !
 address-family ipv6
  redistribute connected
  redistribute ospf 1
  neighbor 3000:1234:1000:1::1 activate
 exit-address-family
 ```

#### R15

Аналогично R14 в OSPF настроен анонс маршрута по умолчанию для других IGP маршрутизаторов.

Настроен BGP с перераспределнием маршрутов connected и ospfv3:

```
router bgp 1001
 bgp router-id 10.1.254.15
 bgp log-neighbor-changes
 neighbor 2500:2345:2000:1::1 remote-as 301
 neighbor 158.43.5.1 remote-as 301
 !
 address-family ipv4
  redistribute connected
  redistribute ospfv3 1
  no neighbor 2500:2345:2000:1::1 activate
  neighbor 158.43.5.1 activate
 exit-address-family
 !
 address-family ipv6
  redistribute connected
  redistribute ospf 1
  neighbor 2500:2345:2000:1::1 activate
 exit-address-family
 ```

### Триада

#### R23

В ISIS настроено перераспределение BGP маршрутов:

```
router isis
 net 49.2222.1720.1825.4023.00
 metric-style wide
 redistribute bgp 520 metric 10
 !
 address-family ipv6
  redistribute bgp 520
 exit-address-family
```

Настроен BGP с перераспределнием маршрутов connected и ISIS Level-2:

```
router bgp 520
 bgp router-id 172.18.254.23
 bgp log-neighbor-changes
 neighbor 3000:1234:1000:3::9 remote-as 101
 neighbor 78.10.11.9 remote-as 101
 !
 address-family ipv4
  redistribute connected
  redistribute isis level-2
  no neighbor 3000:1234:1000:3::9 activate
  neighbor 78.10.11.9 activate
 exit-address-family
 !
 address-family ipv6
  redistribute connected
  redistribute isis level-2
  neighbor 3000:1234:1000:3::9 activate
 exit-address-family
```


#### R24

Аналогично R21 настроены маршруты по-умолчанию через Null0 и route-map для ограничения анонса маршрута по-умолчанию.

В ISIS настроено перераспределение BGP маршрутов:

```
router isis
 net 49.0024.1720.1825.4024.00
 is-type level-2-only
 metric-style wide
 redistribute bgp 520 metric 10
 !
 address-family ipv6
  redistribute bgp 520
 exit-address-family
```

Настроен BGP с перераспределнием маршрутов connected и ISIS Level-2:

```
router bgp 520
 bgp router-id 172.18.254.24
 bgp log-neighbor-changes
 neighbor 23.7.9.2 remote-as 2042
 neighbor 2500:2345:2000:2::5 remote-as 301
 neighbor 2900:3456:3000:1::2 remote-as 2042
 neighbor 158.43.5.5 remote-as 301
 !
 address-family ipv4
  redistribute connected
  redistribute isis level-2
  neighbor 23.7.9.2 activate
  neighbor 23.7.9.2 default-originate
  neighbor 23.7.9.2 route-map SEND-DEFAULT out
  no neighbor 2500:2345:2000:2::5 activate
  no neighbor 2900:3456:3000:1::2 activate
  neighbor 158.43.5.5 activate
 exit-address-family
 !
 address-family ipv6
  redistribute connected
  redistribute isis level-2
  neighbor 2500:2345:2000:2::5 activate
  neighbor 2900:3456:3000:1::2 activate
  neighbor 2900:3456:3000:1::2 default-originate
  neighbor 2900:3456:3000:1::2 route-map SEND-DEFAULT out
 exit-address-family
```

#### R26

Аналогично R21 настроены маршруты по-умолчанию через Null0 и route-map для ограничения анонса маршрута по-умолчанию.

В ISIS настроено перераспределение BGP маршрутов:

```
router isis
 net 49.0026.1720.1825.4026.00
 is-type level-2-only
 metric-style wide
 redistribute bgp 520 metric 10
 !
 address-family ipv6
  redistribute bgp 520
 exit-address-family
```

Настроен BGP с перераспределнием маршрутов connected и ISIS Level-2:

```
router bgp 520
 bgp router-id 172.18.254.26
 bgp log-neighbor-changes
 neighbor 23.7.9.6 remote-as 2042
 neighbor 2900:3456:3000:2::6 remote-as 2042
 !
 address-family ipv4
  redistribute connected
  redistribute isis level-2
  neighbor 23.7.9.6 activate
  neighbor 23.7.9.6 default-originate
  neighbor 23.7.9.6 route-map SEND-DEFAULT out
  no neighbor 2900:3456:3000:2::6 activate
 exit-address-family
 !
 address-family ipv6
  redistribute connected
  redistribute isis level-2
  neighbor 2900:3456:3000:2::6 activate
  neighbor 2900:3456:3000:2::6 default-originate
  neighbor 2900:3456:3000:2::6 route-map SEND-DEFAULT out
 exit-address-family
```

> Нерешенная проблема: при выключении R26 маршруты из офиса Санкт-Петербурга не передаются в сторону Москвы и наоборот. 
> Соответственно пропадает связность между офисами Санкт-Петербурга и Москвы.
> 
> На R23 присутствуют маршруты Москвы.
>
> На R25 и R26 присутствуют маршруты Санкт-Петербурга.

### Санкт-Петербург

#### R18

В EIGRP настроен анонс маршрутов BGP (по-умолчанию) для других IGP маршрутизаторов:

```
router eigrp PITER
 !
 address-family ipv4 unicast autonomous-system 2042
  !
  topology base
   redistribute bgp 2042 metric 10000 1 255 1 1500
  exit-af-topology
  network 10.2.253.0 0.0.0.3
  network 10.2.253.4 0.0.0.3
  network 10.2.254.18 0.0.0.0
  eigrp router-id 10.2.254.18
 exit-address-family
 !
 address-family ipv6 unicast autonomous-system 2042
  !
  topology base
   redistribute bgp 2042 metric 10000 1 255 1 1500
  exit-af-topology
  eigrp router-id 10.2.254.18
 exit-address-family
```

Настроен BGP с перераспределнием маршрутов connected и EIGRP:

```
router bgp 2042
 bgp router-id 10.2.254.18
 bgp log-neighbor-changes
 neighbor 23.7.9.1 remote-as 520
 neighbor 23.7.9.5 remote-as 520
 neighbor 2900:3456:3000:1::1 remote-as 520
 neighbor 2900:3456:3000:2::5 remote-as 520
 !
 address-family ipv4
  redistribute connected
  redistribute eigrp 2042
  neighbor 23.7.9.1 activate
  neighbor 23.7.9.5 activate
  no neighbor 2900:3456:3000:1::1 activate
  no neighbor 2900:3456:3000:2::5 activate
 exit-address-family
 !
 address-family ipv6
  redistribute connected
  redistribute eigrp 2042
  neighbor 2900:3456:3000:1::1 activate
  neighbor 2900:3456:3000:2::5 activate
 exit-address-family
 ```

Полные настройки устройств приведены в в конфигурационных [файлах](./conf).

## Проверка

### Свзяность VPC1: VPC8 и VPC11

```
VPC1> sh ip all
NAME   IP/MASK              GATEWAY           MAC                DNS
VPC1   10.1.1.2/24          10.1.1.1          00:50:79:66:68:01  10.1.1.1

VPC8> sh ip all
NAME   IP/MASK              GATEWAY           MAC                DNS
VPC8   10.2.1.3/24          10.2.1.1          00:50:79:66:68:08  10.2.1.1

VPC11> sh ip all
NAME   IP/MASK              GATEWAY           MAC                DNS
VPC11  10.2.2.2/24          10.2.2.1          00:50:79:66:68:0b  10.2.2.1
```

```
VPC1> trace 10.2.1.3 -P 1
trace to 10.2.1.3, 8 hops max (ICMP), press Ctrl+C to stop
 1   10.1.1.4   0.513 ms  0.444 ms  0.546 ms
 2   10.1.253.33   0.852 ms  0.628 ms  0.547 ms
 3   10.1.253.17   1.176 ms  1.076 ms  0.940 ms
 4   158.43.5.1   0.975 ms  1.251 ms  1.100 ms
 5   158.43.5.6   1.379 ms  1.238 ms  1.370 ms
 6   23.7.9.2   1.831 ms  1.540 ms  1.543 ms
 7   10.2.253.2   1.848 ms  1.554 ms  1.668 ms
 8   10.2.1.3   2.894 ms  1.762 ms  1.647 ms

VPC1> trace 10.2.2.2 -P 1
trace to 10.2.2.2, 8 hops max (ICMP), press Ctrl+C to stop
 1   10.1.1.4   0.798 ms  0.468 ms  0.498 ms
 2   10.1.253.25   1.143 ms  0.825 ms  0.862 ms
 3   10.1.253.5   1.160 ms  1.135 ms  1.041 ms
 4   78.10.11.1   1.213 ms  1.131 ms  1.078 ms
 5   78.10.11.10   1.809 ms  1.246 ms  1.241 ms
 6   172.18.253.2   1.572 ms  1.510 ms  1.544 ms
 7   23.7.9.2   1.817 ms  1.609 ms  1.722 ms
 8   10.2.253.2   1.985 ms  1.741 ms  1.747 ms
```

### Свзяность между пограничными маршрутизаторами R14 и R15: R18

```
R14#sh ip int br
Interface                  IP-Address      OK? Method Status                Protocol
Ethernet0/0                10.1.253.5      YES NVRAM  up                    up
Ethernet0/1                10.1.253.9      YES NVRAM  up                    up
Ethernet0/2                78.10.11.2      YES NVRAM  up                    up
Ethernet0/3                10.1.253.1      YES NVRAM  up                    up
Loopback0                  10.1.254.14     YES NVRAM  up                    up

R15#sh ip int br
Interface                  IP-Address      OK? Method Status                Protocol
Ethernet0/0                10.1.253.17     YES NVRAM  up                    up
Ethernet0/1                10.1.253.13     YES NVRAM  up                    up
Ethernet0/2                158.43.5.2      YES NVRAM  up                    up
Ethernet0/3                10.1.253.21     YES NVRAM  up                    up
Loopback0                  10.1.254.15     YES NVRAM  up                    up

R18#sh ip int br
Interface                  IP-Address      OK? Method Status                Protocol
Ethernet0/0                10.2.253.5      YES manual up                    up
Ethernet0/1                10.2.253.1      YES manual up                    up
Ethernet0/2                23.7.9.2        YES manual up                    up
Ethernet0/3                23.7.9.6        YES manual up                    up
Loopback0                  10.2.254.18     YES manual up                    up
```

```
R14#traceroute 10.2.254.18
Type escape sequence to abort.
Tracing the route to 10.2.254.18
VRF info: (vrf in name/id, vrf out name/id)
  1 78.10.11.1 1 msec 0 msec 0 msec
  2 78.10.11.10 [AS 101] 1 msec 1 msec 1 msec
  3 172.18.253.2 [AS 101] 0 msec 0 msec 1 msec
  4 23.7.9.2 [AS 101] 1 msec *  1 msec

R15#traceroute 10.2.254.18
Type escape sequence to abort.
Tracing the route to 10.2.254.18
VRF info: (vrf in name/id, vrf out name/id)
  1 158.43.5.1 0 msec 1 msec 0 msec
  2 158.43.5.6 [AS 301] 0 msec 1 msec 0 msec
  3 23.7.9.2 [AS 301] 1 msec *  1 msec
```
