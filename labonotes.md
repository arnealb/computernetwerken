# Labo 3

## DNS Resolving
**Resolving** verwijst naar het proces waarbij een achterliggend programma de DNS-server vraagt om het IP-adres dat correspondeert met de opgegeven URL. Dit kan ook handmatig uitgevoerd worden door een gebruiker.

### Manuele Resolving Voorbeelden

```bash
student@cnet:~$ host www.ugent.be
www.ugent.be has address 157.193.43.50
student@cnet:~$ nslookup www.ugent.be
Server: 10.0.2.1
Address: 10.0.2.1#53
Non-authoritative answer:
Name: www.ugent.be
Address: 157.193.43.50
```

- bij **nslookup** (in windows / linux kan ook host of dig): 
```bash
student@cnet:~$ host <gezochte URL> [DNS-server IP-adres of naam]
student@cnet:~$ host www.ugent.be 84.200.69.80
Using domain server:
Name: 8.8.4.4
Address: 8.8.4.4#53
Aliases:
www.ugent.be has address 157.193.43.50
```

- bij **dig** kan ook een andere serverkiezen en enkel een AAAA record opvragen 
``` bash
student@cnet:~$ dig @8.8.8.8 +short AAAA www.belnet.be
2a00:1c98:10:2c::10
```

kan ook extra informatie opvragen zoals bvb de servername (waarbij NS de naam van de servers van een domijn opvraagt)
``` bash
dig +short NS ugent.be
```

- **reverse lookup**: opzetten van een IP-adres naar zijn DSN-naam
``` bash
dig + short -x 157.193.215.171
www.test.atalantis.ugent.be
```

<br>
<br>
<br>
<br>
<br>

# oplossen van de vragen
1. Krijg je een antwoord als je www.ugent.beresolved? Van welke DNS-serverkotm het antwoord?
``` bash
nslookup/dig www.ugent.be

antwoord: 
student@cnet:~$ nslookup www.ugent.be
Server:		10.0.2.3
Address:	10.0.2.3#53

Non-authoritative answer:
Name:	www.ugent.be
Address: 157.193.43.50
```

2. Krijg je een antwoord als je www.ugent.be resolved bij server 8.8.4.4? Of bij 1.1.1.1?

``` bash
voor google : 
nslookup www.ugent.be 8.8.4.4
of
dig @8.8.4.4 www.ugent.be

antwoord: 
Server:		8.8.4.4
Address:	8.8.4.4#53

Non-authoritative answer:
Name:	www.ugent.be
Address: 157.193.43.50
```

3. Kan je rechtstreeks een antwoord opvragen bij de DNS-server van jouw host system?
<br>
``` bash
dunno dis one tho
```


<br>
<br>
<br>
<br>

## DNS-server caching
- **een caching/recursing DNS-server**: is een server die DNS-query’s van client-apparaten (zoals laptops of desktops) ontvangt en de antwoorden tijdelijk opslaat (cachet). Wanneer een identieke query later opnieuw wordt gesteld, kan de server direct antwoorden vanuit zijn cache, zonder opnieuw het volledige DNS-systeem te raadplegen. Dit proces versnelt DNS-resolutie en vermindert netwerkverkeer.

### 1 bind default: caching server via Root DNS-servers
om aan te maken in linux
eerst naar de root gebruiker gaan met : 
``` bash
sudo -i
```
```bash
apt install bind9
cd /etc/bind
mv named.conf.options named.conf.options.orig
```
nieuw bestand aanmaken met inhoud: 
``` bash
vim named.conf.options
options {
directory "/var/cache/bind";
auth-nxdomain no; # conform to RFC1035
};
```
nog de DNS-software herstrten met: 
``` bash
systemctl reload bind9
systemctl status bind9
```
### 2 Caching server met forwarder naar externe DNS-server
- in het bestand /etc/bind/named.conf.options kan je een forwarder instellen
- **forwarder**: een externe DNS-server aan wie je zelf als server recursieve vragen stelt: zo moet het iteratief werk niet allemaal door de root servers uitgevoerd worden 
- hiervoor kan je het IP van de ISP DNS-server gebruiken of een publieke zoals 1.1.1.1 of 8.8.4.4 
- dit moet je toevoegen aan het bestand: 
``` txt
forwarders {
 <IP-adres van een externe DNS-server (en niet je eigen adres)>;
 };
```

### 3 beheer van de DNS-cache
in voorgaan de shit zal de caching server de antwoorden die hij ontvangen heeft opslaan in zijn eigen lokale DNS-cache. Als dan 2 keer dezelfde vraag krijgkt gaat hij naar zijn eigen cache antwoorden, niet uit de root servers of uit de forwarder server

zie de stappen om de cache te raadplegen








# vragen oplossen: 
## 1. Test met je Linux client je eigen DNS-server: vraag een query aan via jouw eigen server. Gebruik hiervoor het localhost adres

``` bash
# testen of de server online staat
systemctl status bind9
# de DNS query uitvoeren vai je eigen DNS server op localhost: 
# localhost: 127.0.0.1
# met nslookup: 
nslookup www.ugent.be 127.0.0.1

# met dig: 
dig @127.0.0.1 www.ugent.be 
 
# antwoord: 
Server:		127.0.0.1
Address:	127.0.0.1#53

Non-authoritative answer:
Name:	www.ugent.be
Address: 157.193.43.50
```

### 2. Caching server zonder forwarder: voer de volgende zaken na elkaar uit: 
### a. maak de DNS cache leeg
### b. bekijk je de DNS pakketten die langs de NIC van de server gaan met tcpdump6:root@cnet:~# tcpdump -ni eth0 udp port 53
### c. voer een query uit naar een URL. Voer hem nogmaals uit, en merk dat het antwoord komt zonder extra DNS verkeer in tcpdump

``` bash
# a dns cache leegmaken
rndc flush

# b bekijk de DNS-pakketten met tcpdump
tcpdump -ni eth0 udp port 53

# een dig / nslook uitvoeren in een ander terminal venster om het verkeer te monitoren
dig @127.0.0.1 www.ugent.be

# nu zou je shit moeten zien binnen komen, voer nu nog een keer 
dig @127.0.0.1 www.ugent.be
# uit en nu zou je niets mogen zien
``` 
- c: <br>
**iteratief**: hier communiceert de cliehnt directly met iedere DNS server involved in the lookup
<br>
**recursief**: neemt de verantwoordelijkheid voor het opzoeken van het antwoord op een DNS-query, wanneer je de query verstuurt, vraagt deze het op zijn beurt aan andere servers (root-server / TLD-servers / ...)

- diehard recursief dus


### 3: Werk je DNS-server bij, zodat hij een forwarder contacteert voor zijn eigen aanvragen. Herstart de bind9 server; clear de cache; monitor opnieuw het verkeer zoals in de vorige vraag. Werkt de server nu iteratief of recursief? Leg uit aan de hand het DNS-verkeer dat je kon ‘buitmaken’

``` bash
# zorgen dat hij een forwarder gebruikt:
sudo nano /etc/bind/named.conf.options
# en voeg dit toe: 
options {
    directory "/var/cache/bind";
    auth-nxdomain no; # conform to RFC1035

    forwarders { # deze shit werd toegevoegd
        8.8.8.8;
    };
};

# herstrt de server en clear de cache
systemctl restart bind9
rndc flush

# monitor het verkeer met tcpdump: 
sudo tcpdump -ni eth0 udp port 53

# en in een ander venster: 
dig @127.0.0.1 www.ugent.be

# nog steeds zelfde soort output
```

- nog steeds zelfde soort output, als ik 1 keer iets vraag krijg ik een output bij die tcmp shit en als ik het dan nog een keer vraag, niet meer ---> recursieffff

## DNS-server authoritative
tot nu toe wordt de server door de client gebruikt maar heeft nog geen eigen ifo die in de database van bind9 opgeslaan wordt

### 1 Authoritative DNS info
- per zone waar de DNS-server voor verantwoordelijk is wordt er een config file gemaakt: named.conf.local 
bv: 
``` txt
zone "example.com" {
 type master;
 notify no;
 file "/etc/bind/db.example.com";
};
```
- hier is deze nu verantwoordelijk voor de zone : "example.com"
- informatie voor deze zone is nu opgeslaan in: "db.ecample.com", in de map /etc/bind

#### merk op: 
- nameserver A heeft ook een record voor zichzelf
- hostnamen die eindigen met een punt en andere niet 
<br>
zonder een punt: bv begint met www: verwijst naar een macine die eigenlijk serv1.exaple.com

### 2 opdracht: authoritatieve DNS
- de naamserver
- een aparte naam voor een www server (zoals fiorano.belnet.be), gebaseerd op jouw voornaam;
als IP-adres geef je deze (niet-bestaande) server 10.0.2.33
- een CNAME record die www laat verwijzen naar deze (niet-bestaande) server
- een A record voor de hostnaam van de client (e.g. jouw tweede voornaam)

``` bash 
# aanmaken: 
vim /etc/bind/named.conf.local

# hierin: 
zone "<albrecht>.be" {
    type master;
    file "/etc/bind/db.<jouw_familienaam>.be";
};

# zonebestand aanmaken: 
sudo cp /etc/bind/db.local /etc/bind/db.<albrecht>.be
sudo nano /etc/bind/db.<albrecht>.be

```
```bash
# dit is het zonebestand: 
$TTL 86400

albrecht.be. IN SOA ns.albrecht.be. admin.albrecht.be. (
  2             ; Serial
  604800     ; Refresh
  86400      ; Retry
  2419200    ; Expire
  604800 )   ; Default TTL

; Naamserver voor de zone
IN NS ns.albrecht.be.

; A record voor de naamserver
ns IN A 10.0.2.33

; A record voor de www-server gebaseerd op je voornaam
arne IN A 10.0.2.33

; CNAME record: laat www.albrecht.be verwijzen naar arne.albrecht.be
www IN CNAME arne.albrecht.be.

; A record voor een andere hostnaam, bijvoorbeeld je tweede voornaam
patrick IN A 10.0.2.16
```
om de een of andere reden kan hij dat bestand niet vinden fz dus kanker ma boeie kga iets fout gedaan hebben suk mie nuts





