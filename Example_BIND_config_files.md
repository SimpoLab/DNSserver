>[!warning]
>Modify the files in order to apply to your case! These files **will not work out of the box**.
>See the [main guide](README.md) for explanation and (e.g. key generating) commands.

Our setup consists of a main zone example.org, a manually mantained subzone user1.example.org, and a dynamic DNS zone dyn.example.org.

Users user1 and user2 can dynamically modify records inside the dynamic zone, each under their subdomain user[1/2].dyn.example.com.

Useruser1 set up ddclient to update an A record `home.user1.dyn.example.com` to their home IP address, which is also CNAMEd to `home.user1.example.com`.

Everything is DNSSEC-enabled.

We disabled ipv6.



# BIND config


## BIND config file
Our main config file looks something like this

```named title="named.conf"
options {
	directory "/var/named";
	pid-file "/run/named/named.pid";

	listen-on	{ any; };
    // listen-on-v6 { any; };

	allow-recursion { none; };
	allow-transfer { none; };
	allow-update { none; };

	version none;
	hostname none;
	server-id none;
};

zone "localhost" IN {
	type master;
	file "localhost.zone";
};

zone "0.0.127.in-addr.arpa" IN {
	type master;
	file "127.0.0.zone";
};

//zone "1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.ip6.arpa" {
//	type master;
//	file "localhost.ip6.zone";
//};

// key for user1's dynamic updates
key "user1key" {
    algorithm hmac-sha256;
    secret "ppeue5dzUkw.........MulajQySI=";
};

// key for user2's dynamic updates
key "user2key" {
    algorithm hmac-sha256;
    secret "ccrhr5qmHxj.........ZhynwDlFV=";
};

// dynamic zone
zone "dyn.example.com" {
	type master;
	auto-dnssec maintain; // deprecated but simpler version
	inline-signing yes;
	key-directory "keys/";
	file "dyn.example.com.zone";
	update-policy {
		grant user1key wildcard *.user1.dyn.example.com ANY;
		grant user2key wildcard *.user2.dyn.example.com ANY;
	};
};

// user1's own zone
zone "user1.example.com" {
	type master;
	auto-dnssec maintain; // deprecated but simpler version
	inline-signing yes;
	key-directory "/srv/named/user1/keys/";
	file "/srv/named/user1/user1.example.com.zone";
	update-policy {
		grant user1key zonesub ANY;
	};
};

// user2's own zone
zone "user2.example.com" {
	type master;
	auto-dnssec maintain; // deprecated but simpler version
	inline-signing yes;
	key-directory "/srv/named/user2/keys/";
	file "/srv/named/user2/user2.example.com.zone";
	update-policy {
		grant user2key zonesub ANY;
	};
};

// main zone
zone "example.com" {
	type master;
	auto-dnssec maintain; // deprecated but simpler version
	inline-signing yes;
	key-directory "keys/";
	file "example.com.zone";
};

logging {
	channel xfer-log {
		file "/var/log/named/server.log";
		print-category yes;
		print-severity yes;
		severity info;
	};
	category xfer-in { xfer-log; };
	category xfer-out { xfer-log; };
	category notify { xfer-log; };

	channel query_log {
		file "/var/log/named/query.log" versions 3 size 10m;
		severity debug 3;
		print-time yes;
		print-severity yes;
		print-category yes;
	};
	category queries { query_log; };
};
```


## Zonefiles
The domain registrar has to be set up to point `ns1.example.com` end `ns2.example.com` to the server's IP address (NS+A records, see their docs), and it should contain a DS record containing the public key corresponding to the main zone KSK (use `dnssec-dsfromkey`).

Our local zonefiles will instead look something like the following.

Main zone:

```zonefile title="/var/named/example.com.zone"
$ORIGIN	example.com.
$TTL	2h

; DNS
@		SOA	ns1	hostmaster (
				2023151101	; Serial
				8h		    ; Refresh
				30m		    ; Retry
				1w		    ; Expire
				1h )		; Negative Cache TTL
@		NS	ns1
@		NS	ns2
ns1		A	42.42.42.42
ns2		A	42.42.42.42

; DNSSEC
@	DNSKEY 256 3 7	AwEAAcRfHHSvLz+NEUa.......7I/gDUr6zvF4s0yJMrv2YoXsNU= ; ZSK, ID: 01234
@	DNSKEY 257 3 7	AwEAAacg3BB/M6KbDXe.......aGuKocjR+n TSyGx6zAahwemuMr ; KSK, ID: 04321
; KSK DS record for reference:
; example.com. IN DS 54766 7 2 D501921A25AE0092AC.......A92A782D8C7029BA67B8A

; RECORDS
@		A	42.42.42.42

git			CNAME	@
info        TXT     "this is some example TXT record"


; OTHER ZONES

; user1
user1	    NS	ns1.user1
user1	    NS	ns2.user1
ns1.user1	A	42.42.42.42
ns2.user1	A	42.42.42.42
user1       DS 44926 7 2 186A498853ED0EE54E.......708481BA231D6CCC11B0D

; user2
user2	    NS	ns1.user2
user2	    NS	ns2.user2
ns1.user2	A	42.42.42.42
ns2.user2	A	42.42.42.42
user2       DS 32994 7 2 A5A0B06E98D776BBC3.......C4A98483357BB637A3768

; DYNAMIC
dyn         NS	ns1.dyn
dyn         NS	ns2.dyn
ns1.dyn	    A	42.42.42.42
ns2.dyn	    A	42.42.42.42
dyn         DS 22180 7 2 00D2BE8B7BA10B11CA.......26C7623A0B69B9FF55311
```

For user1:

```zonefile title="/srv/named/user1/user1.example.com.zone"
$ORIGIN	user1.example.com.
$TTL	2h

; DNS
@		SOA	ns1	hostmaster (
				2023151101	; Serial
				8h		    ; Refresh
				30m		    ; Retry
				1w		    ; Expire
				1h )		; Negative Cache TTL
@		NS	ns1
@		NS	ns2
ns1		A	42.42.42.42
ns2		A	42.42.42.42

; DNSSEC
@   DNSKEY 256 3 7  AwEAAcQUSthDBbzeV3F.......OlhWxgEFyUuxuI ApMuh8kZJyc= ; ZSK, ID: 11234
@   DNSKEY 257 3 7  AwEAAZVwTSB8pR8+n3n.......mGEDU24aHB zkvj84fFENt+hwJj ; KSK, ID: 14321
; KSK DS record for reference:
; user1.example.com. IN DS 44926 7 2 186A498853ED0EE54E.......708481BA231D6CCC11B0D

; RECORDS
@		A	    42.42.42.42
home    CNAME   home.user1.dyn.example.com.
```

For user2:

```zonefile title="/srv/named/user2/user2.example.com.zone"
$ORIGIN	user2.example.com.
$TTL	2h

; DNS
@		SOA	ns1	hostmaster (
				2023151101	; Serial
				8h		    ; Refresh
				30m		    ; Retry
				1w		    ; Expire
				1h )		; Negative Cache TTL
@		NS	ns1
@		NS	ns2
ns1		A	42.42.42.42
ns2		A	42.42.42.42

; DNSSEC
@   DNSKEY 256 3 7  AwEAAdkH5QoDvqxhggB.......Ibjxe8u2tyoCHt YLkcCkopCZE= ; ZSK, ID: 21234
@   DNSKEY 257 3 7  AwEAAao4cLBqQHdhLf9.......lyanhctcMI kPZGNrn0vxtkvg4T ; KSK, ID: 24321
; KSK DS record for reference:
; user2.example.com. IN DS 32994 7 2 A5A0B06E98D776BBC3.......C4A98483357BB637A3768

; RECORDS
@		A	    42.42.42.42
```

For dynamic zone (this is only the initial zonefile, it will be overwritten by the server when dynamic changes are performed - commenting is useless for this reason):

```zonefile title="/var/named/dyn.example.com.zone"
$ORIGIN	dyn.example.com.
$TTL	2h

@		SOA	ns1	hostmaster (
				2023151101
				8h
				30m
				1w
				1h )
@		NS	ns1
@		NS	ns2
ns1		A	42.42.42.42
ns2		A	42.42.42.42

@   DNSKEY 256 3 7  AwEAAbxFBaMjSI0asdm.......zAqrVViCHGdvHe Du/wTG6Qbt8= ; ZSK, ID: 91234
@   DNSKEY 257 3 7  AwEAAcQPOmgD/V4qWHH.......Exj2ymiVoS Yg7Q5qnGNPWC1B+P ; KSK, ID: 94321
; KSK DS record for reference:
; dyn.example.com. IN DS 22180 7 2 00D2BE8B7BA10B11CA.......26C7623A0B69B9FF55311
```



# ddclient config
User user1 copies their TSIG key "user1key" in `/etc/ddclient/user1key.key`, then they modify `/etc/ddclient/ddclient.conf` to match the following:

```conf title="/etc/ddclient/ddclient.conf"
use=web, web=ipinfo.io/ip
protocol=nsupdate
server=ns1.dyn.example.com
password=/etc/ddclient/user1key.key
zone=dyn.example.com
home.user1.dyn.example.com
```


# Other files
Besides the mentioned files, we also have
- private keys, readable by user `named` and in directories writable by user `named`, in `/var/named/keys/`, `/srv/named/user1/keys/`, `/srv/named/user2/keys/` and in user1's home server in `/etc/ddclient/user1key.key` (the last one has nothing to do with `named`).
- public keys which could technically be deleted but it's best to keep for reference
- accurate directory permissions for `/var/named` (only `named` for read and write), `/srv/named/user1/` (`named` for read and `user1` for read and write) and the same for `/srv/named/user2/`, `/etc/ddclient/` on user1's home machine (only `root` has read and write, especially to `user1key.key`)
