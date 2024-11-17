# theoretische oef shit

## 1. is dit een correct host adres
### 102.101.99.98
- ja

### 253.254.255.256
- nee: 256 dakank

### 123.123.123.123 subnetmask : FF.E0.00.00
- FF.E0.00.00: 
``` txt

FF   -> 11111111
E0   -> 11100000
00   -> 00000000
00   -> 00000000

Binair notatie: 11111111.11100000.00000000.00000000             -> hier kan je dus van afleiden dat het / 13 is
                                                                -> hier kan je dus zeggen dat de host bits de laatste 13 bits zijn
- bereken het netwerkadres door een bitwise and uit te voeren tussen het ip acdres en het subnetmasker

subnetmask:         11111111.11100000.00000000.00000000
ipadres:            01111011.01111011.01111011.01111011
bitwise and tussen de 2: 
                    01111011.01100000.00000000.00000000
= netwerk adres: 123.96.0.0


- bereken het broadcast adres door de hostbits die geen deel uit maken van het netwerkadres in te vullen met enen
- host bits:      
ip adres:   01111011.01111011.01111011.01111011
                      _________________________
                      01111111.00011111.11111111

dus hier komt dit neer op : 01111011.01111111.11111111.11111111
en is het dus : broadcastadres: 123.127.255.255
```

dus we zien hier dat 123.123.123.123 tussen 123.96.0.0 en 123.127.255.255 ligt


### 14.143.143.143 subnetmask : FF.0F.00.00

``` txt
FF   -> 11111111
0F   -> 00001111
00   -> 00000000
00   -> 00000000

Binair: 11111111.00001111.00000000.00000000             --> in totaal 12 keer 1 : 32 - 12 : 20 hostbits | /12

- netwerk adres: 
subnetmask :        11111111.00001111.00000000.00000000
ip adres:           00001110.10001110.10001110.10001110
and operatie        0001110.00001110.00000000.00000000           aka = 0001110.00001110.00000000.00000000 
aka 00001110.00001111.11111111.11111111. = 14.15.255.255
14.15.0.0   <  14.143.143.143 < 14.15.255.255     ligt erbuiten dus false 



## 2. 