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

```bash
student@cnet:~$ dig @8.8.8.8 +short AAAA www.belnet.be
2a00:1c98:10:2c::10
```

kan ook extra informatie opvragen zoals bvb de servername (waarbij NS de naam van de servers van een domijn opvraagt)

```bash
dig +short NS ugent.be
```

- **reverse lookup**: opzetten van een IP-adres naar zijn DSN-naam

```bash
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

```bash
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

```bash
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

```bash
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

```bash
sudo -i
```

```bash
apt install bind9
cd /etc/bind
mv named.conf.options named.conf.options.orig
```

nieuw bestand aanmaken met inhoud:

```bash
vim named.conf.options
options {
directory "/var/cache/bind";
auth-nxdomain no; # conform to RFC1035
};
```

nog de DNS-software herstrten met:

```bash
systemctl reload bind9
systemctl status bind9
```

### 2 Caching server met forwarder naar externe DNS-server

- in het bestand /etc/bind/named.conf.options kan je een forwarder instellen
- **forwarder**: een externe DNS-server aan wie je zelf als server recursieve vragen stelt: zo moet het iteratief werk niet allemaal door de root servers uitgevoerd worden
- hiervoor kan je het IP van de ISP DNS-server gebruiken of een publieke zoals 1.1.1.1 of 8.8.4.4
- dit moet je toevoegen aan het bestand:

```txt
forwarders {
 <IP-adres van een externe DNS-server (en niet je eigen adres)>;
 };
```

### 3 beheer van de DNS-cache

in voorgaan de shit zal de caching server de antwoorden die hij ontvangen heeft opslaan in zijn eigen lokale DNS-cache. Als dan 2 keer dezelfde vraag krijgkt gaat hij naar zijn eigen cache antwoorden, niet uit de root servers of uit de forwarder server

zie de stappen om de cache te raadplegen

# vragen oplossen:

## 1. Test met je Linux client je eigen DNS-server: vraag een query aan via jouw eigen server. Gebruik hiervoor het localhost adres

```bash
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

```bash
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

```bash
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

```txt
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

```bash
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

## DNS Client - bijwerken default server

### 1. Client default DNS configuratie

- voe de moment gebruit de host een externe DNS-server --> willen eigen DNS server gebruiken
- de DNS server die de client gebruikt kan je terug vinden in "/etc/resolf.conf"

##### DNS resolutie met korte namen:

- korte namen binnen een domein dat automatisch uitgebreid wordt tot een volledige domeinnaam
- aanpassen in "/etc/resolv.conf"

##### probleem:

- deze wijziging is niet blijvend, wordt vaak dynamisch overschreven

##### resolvconf:

- gebruik dit om handmatig wijziginen aan te brengen in /etc/resolf.conf

##### toevoegen van een vase DNS-server

```bash
/etc/resolfconf/resolv.conf.d/base # -> set to 8.8.4.4
resolvconf -u
```

- testen of inorde

```bash
grep name /etc/resolv.conf
```

### 2. opdracht eigen dns bijwerken

- aanpassen naar 127.0.0.1

# Labo 4

## Listening sockets - deamon software

### 1. ss: show sockets

- tcp poort nummer op een server hangt vast aan een sockets -> voor TCP SYN te ontvangen = **listening socket** met **ss -l** kunnen we dit opvragen

```bash
ss -t
ss -tln
```

### 2 deamon: server software

- **deamon**: is een applicatie die opgestart wordt op een computer, los van andere gebruikers, deze doftware is permanent actief
  <br>
  als dit met het IP-adres gelinkt zijn wordt een listening socket aangemaakt die een poortnummer bindt aan een bepaalde daemon

```bash
systemctl status nginx
# starten / stoppen
systemctl start nginx
systemctl stop nginx
# enabled / dissabled
systemctl disable nginx
systemctl enable nginx
```

- de config van een daemon staat in /etc -> bij aanpassen moet **systemctl reload@**

### opdracht

### 1. Bekijk welke poorten er allemaal actief zijn op jouw VM. Stop de DNS daemon bind9, en bemerk het verschil. Welke poorten gebruikt de DNS daemon allemaal?

```bash
# hoeveel actief
ss -tln
# stoppen
systemctl stop bind9
ss -tln


```

### 2. Verwijder het programma ‘cups-daemon’: apt purge cups-daemon Welke poort was er in gebruik door de software (voor printer-services die we niet gebruiken)?

```bash
apt purge cups-daemon
```

### 3. Installeer de software apache2. Kan je merken welke poort er actief geworden is?

```bash
sudo apt install apache2
ss -tln

```

### 4. Installeer de software nginx. Deze daemon wil echter niet opstarten. Leg uit waarom.

```bash
sudo apt install nginx
sudo systemctl start nginx
sudo systemctl status nginx

```

- shit werkt wel gwn tho, wss probeert te luisteren op poort 80, maar deze is al ingebruik door apache2

### 5. Stel de poort van apache2 in op 8080 – dit kan via het bestand /etc/apache2/ports.conf. Herlaad deze daemon. Kan je de wijziging zien in je listening sockets? Zou je nu nginx kunnen opstarten? Leg uit.

```bash
sudo vi /etc/apache2/ports.conf

"Listen 8080"
sudo systemctl reload apache2
ss -tln
sudo systemctl start nginx

# om te checken of zeker gelukt is
lsof -i -P -n | grep apache2
```

## extra:

### wget:

- laat toe om met http een bestand van een webserver te downloaden en op te slaan als een bestand

```bash
wget 157.193.215.171/pearson.png
```

### curl:

- laat toe om met http een bestand van een webserver te doqnloaden en weer te geven op de CLI

```bash
curl www.ugent.be
```

## Active sockets - client en deamon software

### 1. Client ports

- als de host geen enkele tcp verbinding opgestart heeft, zien we dat er op het systeem geen enkele tcp socket actief is

```bash
ss -tn
```

### 2. Netcat - testn van listening sockets

- **netcat**: wordt gebruikt om een tcp verbinding te openen met een listening socket
- gelijkaardige mogelijkheden als telnet, maar uitgebreidere opties

```bash
comnet1@home:~$ netcat mail.test.atlantis.ugent.be 110
# antwoord:
        +OK Hello there.
        USER comnet1
        +OK Password required.
        PASS XXXXX
        +OK logged in.
        LIST
        +OK POP3 clients that break here, they violate STD53.
        .
        QUIT
        +OK Bye-bye.
```

- laat toe snel te testen of een bepaalde poort op een server geactiveerd is of niet

```bash
netcat -vz -w 1 wwww.standaard.be 80
## antwoord
Connection to www.standaard.be 80 port [tcp/http] succeeded!

comnet1@home:~$ netcat -vz -w 1 www.standaard.be 21
## antwoord
netcat: connect to www.standaard.be port 21 (tcp) timed out
```

## Netcat - opzetten van testes

- kan er listening socket mee aanmaken

```bash
netcat -v -l -p 6789
```

- het ip adres of het localhost adres van de node kan ook op dit poortnummer aangesproken worden

```bash
netcat localhost 6789
```

## opdracht

#### 1. Maak een SSH verbinding naar home.test.atlantis.ugent.be ; bekijk voordien en nadien de uitkomst met ss –tn. Leg uit vanaf wanneer een poortnummer in gebruik is op een client.

```bash
ssh home.test.atlantis.ugent.be
ss -tn
```

- zal niewe verbinding opgezet worden naar poort 22 (de poort voor ssh)

#### 2. Maak nadien een tweede SSH verbinding vanaf dezelfde client. Leg uit a.d.h.v. het resultaat van ss hoe de pakketten van deze beide verbindingen door de computers uit elkaar kunnen gehouden worden.

```bash
ssh home.test.atlantis.ugent.be
ssh home.test.atlantis.ugent.be
```

- Uitleg: Elke verbinding wordt onderscheiden door een unieke 4-tuple: <src IP, dest IP, src port, dst port>. Hoewel beide verbindingen hetzelfde src IP, dest IP en dst port hebben, verschillen ze in de bronpoort (source port) op de client. Deze unieke combinatie zorgt ervoor dat de twee verbindingen niet door elkaar gehaald worden.

#### 3. Bekijk de listening sockets op deze server. Leg van 3 poorten uit welke functie ze vervullen op de server. Hint: bekijk de inhoud van het bestand /etc/services.

- poort 22: ssh
- poort 80: http, webverkeer zonder encryptie
- poort 443: https

#### 4. Je installeerde reeds de nginx webserver. Kan je met netcat testen of hij werkt op jouw systeem? Hoe doe je dat? Kan je met wget het index.html downloaden?

```bash
netcat -vz localhost 80
# antwoord:
Connection to localhost 80 port [tcp/http] succeeded!

wget http://localhost
# je krijg juiste output
```

#### 5. Je installeerde reeds de apache2 webserver, en werkte het poortnummer bij. Kan je met netcat testen of hij werkt op jouw systeem? Hoe doe je dat? Kan je met wget het index.html downloaden?

```bash
netcat -vz localhost 8080
wget http://localhost:8080
```

#### 6. Stel een poortnummer open op je computer (netcat listening socket), zodat je er vanuit een 2e terminal met netcat mee kan verbinden. Kan je dit combineren met input/output redirection (zie vorig labo), zodat je het bestand /etc/services kan sturen van de ene terminal naar de andere doorheen deze TCP socket? Hint: http://www.microhowto.info/howto/copy_a_file_from_one_machine_to_another_using_netcat

```bash
# opstarten
netcat -l -p 6789

# ander venster
netcat localhost 6789

# input redirection
cat /etc/services | netcat localhost 6789
```

## Socets - programming in python3

### 1. TCP server

```python
## maakt een socket aan op poort 678
from socket import *

s = socket(AF_INET, SOCK_STREAM)
s.bind(('', 678))
s.listen(1)

while True:
    c, addr = s.accept()
    g = ("Hello %s" % addr[0])
    c.send(bytes(g, encoding='utf-8'))
    c.close()
```

### 2. TCP client

```python
# een primitieve client die verbindt met deze socket

from socket import *

s = socket(AF_INET, SOCK_STREAM)
s.connect(('127.0.0.1', 678))
s.send('Hello World'.encode())
data = s.recv(1024)
s.close()

print('Received', data)
```

## opdracht

#### 1. Kan je het TCP gesprek capturen met WireShark? Op welke interface werk je?

- de server

```python
from socket import *

# Maak een TCP/IP-socket op poort 6789
s = socket(AF_INET, SOCK_STREAM)
s.bind(('', 6789))
s.listen(1)

print("Server is listening on port 6789...")

while True:
    # Accepteer inkomende verbindingen
    c, addr = s.accept()
    print(f"Connection from {addr}")

    # Stuur een begroeting terug, inclusief het IP-adres van de client
    g = f"Hello {addr[0]}"
    c.send(bytes(g, encoding='utf-8'))
    c.close()
```

- de client

```python
from socket import *

# Maak verbinding met de server op het IP-adres van jouw VM en poort 6789
s = socket(AF_INET, SOCK_STREAM)
s.connect(('10.0.2.15', 6789))  # Vervang '10.0.2.15' door jouw VM IP

# Stuur een bericht naar de server
s.send('Hello World'.encode())

# Ontvang het antwoord van de server
data = s.recv(1024)
s.close()
print('Received', data)
```

- uitvoeren

```bash
python3 tcp-server.py
python3 tcp-client.py
```

#### 2. Kan je de server code aanpassen zodat hij het client poortnummer teruggeeft, i.p.v. het IP-adres van de client?

// low key skipped ket get

- gebruik tcp.port == 6789 in wireshark

- pas de servercode aan:

```python

from socket import *

# Maak een TCP/IP-socket op poort 6789
s = socket(AF_INET, SOCK_STREAM)
s.bind(('', 6789))
s.listen(1)

print("Server is listening on port 6789...")

while True:
    # Accepteer inkomende verbindingen
    c, addr = s.accept()
    print(f"Connection from {addr}")

    # Stuur het client poortnummer terug, in plaats van het IP-adres
    g = f"Hello, your port is {addr[1]}"
    c.send(bytes(g, encoding='utf-8'))
    c.close()

# dit zou de output moeten zijn
Received b'Hello, your port is 12345'
```

## extra 2: secure CoPy

-**scp commando**: laat toe bestanden te kopieren doorheen een ssh verbinding, is een veilige ftp verbinding

zie wc voor die extra shit

## Socket scanning: nmap

**Nmap** is een krachtige tool voor het uitvoeren van **portscans** op netwerken en apparaten. Het helpt bij het detecteren van open poorten door **SYN-pakketten** naar specifieke poorten te sturen en te analyseren hoe de host reageert.

- **Open poort**: Als de server een SYN/ACK terugstuurt, betekent dit dat er een **listening socket** actief is en de poort open is.
- **Filtered poort**: Als er geen reactie is (meestal door een firewall die de SYN-pakketten blokkeert), wordt de status als **filtered** gemarkeerd.

Nmap start standaard met een **ping-probe** om te controleren of een host actief is. Met de optie `-Pn` wordt deze probe overgeslagen, wat nuttig is bij firewalls die ping-verkeer blokkeren. Nmap ondersteunt ook het scannen van meerdere poorten tegelijk, en er zijn talloze opties om gedetailleerde netwerk-informatie te verzamelen.

## opdrachten

#### 1. Voer een basisscan uit op scanme.nmap.org. Welke poorten zijn open? Welke poorten werden allemaal getest?

```bash
nmap scanme.nmap.org
```

#### 2. Voer een scan uit op de server home.test.atlantis.ugent.be. Welke poorten zijn er allemaal actief op deze server?

- nmap test de meest voorkomende poorten

```bash
nmap home.test.atlantis.ugent.be
```

#### 3. Start WireShark op, en start capturing. Scan met nmap je eigen Linux VM op poort 25. Stop het capturen. Welke status krijg je terug? Kan je dit linken aan de TCP-pakketten die je ziet?

```bash
nmap -p 25 localhost
```

#### 4. Herhaal dit experiment, maar test nu poort 25 op de server. home.test.atlantis.ugent.be. Welke status krijg je terug? Kan je dit linken aan de TCP-pakketten die je ziet?

```bash
nmap -p 25 home.test.atlantis.ugent.be
```

#### 5. Na stap 3 en 4 zou je het onderscheid moeten kunnen maken tussen open en filtered als status. Voer een basis portscan uit op www.meemoo.be, en formuleer welk advies je zou kunnen geven aan de beheerder van de firewall van deze server.

# Labo 5: ip routing mininet

- dit labo: De organisatie van ip-ranges voor kleine (bedrijfs)netwerken | hoe ip-adressen op hosts en op routers ingesteld worden

## Emulatie: mininet

- meerdere hosts en viruele router organiseren door meerdere VMs aan elkaar te schakelen in VMware of VirtualBox
- We hebben een omgeveing nodig waar verschillende netwerkkaarten kunnen op bestaan en waar de nodige applicatielaag software op actief kan zijn.
  -> **Mininet**

```bash
# eerst shit installeren:
sudo –s ## (werk als root)
wget http://157.193.215.171/cnet_lab_IProuting.py
python cnet_lab_IProuting.py

# kan met nodes de ingeladen nodes opvragen
nodes
# geeft: available nodes are:
#        c0 ftp rISP rout s1 s2 s3 s4 visit web ws1 ws2 ws3


# kan je met mininet een CLI opstarten voor elke node met xterm <naam van node>
xterm ws1 # dit opent nieuw cli venster

```

## your address range

# opdracht:

# Computernetwerken II - cnet2

# Lab 4: IP routing

> (base 02105980) - complete your personal information below

date:

name:
year:
name:
year:

## Design

### IP addresses

> These are the assigned IP address for the different hosts in your network.

| Host                         | IP address                          |
| ---------------------------- | ----------------------------------- |
| workstation1                 | 10.20.82.243                        |
| workstation2                 | 10.20.82.244                        |
| webserver                    | 10.20.82.28                         |
| ftpserver                    | 10.20.82.62                         |
| Host in 12-host network      | 10.20.82.180                        |
| Uplink address on ISP router | 10.20.82.254/<Smallest SN possible> |

### Subnetworks calculated

> Fill in the resulting network information in the table below; use the format as in the theoretical exercises (Linux differs on some points in itâ€™s display).

| Network | Netmask | Interface |
| ------- | ------- | --------- |

xterm ws1 # dit opent nieuw cli venster

```

## your address range


---- |
|         |         |           |
|         |         |           |
|         |         |           |
|         |         |           |

## Configure the network

### ... the router
> screenshot of the router pinging external node


### ... the hosts
> What do you need to add on the host to make this work?

> screenshot of the workstation pinging the webserver

> Your built-up routing table of the company router (ip route output):


### ... the ISP router
> Which networks are we missing?

> screenshot of the workstation pinging external node

> Explain what general rule make the entry match.
```

## het maken van de shit :

#### 2.1 Configuring interfaces:

```bash
sudo ip address add 10.20.82.241/29 dev rout-eth1   # Workstation subnet
sudo ip address add 10.20.82.1/25 dev rout-eth2     # Server park
sudo ip address add 10.20.82.177/28 dev rout-eth3   # Visitor subnet
sudo ip address add 10.20.82.253/30 dev rout-eth4   # Uplink naar ISP
```

- uitvoeren voor het configureren van de interfaces

```bash
sudo ip route add 10.20.82240/29 dev rout-eth1   # Route naar Workstation subnet
sudo ip route add 10.20.82.0/25 dev rout-eth2     # Route naar Server park
sudo ip route add 10.20.82.176/28 dev rout-eth3   # Route naar Visitor subnet
sudo ip route add 10.20.82.252/30 dev rout-eth4   # Route naar ISP uplink
sudo ip route add default via 10.20.82.254        # Standaardroute naar ISP
```

- uitvoeren voor de routes te configureren

door nu uit te voeren krijg ik dit:

```bash
ip route
```

![foto](/labo5_ip_routing/fotos/routes.png)

#### nu de routes in de verschillende hosts uitvoeren

- als de foutmelding komt dat de device niet gevonden werd:

```bash
ip link show
#wat nu na de "2:" staat is de device name
```

- voor ws1

```bash
sudo ip address add 10.20.82.2/25 dev ws1-eth0
sudo ip route add default via 10.20.82.1
```

## effectief inorde nu:

- voor routing table zie goodnotes ofc

#### 1. router instellen

- alle subnetten

```bash
sudo ip address add 10.20.82.241/29 dev rout-eth2  # Workstation subnet
sudo ip address add 10.20.82.1/26 dev rout-eth3   # Server subnet
sudo ip address add 10.20.82.177/28 dev rout-eth4 # Visitor subnet
sudo ip address add 10.20.82.253/30 dev rout-eth1 # Uplink naar ISP
```

- default route zodat verkeer naar het internet via de ISP-router gestuurd wordt

```bash
sudo ip route add default via 10.20.82.254
# testen
ping 10.20.82.254
ping 8.8.8.8
```

- config workstation1

```bash
sudo ip address add 10.20.82.243/29 dev eth0
sudo ip route add default via 10.20.82.241

ping 10.20.82.241  # Ping de router
```

- config workstation2

```bash
sudo ip address add 10.20.82.244/29 dev eth0
sudo ip route add default via 10.20.82.241
ping 10.20.82.241  # Ping de router
```

- config webserver

```bash
sudo ip address add 10.20.82.28/26 dev eth0
sudo ip route add default via 10.20.82.1
ping 10.20.82.241  # Ping de router
```

- config ftpserver

```bash
sudo ip address add 10.20.82.62/26 dev eth0
sudo ip route add default via 10.20.82.1
ping 10.20.82.1    # Ping de router
ping 10.20.82.243  # Ping een Workstation
```

- config visitor

```bash
sudo ip address add 10.20.82.178/28 dev eth0
sudo ip route add default via 10.20.82.177
ping 10.20.82.177  # Ping de router
ping 10.20.82.243  # Ping een Workstation
```

- laatste deel: van de isp server: isp moet @eten hoe verkeer naar de bedrijsnetwerken meot worden geretourneerd:

```bash
sudo ip route add 10.20.82.0/24 via 10.20.82.253
```

- nu kan van uit alle hosts naar alles geping worden

# lab6: L2 configurations

## Netwerk beschrijving

### 1: Gegevens over de set-up

- de tabellen

```txt
Hostname       IP-address        VLAN    L2 connection
_______________________________________________________
PC10           10.20.101.10/26   10      S1 - Fa0/10
PC50           10.20.101.50/26   10      S2 - Fa0/10
_______________________________________________________
PC78           10.20.101.78/26   20      S1 - Fa0/18
PC96           10.20.101.96/26   20      S3 - Fa0/18
_______________________________________________________
PC182          10.20.101.182/25  33      S3 - Fa0/6
PC193          10.20.101.193/25  33      S2 - Fa0/6
CompServ       10.20.101.253/25  -       S0 - Gig1/0/6
_______________________________________________________

De switches onderling zijn met elkaar verbonden met de volgende ethernet-poorten:

Device    Port         Port        Device
S0        Gig1/0/1     Fa0/24      S1
_______________________________________________________
S0        Gig1/0/2     Fa0/24      S2
_______________________________________________________
S0        Gig1/0/3     Fa0/24      S3
_______________________________________________________
S0        Gig1/0/4     Gig0/0      Rout
_______________________________________________________
S2        Fa0/23       Fa0/24      S1
_______________________________________________________
S2        Fa0/22       Fa0/24      S3
_______________________________________________________

```

### 2: Opdracht: subnetten

#### Bereken de route interface voord e 3 subneten, kies het hoogst mogelijke IP van het subnet.

- vlan 10: 10.20.101.0 /26
- vlan 20: 10.20.101.64 /26
- vlan 33: 10.20.101.128 /25

**bereken het broadcast-adres van elk subnet**

vlan 10: 10.20.101.0 /26 : van 0 -> 63: broadcast-adres: 10.20.101.63
vlan 20: 10.20.101.64 /26 : van 64 -> 127: broadcast-adres: 10.20.101.127
vlan 33: 10.20.101.128 /25 : van 128 -> 255: broadcast-adres: 10.20.101.255

**Kies hoogste bruikbare ip adres en toewijzen van de router interfaces**
vlan 10: 10.20.101.62
vlan 20: 10.20.101.126
vlan 33: 10.20.101.254

#### 3: Stel deze berekende adressen in als de default gateway bij alle 6 de host PC's, configureer

#### ook de Dns-server van het bedrijf (DNS server is 10.20.101.253)

- stel voor iedere pc in zijn vlan in met de juiste default gateway, en stel dezelfde
  DNS-server in (dit is gelijk voor iedere pc)

#### 4: Nog niet kan pingen van geen enkele host naar een andere host

- ik heb gdn met in pc50 te gaan en daar bv dit uit te voeren

```bash
ping 10.20.101.78
```

- request timed out / send x aantal en geen kanker terug gekregen

## VLAN: access ports

- een host stuurt ethernet verkeer naar de switch als een nrml ethernetframe.
  --> dit verkeer toegang tot VLAN? switch die deze toegang bepaalt

### 1: Configuratie access ports

```bash
enable
configure terminal
# of met
en
conf t
# eerste geeft je de nodige rechten
# tweede geeft toegang om te configureren
```

- poort instellen

```bash
int fa0/1
switchport mode acces
switchport access vlan 10
# output: Access VLAN does not exist. Creating vlan 10
```

- overzicht van de ingestelde VLANs kan je krijgen met

```bash
show vlan brief
```

### 2: opdrachten

#### 1: op iedere switch de juiste poorten instellen

- voor switch 1

```bash
#eerst vlan aanmaken
vlan 10
# naam gegeven idk ofdit meot tho
interface Fa0/10
switchport mode access
switchport access vlan 10

vlan 20
interface Fa0/18
switchport mode access
switchport access vlan 20
```

- **Do nu voe alle andere kanker tyfus idk**

#### 2: Ga na dat hosts binnen de VLAN naar elkaar kunnen pingen (PC10 naar PC50; PC182 naar PC193).

#### Ga ook na dat PC78 niet kan pingen naar PC96 – we bestuderen dit later.

- als vanuit pc 10 een ping uitvoer kan dit, aangezien ze in zelfde vlan zitten, bij de
  rest kan dit niet

```bash
ping 10.20.101.50
```

#### 3: Reflecteer over de verbinding s1-s2 en s2-s3. Hoe moeten deze poorten geconfigureerd zijn opdat

#### het netwerk kan werken zoals het nu werkt. Leg uit.

- enlke vlan = virtuele switch, zond, de verbindingen tussen de switches worden statndaard behandeld als access-poorten wat bekeent dat het
  verkeer alleen van 1 specifiek VLAN doorgelaten word

##### hoe kunnen ze werken zoals het nu is?

de verbindingen werken standaard in access-modus en staan in van1, dit betekent dat alleen verkeer van vlan1 wordt ddoorgestuurd tussen switches,
het verkeer van andere vlans wordt niet doorgestuurd omdat er geen trunking is geconfigureerd, hierdoorkunnne host in verschillende vlans of
op verschillende switches niet met elkaar communiceren

## VLANs trunk:

### 1: Configuratie trunk links

- verbindign tussen meerdere switches horen niet tot 1 vlan, switches voegen een VLAN tag toe als het verkeer over meerdere gaat
- ene trunk opstellen:

```bash
int fa0/1
switchport mode trunk
```

- alle trunks opvragen

```bash
show int trunk
```

### 2: opdrachten

#### 1: Stel op swtich s0 de juiste poorten in opdat alle nodige trunks actief worden in het netwerk

- via de cli van van S0

```bash
enable
configure terminal

interface Gig1/0/1
switchport mode trunk
exit

interface Gig1/0/2
switchport mode trunk
exit

interface Gig1/0/3
switchport mode trunk
exit
```

- toon de beschikbare trunks

```bash
show int trunks
```

#### 2: vlan 33 de compserv kan pingen

- uit de cmd prompt van een pc in vlan 33 zoals pc 193

```bash
ping 10.20.101.253

```

- EN DIS SHIT WERKT GWN WJWJWJWJWJWJWJ

#### 3: kan een host een DNS query uitvoeren naar de server

- uit de cmd prompt van een pc in vlan 33 zoals pc 193

```bash
nslookup www.ugent.be
```

## Router-On-a-Stick

- een trunk link naar de router te installeren: itnerface op de router instellen dat hij VLAN
  tags kan interpreteren aka een 'Router-On-a-Stick'

### 1: configuratie subinterface

- een vlan interface toevoegen oip een router: eerst subinterface, deze laat je toe tot een
  welbepaalde vlan, eensmaal de subinterface bestaat, kan je er een ip-adres aan toekennen

```bash
int gig0/0.27
encapsulation dot1Q 27
ip addr 10.20.123.191 255.255.255.192
```

- subinterfaces worden aangemaakt op bestaande interface, deze moet actief gemaakt worden

```bash
int gig0/0
no shit #tis shut but im funny like that
```

- op deze route is en defaul route ingesteld naard e ISP router

```bash
ip route 0.0.0.0 0.0.0.0 203.0.113.254
```

- de routerinstabel op de router kan je checken met

```bash
shop ip route
```

### 2: opdracht : router-on-a-stick

#### 1: Stel op de router de nodige subinterfaces in, zodat de routeer de fefault gateway kan zijn voor elke vlan van je netwerk

- die gig0/0.10 verwijst naar de vlan shit al de rest redeli

dig voe d emoment

```bash
interface Gig0/0.10
encapsulation dot1Q 10
ip address 10.20.101.62 255.255.255.192
exit

interface Gig0/0.20
encapsulation dot1Q 20
ip address 10.20.101.126 255.255.255.192
exit

interface Gig0/0.33
encapsulation dot1Q 33
ip address 10.20.101.254 255.255.255.128
exit

interface Gig0/0
no shutdown
exit

show ip interface brief
```

## STP

### lowkey skipped dit deel, al hele kanker middag mee bezig en ket had

- **Spanning Tree Proctocol**: organiseert het L2 netwerk zodanig dat er geen actieve loops
  kunnen bestaan. afh van welke switch de root switch is kan je een link die je lieven actief hebt toch in blocking mode blijven steken.

Nu zijn er 4 spanning trees actief, een voor elke vlan en een voor de default vlan 1 die je niet gebruikt

```bash
# overzicht van de spanning trees bekijken
show spanning-tree
# een switch een prioriteit hoiger in te stellen
spanning-tree vlan 1 root primary
```

### opracht

```bash
enable
configure terminal

spanning-tree vlan 10 root primary
spanning-tree vlan 20 root primary
spanning-tree vlan 33 root primary

exit

show spanning-tree
```

#### 2:

1. Root bridge: S0 si nu de root bridge voor vlan 10, vlan 20, vlan 33
2. actieve forwardigns poorten: poorten op S0 die verbinden zijn met s1,s2, s3 zullen in forwarding staan
3. geblokkeerde poorten: op s1, s2, s3 wordt 1 poort per redunante verbinding naar een andere switch in bloking mode geplaatst door stp

### Bekijk op S1, S2 en S3 welke poorten geblokkeerd zijn door het STP protocol. Beschrijf in je persoonlijke verslag hoe de tree er uitziet voor jouw drie VLANs

1. vlan 1: niet sure of dit allemaal klopt tho

```txt
S0>show spanning-tree vlan 20
VLAN0020
  Spanning tree enabled protocol ieee
  Root ID    Priority    24596
             Address     0050.0F5D.2C43
             This bridge is the root                                            toont de root
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec

  Bridge ID  Priority    24596  (priority 24576 sys-id-ext 20)
             Address     0050.0F5D.2C43
             Hello Time  2 sec  Max Age 20 sec  Forward Delay 15 sec
             Aging Time  20

Interface        Role Sts Cost      Prio.Nbr Type
---------------- ---- --- --------- -------- --------------------------------
Gi1/0/1          Desg FWD 19        128.1    P2p                                fwd is dat forwarding, dus niet geblockt
Gi1/0/4          Desg FWD 4         128.4    P2p
Gi1/0/3          Desg FWD 19        128.3    P2p
Gi1/0/2          Desg FWD 19        128.2    P2p

S0>
```
