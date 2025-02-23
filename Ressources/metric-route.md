Ma passerelle par défaut, qui me permet d'accéder à Internet, a une métrique de 100. J'ai ajouté manuellement une autre passerelle par défaut sans spécifier de métrique. Cette nouvelle route a pris la métrique 0 : je ne pouvais plus accéder à Internet.

```
root@vm2:~# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         172.16.6.1      0.0.0.0         UG    0      0        0 enp0s9
0.0.0.0         10.0.2.2        0.0.0.0         UG    100    0        0 enp0s3
0.0.0.0         172.16.6.1      0.0.0.0         UG    102    0        0 enp0s9
```

Lorsque j'essayais de pinguer une IP publique, cela ne fonctionnait pas :

```
root@vm2:~# ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
^C
--- 8.8.8.8 ping statistics ---
4 packets transmitted, 0 received, 100% packet loss, time 3049ms
```

Au début, je ne comprenais pas pourquoi je n'avais pas accès à Internet alors que le NAT sur VirtualBox était bien configuré. Grace à un ping qui spécifie l'interface à utiliser j'ai compris :

```
root@vm2:~# ping -I enp0s3 8.8.8.8
PING 8.8.8.8 (8.8.8.8) from 10.0.2.15 enp0s3: 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=63 time=53.4 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=63 time=51.8 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=63 time=55.2 ms
```

Pour valider mon hypothèse, j'ai supprimé la route avec la métrique 0 :

```
root@vm2:~# route del default gw 172.16.6.1
root@vm2:~# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.0.2.2        0.0.0.0         UG    100    0        0 enp0s3
0.0.0.0         172.16.6.1      0.0.0.0         UG    102    0        0 enp0s9
10.0.2.0        0.0.0.0         255.255.255.0   U     100    0        0 enp0s3
10.0.2.2        0.0.0.0         255.255.255.255 UH    100    0        0 enp0s3
10.0.2.3        0.0.0.0         255.255.255.255 UH    100    0        0 enp0s3
172.16.6.0      0.0.0.0         255.255.255.192 U     102    0        0 enp0s9
192.168.56.0    0.0.0.0         255.255.255.0   U     101    0        0 enp0s8

root@vm2:~# ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=63 time=72.2 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=63 time=46.0 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=63 time=23.4 ms
```

Maintenant lorsque j'ajouterai une passerelle par défaut, je ferai attention à la métrique !

```
root@vm2:~# route add default gw 172.16.6.1 metric 101 dev enp0s9
root@vm2:~# route -n
Kernel IP routing table
Destination     Gateway         Genmask         Flags Metric Ref    Use Iface
0.0.0.0         10.0.2.2        0.0.0.0         UG    100    0        0 enp0s3
0.0.0.0         172.16.6.1      0.0.0.0         UG    101    0        0 enp0s9
0.0.0.0         172.16.6.1      0.0.0.0         UG    102    0        0 enp0s9
10.0.2.0        0.0.0.0         255.255.255.0   U     100    0        0 enp0s3
10.0.2.2        0.0.0.0         255.255.255.255 UH    100    0        0 enp0s3
10.0.2.3        0.0.0.0         255.255.255.255 UH    100    0        0 enp0s3
172.16.6.0      0.0.0.0         255.255.255.192 U     102    0        0 enp0s9
192.168.56.0    0.0.0.0         255.255.255.0   U     101    0        0 enp0s8

root@vm2:~# ping 8.8.8.8 
PING 8.8.8.8 (8.8.8.8) 56(84) bytes of data.
64 bytes from 8.8.8.8: icmp_seq=1 ttl=63 time=40.7 ms
64 bytes from 8.8.8.8: icmp_seq=2 ttl=63 time=39.1 ms
64 bytes from 8.8.8.8: icmp_seq=3 ttl=63 time=41.2 ms
```