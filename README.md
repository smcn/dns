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


1								~      +---------------+                            ~
2								~ 	   |               |                            ~
3				   +-------------------+    unbound    +--------------+             ~
4				   |            ~  	   |               |              |             ~
5				   v            ~      +-------+-------+              v             ~    +----------+
6			 +-----------+      ~ 		       ^               +--------------+     ~    |          +----+
7		+----|           |		~			   |               |              |     ~    |          |    |
8		|    |           |		~	   +-------+-------+       |    nsd(m)    | <--------+          |    |
9		|    |   Harici  |		~	   |               |       |              |     ~    |  Harici  |    |
10		|    |    DNS    |		~	   |    Yerel      |       +--------------+     ~    |   DNS    |    |
11		|    |           |		~	   | Kullanıcılar  |              |             ~    |          |    |
12		|    |           |		~	   |               |              |             ~    |          |    |
13		|    |           |		~	   +---------------+              v             ~    |          |    |
14		|    |           |		~	  / ............. /        +--------------+     ~    |          |    |
15		|    |           |		~	 / ............. /         |              |     ~    |          |    |
16		|    |           |		~	+----------+----+	       |    nsd(s)    | <--------+          |    |
17		|    |           |      ~              |               |              |     ~    |          |    |
18		|    +-----+-----+		~			   v               +--------------+     ~    +----------+    |
19		+----+     ^            ~      +---------------+              ^             ~         +----------+
20				   |    		~	   |               |              |             ~
21				   +-------------------+    unbound    +--------------+             ~
22								~	   |               |                            ~
23								~      +---------------+                            ~
24 (m) = master
25 (s) = slave
26
