# Farklı bir DNS çözümü (NSD, Unbound)

* NSD ve Unbound kar amacı güdmeyen, kamu yararına çalışan kuruluş olan NLnet Labs tarafından geliştirilmekte ve halen sürdürülmektedir.
Her iki yazılım, OpenBSD temel sisteminin bir parçasıdır, herhangi bir paket yüklemeniz gerekmez.
#### NSD
* NLnet Labs, RIPE NCC ile işbirliği içinde, yetkili bir isim sunucusu olarak sıfırdan geliştirilmiştir. 
Tasarım gereği özyineleme ve önbellekleme işlevi yoktur. 
Sadece kendi etki alanlarına hizmet eder
#### Unbound
* Unbound, doğrulayıcı, özyinelemeli ve önbellekleme sunucusu olarak en az gereksinimleri karşılayacak bir çözümdür. 
Dinamik DNS güncellemelerini veya bölge aktarımlarını içeren kod barındırmaz.

| DNS    | Yayınlanma Tarihi           | Güvenlik Açığı(cvedetails.com) |
| ------------- |:-------------:| -----:|
| Powerdns      | 2002 | 26 |
| BIND      | 1988      |   102 |
| NSD, Unbound | 2002, 2007     |    3,3 -> 6 |


ROOT DNS sunucularının 3’ ü NSD

1. k.root-servers.net
2. h.root-servers.net
3. l.root-servers.net

### Örnek bir kampüs çözümü

- 2 adet master ve slave olarak farklı yapılandırmaya sahip NSD, sadece yerel ağda yer alan unbound sunucuları(iç DNS bölgesi) ve Harici DNS ler için yetkili isim sunucusu olacaktır.
- Ayarları tamamen aynı 2 adet Unbound, dahili ağdaki tüm istemciler için isim çözme hizmetleri sağlayacak, mümkünse harici isim sunucularından gelen cevaplar - - DNSSEC kullanılarak doğrulanacak, başarılı isim çözümlerinin önbellek kaydını yapacaktır.
Yazılımlar minimal kurulum yapılmış OpenBSD üzerinde yapılandırılacaktır.

![kampüs](https://github.com/smcn/dns/blob/master/nsd.JPG)

Zone transfer için anahtar oluşturuluyor

```
# echo "secret" | sha256 –b
s35QztzT4/H/ZPSvwEIghK5pQlPPOZMmho4Ho19KRfs=
```

```
nano /var/nsd/etc/nsd.conf
server:
    hide-version: yes
    verbosity: 1
    database: "" 
    logfile: "/var/log/nsd.log"
    ip-address: 10.10.1.51

remote-control:
    control-enable: yes
	
key:
    name: "sec_key"
    algorithm: hmac-sha256
    secret: "s35QztzT4/H/ZPSvwEIghK5pQlPPOZMmho4Ho19KRfs="

pattern:
    name: "master"
    zonefile: "master/%s"
    notify: 10.10.1.52 sec_key
    provide-xfr: 10.10.1.52 sec_key

zone:
    name: "omu.edu.tr"
    include-pattern: "master"
	
zone:
    name: 28.140.193.in-addr.arpa
    include-pattern: "master"
	
zone:
    name: 29.140.193.in-addr.arpa
    include-pattern: "master"

zone:
    name: "uzem.omu.edu.tr"
    include-pattern: "master"	
```

```
# nano /var/nsd/zones/master/omu.edu.tr	
$TTL    3600
omu.edu.tr.     IN      SOA     ns1.omu.edu.tr. can.omu.edu.tr. (
                        2018032601
                        10800
                        3600
                        1209600
                        86400 )
omu.edu.tr.     IN      NS      ns1.omu.edu.tr.
omu.edu.tr.     IN      NS      ns2.omu.edu.tr.
omu.edu.tr.     IN      MX      5 mx.omu.edu.tr.
ns1.omu.edu.tr. IN      A       193.140.28.6
ns2.omu.edu.tr. IN      A       193.140.28.7
webservers.omu.edu.tr.     IN      A       193.140.28.30
bidb.omu.edu.tr.           IN      CNAME   webservers.omu.edu.tr.
www.omu.edu.tr.            IN      A       193.140.28.8
```

```
# nano /var/nsd/zones/master/28.140.193.in-addr.arpa
$TTL    3600
28.140.193.in-addr.arpa.     IN    SOA    ns1.omu.edu.tr. can.omu.edu.tr. (
                        2017121301
                        10800
                        3600
                        1209600
                        86400 )
28.140.193.in-addr.arpa.     IN     NS    ns1.omu.edu.tr.
28.140.193.in-addr.arpa.     IN     NS    ns2.omu.edu.tr.

204.28.140.193.in-addr.arpa.  IN    PTR   ayna.omu.edu.tr.
```

NSD başlatma

```
# rcctl enable nsd
# rcctl -d start nsd
```

Ayrıntılı log için nsd.conf, verbose değeri artırılmalı, 1-5

```
# tail –f /var/log/nsd.log

```

```
# nano /var/nsd/etc/nsd.conf
server:
     hide-version: yes
     verbosity: 1
     database: "" # disable database
     logfile: "/var/log/nsd.log"
     ip-address: 10.10.1.52
	 
remote-control:
     control-enable: yes
	 
key:
     name: "sec_key"
     algorithm: hmac-sha256
     secret: "s35QztzT4/H/ZPSvwEIghK5pQlPPOZMmho4Ho19KRfs="
pattern:
    name: "slave"
    zonefile: "slave/%s"
    allow-notify: 10.10.1.51 sec_key
    request-xfr: AXFR 10.10.1.51@53 sec_key
zone:
    name: "omu.edu.tr"
    include-pattern: "slave"
zone:
    name: 28.140.193.in-addr.arpa
    include-pattern: "slave"
zone:
    name: "uzem.omu.edu.tr"
    include-pattern: "slave"

```

```
# rcctl -d restart
```

```
# tail –f /var/log/nsd.log
```

master

```
[2018-03-26 12:02:35.314] nsd[13817]: notice: nsd starting (NSD 4.1.17)
[2018-03-26 12:02:35.362] nsd[16267]: info: zone omu.edu.tr read with success
[2018-03-26 12:02:35.363] nsd[16267]: notice: nsd started (NSD 4.1.17), pid 88488
[2018-03-26 12:02:35.365] nsd[42800]: info: axfr for omu.edu.tr. from 10.10.1.52
```

slave

```
[2018-03-26 12:02:35.363] nsd[3633]: info: notify for omu.edu.tr. from 10.10.1.51 serial 2018032601
[2018-03-26 12:02:35.366] nsd[8239]: info: xfrd: zone omu.edu.tr committed "received update to serial 2018032601 at 2018-03-26T12:02:35 from 10.10.1.51@53 TSIG verified with key sec_key"
[2018-03-26 12:02:35.367] nsd[44562]: info: zone omu.edu.tr. received update to serial 2018032601 at 2018-03-26T12:02:35 from 10.10.1.51@53 TSIG verified with key sec_key of 257 bytes in 0.000104 seconds
[2018-03-26 12:02:35.371] nsd[8239]: info: zone omu.edu.tr serial 2018032201 is updated to 2018032601.
```

Slave DNS ZONE kayıtlarını hafızasında veya binary olarak tutmakta, Master DNS te bir sorun olduğunda text zone kayıtlarının kaybedilmemesi için:

```
# nsd-control write
```

### Unbound
- Unbound, dahili ağdaki tüm istemciler için isim çözme hizmetleri sağlayacaktır. Mümkünse harici isim sunucularından gelen cevaplar DNSSEC kullanılarak doğrulanacaktır.
- Unbound‘ un hafif kod yapısı, basit ve modüler tasarımı, Unbound‘ u son derece yüksek performanslı bir özyinelemeli isim sunucusu haline getirmeye katkıda bulunur. 
- Kıyaslama testi, Unix'in diğer ad sunucuları üzerinden (DNSSEC Doğrulaması etkin veya etkin değilken) 2 kata kadar performans sunmasını sağladı. 

```
# nano /var/unbound/etc/unbound.conf
server:
     interface: 127.0.0.1
     interface: 10.10.10.10
     do-ip6: no
     use-syslog: yes
     verbosity: 1
     access-control: 0.0.0.0/0 refuse
     access-control: 10.0.0.0/8 allow
     #root-hints: "/var/unbound/etc/root.hints"
     hide-identity: yes
     hide-version: yes
     #auto-trust-anchor-file: "/var/unbound/db/root.key"
	 
remote-control:
     control-enable: yes
     control-use-cert: no
     control-interface: /var/run/unbound.sock
	 
stub-zone:
     name: "omu.edu.tr"
     stub-addr: 10.10.1.51
     stub-addr: 10.10.1.52
	 
stub-zone:
     name: 28.140.193.in-addr.arpa
     stub-addr: 10.10.1.51
     stub-addr: 10.10.1.52
	 
stub-zone:
     name: "uzem.omu.edu.tr"
     stub-addr: 10.10.1.51
     stub-addr: 10.10.1.52
```

```
forward-zone:
        name: "."                    # use for ALL queries
        forward-addr: 8.8.8.8        # google.com
        forward-addr: 8.8.4.4        # google.com
        forward-addr: 37.235.1.174   # FreeDNS
        forward-addr: 37.235.1.177   # FreeDNS
        forward-addr: 50.116.23.211  # OpenNIC
        forward-addr: 64.6.64.6      # Verisign
        forward-addr: 64.6.65.6      # Verisign
        forward-addr: 74.82.42.42    # Hurricane Electric
        forward-addr: 84.200.69.80   # DNS Watch
        forward-addr: 84.200.70.40   # DNS Watch
        forward-addr: 91.239.100.100 # censurfridns.dk
        forward-addr: 109.69.8.51    # puntCAT
        forward-addr: 208.67.222.220 # OpenDNS
        forward-addr: 208.67.222.222 # OpenDNS
        forward-addr: 216.146.35.35  # Dyn Public
        forward-addr: 216.146.36.36  # Dyn Public
        forward-addr: 4.2.2.1        # Level 3 DNS
        forward-addr: 4.2.2.2        # Level 3 DNS
```

Unbound başlatma

```
# rcctl enable unbound
# rcctl -d start unbound
# tail –f /var/log/daemon
```

```
# ftp -o /var/unbound/db/root.hints ftp://ftp.internic.net/domain/named.cache
# root-hints: "/var/unbound/db/root.hints"
# unbound-anchor -a /var/unbound/db/root.key
# auto-trust-anchor-file: "/var/unbound/db/root.key"
```

```
unbound-control status
unbound-control reload
unbound-control dump_cache
unbound-checkconf       
unbound-control         
unbound-control-setup   
unbound-host
```

Unbound test

```
nslookup - 127.0.0.1
> set q=any
```

```
> bidb.omu.edu.tr
Server:         127.0.0.1
Address:        127.0.0.1#53

Non-authoritative answer:
bidb.omu.edu.tr canonical name = webservers.omu.edu.tr.

Authoritative answers can be found from:
omu.edu.tr      nameserver = ns1.omu.edu.tr.
omu.edu.tr      nameserver = ns2.omu.edu.tr.
ns1.omu.edu.tr  internet address = 193.140.28.6
ns2.omu.edu.tr  internet address = 193.140.28.7
```

```
> omu.edu.tr
Server:         127.0.0.1
Address:        127.0.0.1#53

Non-authoritative answer:
omu.edu.tr
        origin = ns1.omu.edu.tr
        mail addr = can.omu.edu.tr
        serial = 2018032601
        refresh = 10800
        retry = 3600
        expire = 1209600
        minimum = 86400
omu.edu.tr      nameserver = ns1.omu.edu.tr.
omu.edu.tr      nameserver = ns2.omu.edu.tr.
omu.edu.tr      mail exchanger = 5 mx.omu.edu.tr.

Authoritative answers can be found from:
ns1.omu.edu.tr  internet address = 193.140.28.6
ns2.omu.edu.tr  internet address = 193.140.28.7
```

```
nano /etc/pf.conf
set skip on lo

tcp_services = "{ ssh, domain }"
udp_services = "{ domain, ntp }"

block log all

pass in log proto tcp to port $tcp_services
pass out log proto tcp to port domain modulate state
pass log proto udp to port $udp_services
```

OpenBSD ile PF hazır ve çalışır durumda
PF log için

```
tcpdump -n -e -ttt -i pflog0
```

enable, disable, pf.conf reload

```
# pfctl -e
# pfctl -d
# pfctl -f  /etc/pf.conf
```
