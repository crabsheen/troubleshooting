### how to find server's uplink switch?

Sometimes we need to know which switch that app server connected on because when there has network problem.

some problems for example

```
traffic bandwidth limitation stricted,
interface negotiation rate is changed;
packets loss a little much.
```

So we can obtain switch info from LLDP,Let's have a look.

```
$ sudo tcpdump -i bond0 ether proto 0x88cc -A -s0 -t -c 100 -vv 
tcpdump: listening on bond0, link-type EN10MB (Ethernet), capture size 65535 bytes
LLDP, length 347
        Chassis ID TLV (1), length 7
          Subtype MAC address (4): 50:1c:bf:71:12:00 (oui Unknown)
          0x0000:  0450 1cbf 7112 00
        Port ID TLV (2), length 9
          Subtype Interface Name (5): Gi1/0/25
          0x0000:  0547 6931 2f30 2f32 35
        Time to Live TLV (3), length 2: TTL 180s
          0x0000:  00b4
        System Name TLV (5), length 8: qihe1030
          0x0000:  7169 6865 3130 3330
        System Description TLV (6), length 247
          Cisco IOS Software, C2960S Software (C2960S-UNIVERSALK9-M), Version 15.0(2)SE8, RELEASE SOFTWARE (fc1)\0x0aTechnical Support: http://www.cisco.com/techsupport\0x0aCopyright (c) 1986-2015 by Cisco Systems, Inc.\0x0aCompiled Wed 13-May-15 23:10 by prod_rel_team
          0x0000:  4369 7363 6f20 494f 5320 536f 6674 7761
          0x0010:  7265 2c20 4332 3936 3053 2053 6f66 7477
          0x0020:  6172 6520 2843 3239 3630 532d 554e 4956
          0x0030:  4552 5341 4c4b 392d 4d29 2c20 5665 7273
          0x0040:  696f 6e20 3135 2e30 2832 2953 4538 2c20
          0x0050:  5245 4c45 4153 4520 534f 4654 5741 5245
          0x0060:  2028 6663 3129 0a54 6563 686e 6963 616c
          0x0070:  2053 7570 706f 7274 3a20 6874 7470 3a2f
          0x0080:  2f77 7777 2e63 6973 636f 2e63 6f6d 2f74
          0x0090:  6563 6873 7570 706f 7274 0a43 6f70 7972
          0x00a0:  6967 6874 2028 6329 2031 3938 362d 3230
          0x00b0:  3135 2062 7920 4369 7363 6f20 5379 7374
          0x00c0:  656d 732c 2049 6e63 2e0a 436f 6d70 696c
          0x00d0:  6564 2057 6564 2031 332d 4d61 792d 3135
          0x00e0:  2032 333a 3130 2062 7920 7072 6f64 5f72
          0x00f0:  656c 5f74 6561 6d
        Port Description TLV (4), length 21: GigabitEthernet1/0/25
```
last line show the info about uplink port.

```
Then we can use cacti traffic graph to see the traffic monitor
or cpu usage
or in/out errors counting 
or any running state about switch.


such as
$ snmpwalk -c community -v 2c 10.11.1.30 1.3.6.1.2.1.2.2|head
IF-MIB::ifIndex.1 = INTEGER: 1
IF-MIB::ifIndex.51 = INTEGER: 51
IF-MIB::ifIndex.5001 = INTEGER: 5001
IF-MIB::ifIndex.5137 = INTEGER: 5137
IF-MIB::ifIndex.5138 = INTEGER: 5138
IF-MIB::ifIndex.5139 = INTEGER: 5139
IF-MIB::ifIndex.5140 = INTEGER: 5140
IF-MIB::ifIndex.5141 = INTEGER: 5141
IF-MIB::ifIndex.5142 = INTEGER: 5142
IF-MIB::ifIndex.10101 = INTEGER: 10101

It's very useful below.
IF-MIB::ifIndex
IF-MIB::ifDescr
IF-MIB::ifType
IF-MIB::ifMtu
IF-MIB::ifSpeed
IF-MIB::ifPhysAddress
IF-MIB::ifAdminStatus
IF-MIB::ifOperStatus
IF-MIB::ifLastChange
IF-MIB::ifInOctets
IF-MIB::ifInUcastPkts
IF-MIB::ifInDiscards
IF-MIB::ifInErrors
IF-MIB::ifInUnknownProtos
IF-MIB::ifOutOctets
IF-MIB::ifOutUcastPkts
IF-MIB::ifOutDiscards
IF-MIB::ifOutErrors
```
