# BGP. Управление анонсами

## Цель

Настроить фильтрацию для офисе Москва.
Настроить фильтрацию для офисе Санкт-Петербург.

1. Настроить фильтрацию в офисе Москва так, чтобы не появилось транзитного трафика(As-path).
2. Настроить фильтрацию в офисе С.-Петербург так, чтобы не появилось транзитного трафика(Prefix-list).
3. Настроить провайдера Киторн так, чтобы в офис Москва отдавался только маршрут по умолчанию.
4. Настроить провайдера Ламас так, чтобы в офис Москва отдавался только маршрут по умолчанию и префикс офиса С.-Петербург.
5. Все сети в лабораторной работе должны иметь IP связность.

![hw11-1](../hw09-bgp/images/hw09-1.png)

## Описание

1. В Москве на маршрутизаторах R14 и R15 настроен filter-list на блокирование исходящих маршрутов с любой AS, кроме 1001 .
2. В Санкт-Петербурге на маршрутизаторе R18 настроен prefix-list на блокирование любых исходящих маршрутов, кроме локальных сетей.
3. В Киторн на машрутизаторе R22 настроен route-map SEND-DEFAULT out, который отправляет только маршрут по-умолчанию на R14.
4. В Ламас на машрутизаторе R21 также настроен route-map SEND-BGP-ANNOUNCE out, который отправляет только маршрут по-умолчанию и сети Санкт-Петербурга 10.2.0.0/16 и FC00:ABCD:2::/48 на R15.

## Настройка

### Москва

Настроим фильтр "нулевого" анонса других маршрутов, кроме своего с помощью ```ip as-path```.

#### R14

```
ip as-path access-list 10 permit ^$
!
router bgp 1001
 bgp router-id 10.1.254.14
 bgp log-neighbor-changes
 neighbor 10.1.254.15 remote-as 1001
 neighbor 10.1.254.15 update-source Loopback0
 neighbor 3000:1234:1000:1::1 remote-as 101
 neighbor 78.10.11.1 remote-as 101
 neighbor FC00:ABCD:1:FFFE::15 remote-as 1001
 neighbor FC00:ABCD:1:FFFE::15 update-source Loopback0
 !
 address-family ipv4
  network 10.1.0.0 mask 255.255.0.0
  neighbor 10.1.254.15 activate
  no neighbor 3000:1234:1000:1::1 activate
  neighbor 78.10.11.1 activate
  neighbor 78.10.11.1 filter-list 10 out
  no neighbor FC00:ABCD:1:FFFE::15 activate
 exit-address-family
 !
 address-family ipv6
  network FC00:ABCD:1::/48
  neighbor 3000:1234:1000:1::1 activate
  neighbor 3000:1234:1000:1::1 filter-list 10 out
  neighbor FC00:ABCD:1:FFFE::15 activate
 exit-address-family
```

#### R15

```
ip as-path access-list 10 permit ^$
!
router bgp 1001
 bgp router-id 10.1.254.15
 bgp log-neighbor-changes
 neighbor 10.1.254.14 remote-as 1001
 neighbor 10.1.254.14 update-source Loopback0
 neighbor 2500:2345:2000:1::1 remote-as 301
 neighbor 158.43.5.1 remote-as 301
 neighbor FC00:ABCD:1:FFFE::14 remote-as 1001
 neighbor FC00:ABCD:1:FFFE::14 update-source Loopback0
 !
 address-family ipv4
  network 10.1.0.0 mask 255.255.0.0
  neighbor 10.1.254.14 activate
  no neighbor 2500:2345:2000:1::1 activate
  neighbor 158.43.5.1 activate
  neighbor 158.43.5.1 filter-list 10 out
  no neighbor FC00:ABCD:1:FFFE::14 activate
 exit-address-family
 !
 address-family ipv6
  network FC00:ABCD:1::/48
  neighbor 2500:2345:2000:1::1 activate
  neighbor 2500:2345:2000:1::1 filter-list 10 out
  neighbor FC00:ABCD:1:FFFE::14 activate
 exit-address-family
```

### Санкт-Петербург

Настроим фильтр с помощью ```prefix-list``` только своих марщрутов.

#### R18

```
ip prefix-list NO-TRANSIT seq 5 permit 10.2.0.0/16
ipv6 prefix-list NO-TRANSIT seq 5 permit FC00:ABCD:2::/48
!
router bgp 2042
 bgp router-id 10.2.254.18
 bgp log-neighbor-changes
 neighbor 23.7.9.1 remote-as 520
 neighbor 23.7.9.5 remote-as 520
 neighbor 2900:3456:3000:1::1 remote-as 520
 neighbor 2900:3456:3000:2::5 remote-as 520
 !
 address-family ipv4
  network 10.2.0.0 mask 255.255.0.0
  neighbor 23.7.9.1 activate
  neighbor 23.7.9.1 prefix-list NO-TRANSIT out
  neighbor 23.7.9.5 activate
  neighbor 23.7.9.5 prefix-list NO-TRANSIT out
  no neighbor 2900:3456:3000:1::1 activate
  no neighbor 2900:3456:3000:2::5 activate
  maximum-paths 2
 exit-address-family
 !
 address-family ipv6
  maximum-paths 2
  network FC00:ABCD:2::/48
  neighbor 2900:3456:3000:1::1 activate
  neighbor 2900:3456:3000:1::1 prefix-list NO-TRANSIT out
  neighbor 2900:3456:3000:2::5 activate
  neighbor 2900:3456:3000:2::5 prefix-list NO-TRANSIT out
 exit-address-family
```

### Киторн

#### R22

Для ограничения анонса маршрута по-умолчанию создан ```route-map SEND-DEFAULT```.

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
!
ip prefix-list DEFAULT-ONLY seq 5 permit 0.0.0.0/0
!
ipv6 prefix-list DEFAULT-ONLY seq 5 permit ::/0
route-map SEND-DEFAULT permit 10
 match ip address prefix-list DEFAULT-ONLY
 match ipv6 address prefix-list DEFAULT-ONLY
```

### Ламас

#### R21

Для ограничения анонса маршрута по-умолчанию и сетей Санкт-Петербурга 10.2.0.0/16 и FC00:ABCD:2::/48 создан ```route-map SEND-BGP-ANNOUNCE```.

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
  neighbor 158.43.5.2 route-map SEND-BGP-ANNOUNCE out
  neighbor 158.43.5.6 activate
 exit-address-family
 !
 address-family ipv6
  redistribute connected
  neighbor 2500:2345:2000:1::2 activate
  neighbor 2500:2345:2000:1::2 default-originate
  neighbor 2500:2345:2000:1::2 route-map SEND-BGP-ANNOUNCE out
  neighbor 2500:2345:2000:2::6 activate
  neighbor 3000:1234:1000:2::5 activate
 exit-address-family
!
ip prefix-list BGP-ANNOUNCE seq 10 permit 0.0.0.0/0
ip prefix-list BGP-ANNOUNCE seq 20 permit 10.2.0.0/16
!
ipv6 prefix-list BGP-ANNOUNCE seq 10 permit ::/0
ipv6 prefix-list BGP-ANNOUNCE seq 20 permit FC00:ABCD:2::/48
route-map SEND-BGP-ANNOUNCE permit 10
 match ip address prefix-list BGP-ANNOUNCE
 match ipv6 address prefix-list BGP-ANNOUNCE
```

Полные настройки устройств приведены в в конфигурационных [файлах](./conf).

## Проверка

### Проверка анонса маршрута по-умолчанию для Москвы с Киторн

На маршрутизаторе R14 проверим, что маршрут по-умолчанию приходит от R22:

```
R14#sh bgp ipv4 unicast neighbors 78.10.11.1 routes
BGP table version is 129, local router ID is 10.1.254.14
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>  0.0.0.0          78.10.11.1                             0 101 i

Total number of prefixes 1

R14#sh bgp ipv6 unicast neighbors 3000:1234:1000:1::1 routes
BGP table version is 106, local router ID is 10.1.254.14
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *   ::/0             3000:1234:1000:1::1
                                                              0 101 i

Total number of prefixes 1

R14#sh ip route bgp
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       a - application route
       + - replicated route, % - next hop override

Gateway of last resort is 78.10.11.1 to network 0.0.0.0

B*    0.0.0.0/0 [20/0] via 78.10.11.1, 00:13:59

R14#sh ipv6 route bgp
IPv6 Routing Table - default - 31 entries
Codes: C - Connected, L - Local, S - Static, U - Per-user Static route
       B - BGP, HA - Home Agent, MR - Mobile Router, R - RIP
       H - NHRP, I1 - ISIS L1, I2 - ISIS L2, IA - ISIS interarea
       IS - ISIS summary, D - EIGRP, EX - EIGRP external, NM - NEMO
       ND - ND Default, NDp - ND Prefix, DCE - Destination, NDr - Redirect
       O - OSPF Intra, OI - OSPF Inter, OE1 - OSPF ext 1, OE2 - OSPF ext 2
       ON1 - OSPF NSSA ext 1, ON2 - OSPF NSSA ext 2, la - LISP alt
       lr - LISP site-registrations, ld - LISP dyn-eid, a - Application
B   ::/0 [200/0]
     via 2500:2345:2000:1::1
```

### Проверка анонса маршрута по-умолчанию и сетей Санкт-Петербурга для Москвы с Ламас

На маршрутизаторе R15 проверим, что маршрут по-умолчанию и сети Санкт-Петербурга приходят от R21:

```
R15#sh ip bgp neighbors 158.43.5.1 routes
BGP table version is 110, local router ID is 10.1.254.15
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>  0.0.0.0          158.43.5.1                    150      0 301 i
 *>  10.2.0.0/16      158.43.5.1                    150      0 301 520 2042 i

Total number of prefixes 2

R15#sh bgp ipv6 unicast neighbors 2500:2345:2000:1::1 routes
BGP table version is 101, local router ID is 10.1.254.15
Status codes: s suppressed, d damped, h history, * valid, > best, i - internal,
              r RIB-failure, S Stale, m multipath, b backup-path, f RT-Filter,
              x best-external, a additional-path, c RIB-compressed,
Origin codes: i - IGP, e - EGP, ? - incomplete
RPKI validation codes: V valid, I invalid, N Not found

     Network          Next Hop            Metric LocPrf Weight Path
 *>  ::/0             2500:2345:2000:1::1
                                                     150      0 301 i
 *>  FC00:ABCD:2::/48 2500:2345:2000:1::1
                                                     150      0 301 520 2042 i

Total number of prefixes 2

R15#sh ip route bgp
Codes: L - local, C - connected, S - static, R - RIP, M - mobile, B - BGP
       D - EIGRP, EX - EIGRP external, O - OSPF, IA - OSPF inter area
       N1 - OSPF NSSA external type 1, N2 - OSPF NSSA external type 2
       E1 - OSPF external type 1, E2 - OSPF external type 2
       i - IS-IS, su - IS-IS summary, L1 - IS-IS level-1, L2 - IS-IS level-2
       ia - IS-IS inter area, * - candidate default, U - per-user static route
       o - ODR, P - periodic downloaded static route, H - NHRP, l - LISP
       a - application route
       + - replicated route, % - next hop override

Gateway of last resort is 158.43.5.1 to network 0.0.0.0

B*    0.0.0.0/0 [20/0] via 158.43.5.1, 00:05:09
      10.0.0.0/8 is variably subnetted, 26 subnets, 4 masks
B        10.2.0.0/16 [20/0] via 158.43.5.1, 00:05:09

R15#sh ipv route bgp
IPv6 Routing Table - default - 31 entries
Codes: C - Connected, L - Local, S - Static, U - Per-user Static route
       B - BGP, HA - Home Agent, MR - Mobile Router, R - RIP
       H - NHRP, I1 - ISIS L1, I2 - ISIS L2, IA - ISIS interarea
       IS - ISIS summary, D - EIGRP, EX - EIGRP external, NM - NEMO
       ND - ND Default, NDp - ND Prefix, DCE - Destination, NDr - Redirect
       O - OSPF Intra, OI - OSPF Inter, OE1 - OSPF ext 1, OE2 - OSPF ext 2
       ON1 - OSPF NSSA ext 1, ON2 - OSPF NSSA ext 2, la - LISP alt
       lr - LISP site-registrations, ld - LISP dyn-eid, a - Application
B   ::/0 [20/0]
     via FE80::A8BB:CCFF:FE01:5000, Ethernet0/2
B   FC00:ABCD:2::/48 [20/0]
     via FE80::A8BB:CCFF:FE01:5000, Ethernet0/2
```

### Проверка связности сетей (R12 -> R17)

```
R12>ping 10.2.254.17
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 10.2.254.17, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms

R12>ping FC00:ABCD:2:FFFE::17
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to FC00:ABCD:2:FFFE::17, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5), round-trip min/avg/max = 1/1/2 ms
```
