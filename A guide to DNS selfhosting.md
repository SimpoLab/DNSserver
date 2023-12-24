[Repository version](https://github.com/SimpoLab/DNSserver)

BIND (or named) is the most widely used Domain Name System (DNS) server.

As a disclaimer, I am not an expert on the matter by any means. I'm just reporting as a guide what I have done on my server and what I found to work.

[[Example BIND config files|TL;DR: example config files]] ^3f95f5

You probably want to configure your domain registrar to set their NS+A records for your domain to point to your server.
See your provider's documentation.

The main BIND configuration file is situated in `/etc/named.conf`.

Keep in mind that when debugging it can be useful to delete cached files, e.g.:

```sh
systemctl stop named.service
find /var/named \( -name "*.jbk" -o -name "*.jnl" -o -name "*.signed" -o -name "*.state" \) -delete
systemctl start named.service
```



# Zones
A zone is a hierarchical, independently configured set of subdomains.
A basic configuration of a zone in `named.conf` looks like this

```named title="named.conf"
zone "example.com" {
	type master;
	file "example.com.zone";
};
```

The zone file, specified by `file`, tells the server what DNS record to host, and usually looks something like this:
The `file` attribute takes either an absolute path or one relative to the main directory (`options/directory` in `named.conf`, by default `/var/named`)

```zonefile title="example.com.zone"
; this is a comment
$ORIGIN	example.com.    ; names which are not completely specified (i.e. don't end with .) are assumed to end with $ORIGIN; also @:=$ORIGIN
$TTL	2h              ; default time-to-live

; generic record syntax:
; <host> [IN] <record type> <record content>
; you can also omit the host if it's the same of the line above

; DNS meta-records (records about DNS itself)
@		SOA	ns1	hostmaster (
				2023111401	; Serial
				8h		    ; Refresh
				30m		    ; Retry
				1w		    ; Expire
				1h )		; Negative Cache TTL
		NS	ns1
	    NS	ns2
ns1		A	42.42.42.42
ns2		A	42.42.42.42

; RECORDS
@		A	42.42.42.42
test	CNAME	@
```

You can easily find information about the record types online.
Basically, the SOA record gives information about the server, the NS records give where to find the authoritative name servers for the domain, and have to point to an A record (and/or AAAA) with the IP of the servers themselves.

When you modify a zonefile you have to increment the serial (in this case we use the convention YYYYMMDDnn, where nn is just an incremental integer) and then reload BIND (if using systemd, just run `systemctl reload named.service`).



# DNSSEC
DNSSEC adds cryptographic signatures to DNS records to make the protocol secure.
See e.g. [Cloudflare](https://www.cloudflare.com/dns/dnssec/how-dnssec-works/)'s introduction to have an idea of how the protocol works and what the various DNSSEC record types do.

First, you need to update the configuration of your zone, e.g.:
```named title="named.conf"
zone "example.com" {
	type master;
	auto-dnssec maintain; // deprecated but simpler version
	inline-signing yes;
	key-directory "keys/";
	file "example.com.zone";
};
```

In said directory, create two key pairs, one for the Zone Singing Key (ZSK) and one for the Key Signing Key (KSK):
```sh
dnssec-keygen -a NSEC3RSASHA1 -b 2048 -n ZONE example.com
dnssec-keygen -f KSK -a NSEC3RSASHA1 -b 4096 -n ZONE example.com
```
Each command will generate a `.key` and a `.private` file.
Make sure the `.private` files are readable by the user BIND runs as (typically `named`).
The content of each `.key` file is a DNSKEY record which has to be added to your zone.
We suggest to add its id and key type (contained in the file name as well as in the file itself as a comment) in the zonefile as a comment:
```zonefile title="example.com.zone"
; DNSSEC
@		DNSKEY	256 3 7 YRVJMpS3.........FhqKZ= ; ZSK, ID: 12345
@		DNSKEY	257 3 7 FGebrFGC.........tVHGYV ; KSK, ID: 54321
```

Because of how the protocol works, the parent zone has to contain a DS record with the hash of the DNSKEY for the KSK.
You can generate such record with the command:
```sh
dnssec-dsfromkey Kexample.com.+007+54321.key
```

The result, similar to the following, must be added to the parent zone.
In case you are securing your domain, you have to add it in the domain registrar's interface (see their docs).
```zonefile
example.com. IN DS 20716 7 2 AE1F5C4F.........A12F12AB6F3 ; KSK, ID: 54321
```

We suggest to add the DS record as a comment in the child zonefile too.

Finally, reload BIND.



# Subzones
Say you want to independently configure a subdomain and its subdomains, e.g. to give a way to user1 to manage their zone user1.example.com.
You can do that by defining a zone user1.example.com and correctly pointing at it in the zone example.com.

We define a new zone in `named.conf` as follows:
```named title="named.conf"
zone "user1.example.com" {
	type master;
	file "/srv/named/user1/user1.example.com.zone";
};
```
Where `user1.example.com.zone` is owned by user1 but read/writable by the `named` user.

We then manage the subzonefile as usual (with SOA, NS and A records) and in the parent zonefile we add NS and A records for the child zone:

```zonefile title="example.com.zone"
; USER1 SUBZONE
user1	    NS	ns1.user1
user1	    NS	ns2.user1
ns1.user1	A	42.42.42.42
ns2.user1	A	42.42.42.42
```

A reload makes our changes effective.


## DNSSEC
In order to implement DNSSEC in our subzones we just make the new zone compliant to the protocol.

As before (see [[#DNSSEC]]), we update `named.conf` by adding DNSSEC, we generate ZSK and KSK key-pairs and add DNSKEY records to the zonefile `user1.example.com.zone`.
We also add the DS record for the KSK in the parent zonefile (`example.com.zone`) under the host `user1`:

```zonefile title="example.com.zone"
user1 IN DS 20716 7 2 NR1S5P4S.........N12S12NO6S3 ; KSK, ID: 69420
```

Make sure che `key-directory` is read-writable by both `user1` and `named`, as for the zonefile.



# Dynamic DNS

[DDNS](https://en.wikipedia.org/wiki/Dynamic_DNS) automatically updates a DNS record.
You can use it e.g. for keeping a record pointing to your home's IP address, even if such address changes periodically.

In order to configure BIND so that we can use DDNS with it, it's better to define a zone for that specific purpose.
This is because the dinamically managed zones should be separated, as their zonefiles are modified by the server itself.

BIND provides a convenient command to generate configuration snippets for DDNS:
```sh
ddns-confgen -k mainkey -z dyn.example.com
```

As explained by the command output, the first code snippet adds to our configuration (`named.conf`) a symmetric key^[The key generation performed by `ddns-confgen` can be done independently by using `tsig-keygen`.], which we'll use to update our record dynamically:
```named title="named.conf"
key "mainkey" {
        algorithm hmac-sha256;
        secret "ppeue5dzUkw.........MulajQySI=";
};
```
We also add the snippet to a file `mainkey.key` which we'll use later.

The second part has to be added to the zone configuration and allows the key to modify the zone records.
In our case we added a zone `dyn.example.com`:

```named title="named.conf"
zone "dyn.example.com" {
	type master;
	file "dyn.example.com.zone";
	update-policy {
		grant mainkey zonesub ANY;
	};
};
```

Remember to add the subzone as [any other child zone](#Subzones).
Before reloading the server, configure an initial zonefile `dyn.example.com.zone` with the usual DNS authoritative records, as well as DNSSEC records if you want them.
The file serves as a starting point but will be overwritten by the dynamic updates.

Once reloaded the server, you may dynamically add records via the `nsupdate` command, provided that you have the key.
This also works remotely^[You may omit the server command if the server is local.].
For example `nsupdate -k mainkey.key` will open an interactive command line, in which you may type:

```nsupdate
server ns1.dyn.example.com
zone dyn.example.com
update add test.dyn.example.com 3600 TXT "dynamic updates work!"
send
```

Instead of writing a custom nsupdate script in order to update an A record, you may use tools like [ddclient](https://ddclient.net/), which can be used with many DDNS protocols, included nsupdate.
You may install ddclient in a machine which has a copy of `mainkey.key` and is located at the IP we want the A record to contain.

An example ddclient config might look like this

```conf title="ddclient.conf"
use=web, web=ipinfo.io/ip
protocol=nsupdate
server=ns1.dyn.example.com
password=/etc/ddclient/mainkey.key
zone=dyn.example.com
home.dyn.example.com
```

The client will periodically check if its IP address changed, and if so it will use the given key to remotely update an A record for the name `home.dyn.example.com`.


## Dynamic subzones with multiple keys
You might need multiple users to have dynamic records, but you don't want each user to have their own dynamic zone or users to be able to modify each other's records.
This problem is solved by creating a dynamic zone `dyn.example.com`, and only allowing user1 to modify records ending in `user1.dyn.example.com`.

In order to do that, we need to tweak our usual dynamic zone configuration (we assume DNSSEC set up):

```named title="named.conf"
key "user1key" {
        algorithm hmac-sha256;
        secret "ppeue5dzUkw.........MulajQySI=";
};

key "user2key" {
        algorithm hmac-sha256;
        secret "ccrhr5qmHxj.........ZhynwDlFV=";
};

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
```

Where `user1key` and `user2key` are generated using `tsig-keygen` (or `ddns-confgen` as earlier).
A copy of `user1key` will be provided to user1 and a copy of `user2key` to user2.

Of course a more precise permission scheme can be set up, see `named.conf(5)`.



# Tweaks


## Disable ipv6
At the moment we disabled ipv6 because it gave us problems and we don't really need it right now.
You can do that by overriding the systemd unit file:

```sh
systemctl edit named.service
```

adding the flag `-4`:

```systemd
[Service]
ExecStart=
ExecStart=/usr/bin/named -4 -f -u named
```

You can then disable ipv6 listening by having the following line commented:
```named
listen-on-v6 { any; }
```



# Sources
- [BIND docs](https://bind9.readthedocs.io/en/latest/chapter3.html)
- [BIND ArchWiki page](https://wiki.archlinux.org/title/BIND)
- [Cloudflare's introduction to DNSSEC](https://www.cloudflare.com/dns/dnssec/how-dnssec-works)
- [ddclient docs](https://ddclient.net/protocols.html#nsupdate)



# License
This work is licensed under a [CC BY-NC-SA 4.0 license](https://creativecommons.org/licenses/by-nc-sa/4.0/legalcode)
