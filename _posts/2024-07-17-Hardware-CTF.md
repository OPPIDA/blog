---
layout: post
author: [PLM]
title: "CTF - Hardware"
date: 2024-07-17
categories: CTF
background_image: assets/Tridroid-Banner.jpg
title_color: "#ffffff"
---

Objectifs :
- Apprentissage de la recherche documentaire (Datasheet)
- Découverte d’un maximum de protocoles (SPI,UART,I2C, …)
- Découverte d’outils (analyseur logique, oscilloscope)
- Manipulations (breadboard, branchement des pins, glitch)
- Sélection simple des exercices
- Rappeler qu’avec quelques Ko on peut faire des trucs funs

Résultats :
- 9 Flags
- Occupation 50% ROM et 90% de RAM
- Des heures de fun (ou pas ☺ )
- Multi platform -> ESP32-WROOM

Ce projet est réalisé par [samuel.marrazzo](https://twitter.com/EnlargeYourGeek)

Repo Github : [CTF-Hardware](https://github.com/smarrazzo/CTF-Hardware)

Plusieurs devices sont possible pour heberger ce challenge, mais pour cet article, les exerice ont été réalisé sur une ATmega32U4:  
![](assets/posts/2024-07-17-Hardware-CTF/1.png)

## Soudure  
![](assets/posts/2024-07-17-Hardware-CTF/2.png)

## Flash + Accès au menu

Flash :

```
python3 reset.py /dev/ttyACM0
```

```
avrdude "-Cavrdude.conf" -v -V -patmega32u4 -cavr109 "-P/dev/ttyACM1" -b57600 -D "-Uflash:w:CTF_Leonardo.hex:i"
```


L'accès au menu se fait par une connexion en serie sur le port USB de l'ATmega :

![](assets/posts/2024-07-17-Hardware-CTF/3.png)  
![](assets/posts/2024-07-17-Hardware-CTF/4.png)

## Installation sur une breadbord : 
Une breadbord est composée de trous espacés de 2.54mm permettant d’enfoncer des composants afin de les relier ensemble sans avoir à les souder, ce qui permet de faire des tests très facilement et très rapidement.

Les trous qui appartiennent à une même ligne, sont reliés électriquement.


![](assets/posts/2024-07-17-Hardware-CTF/6.png)

![](assets/posts/2024-07-17-Hardware-CTF/7.png)

## Exercices _________________________________________________________________________

## Exercice 1 :  Donne moi le flag !

En démarrant l'exercice 1, le flag est affiché à l'écran  
![](assets/posts/2024-07-17-Hardware-CTF/7.png)

```
USB_S3Ri@l_1s_T0_3@sY
```

## Exercice 2 :  Je crois que nous avons affaire à un serial killer!

![](assets/posts/2024-07-17-Hardware-CTF/8.png)

Cette phrase nous indique qu'il mapper les pins A0 sur **True** et A1 sur **False** :  
![](assets/posts/2024-07-17-Hardware-CTF/9.png)

![](assets/posts/2024-07-17-Hardware-CTF/10.png)

Le message : "Tu bluffes Martoni ! " apparait. 
Nous avons donc réussi à mapper les bon pins, mais le flag n'apparait pas.
L'énoncé nous parle de série, avec un Analyseur logique, il faut écouter TX et RX : 

![](assets/posts/2024-07-17-Hardware-CTF/11.png)

Lors de l'écoute, la capture permet de voir ceci : 

![](assets/posts/2024-07-17-Hardware-CTF/12.png)

Calcul du Baudrate avec l'extention "Baud Rate Estimate " 

![](assets/posts/2024-07-17-Hardware-CTF/13.png)

Ajustement de la configuration de l'analyse :

![](assets/posts/2024-07-17-Hardware-CTF/14.png)

Le flag apparait en ASSCI :  
![](assets/posts/2024-07-17-Hardware-CTF/15.png)

```
The flag is : S3ri@L_1S_FuN!\r\n
```

## Exercice 3 :  Mosi et Miso sont sur un bateau

![](assets/posts/2024-07-17-Hardware-CTF/16.png)

L'énoncé indique MOSI / MISO, ce qui nous fait penser au SPI : 
Le bus SPI utilise quatre signaux logiques :
- **SCLK** => Serial Clock, Horloge (généré par le maître)
- **MOSI**  => Master Output, Slave Input (généré par le maître)
- **MISO** => Master Input, Slave Output (généré par l'esclave)
- **SS** => Slave Select (optionel)

 Dans la datasheet de l'ATemaga32 nous pouvons identifier les PINS du SPI : 

![](assets/posts/2024-07-17-Hardware-CTF/17.png)


Avec l'analyseur logique : 

![](assets/posts/2024-07-17-Hardware-CTF/18.png)

il y a bien 3 flux : 
- Le chanel 2 en rouge, nous identifions bien la clock (un signal régulier)
- Le chanel 0 en orange : MOSI  
- Le chanel 1 en blanc : MISO 


![](assets/posts/2024-07-17-Hardware-CTF/19.png)
```
0x37 0x7A 0xBC 0xAF 0x27 0x1C
```

![](assets/posts/2024-07-17-Hardware-CTF/20.png)


si nous copions donc l'entiéreté des data du flux :  
![](assets/posts/2024-07-17-Hardware-CTF/21.png)


![](assets/posts/2024-07-17-Hardware-CTF/22.png)

Nous n'avons malheureusement pas encore le mot de passe pour ouvrir cette archive :( 
peut être dans un futur exercice... 
## Exercice 4/5 : Appel moi maître !
#### Fonctionnement de l'I2C : 
I2C : Inter-Integrated Circuit 

Les échanges ont toujours lieu entre un maître et un ou plusieurs esclave(s). 

SCL = clock 
SDA = la data 
![](assets/posts/2024-07-17-Hardware-CTF/23.png)


![](assets/posts/2024-07-17-Hardware-CTF/24.png)

![](assets/posts/2024-07-17-Hardware-CTF/25.png)

je suis longtemps resté bloqué sur ce chall jusqu'a ce que j'ai un hint du créateur du chall : 
![](assets/posts/2024-07-17-Hardware-CTF/26.png)

je dois donc m'aider d'un autre exercice pour résoudre celui ci, 
Si l'on regarde donc l'exo d'après :  
![](assets/posts/2024-07-17-Hardware-CTF/27.png)

nous avons bien à faire à un esclave, cela est donc cohérent.  
![](assets/posts/2024-07-17-Hardware-CTF/28.png)

Les pins de l'atmega32 correspondant à l'I2C sont donc les 2 et 3
2 pour la data et 3 pour la clock. 

Lorsque le maitre envoie ses commandes I2C : 
aucune réponse de slave ne lui est renvoyé donc 0x2A NAK :  
![](assets/posts/2024-07-17-Hardware-CTF/29.png)

mais par contre lorsqu'on connecte un autre atémega3 et qu'on le lance l'exo 5, on le passe donc en mode slave. 
L'échange est beaucoup plus important et l'on voit de la data passer :  

![](assets/posts/2024-07-17-Hardware-CTF/30.png)

- Le câble SDA Maitre sur le SDA Escale (Data)
- Le câble SCL Maitre sur le SCL Maitre (Clock)  
![](assets/posts/2024-07-17-Hardware-CTF/31.png)

Durant la capture : 

![](assets/posts/2024-07-17-Hardware-CTF/32.png)

Utilisation ensuite de python et unicode escape pour convertir notre data en GIF :   
![](assets/posts/2024-07-17-Hardware-CTF/33.png)

nous pouvons donc maintenant décoder notre archive 7z avec ce password : 

![](assets/posts/2024-07-17-Hardware-CTF/34.png)

```
SPI_1S_US3FU77
```

## Exercice 6 : Une histoire d'écoute 

![](assets/posts/2024-07-17-Hardware-CTF/35.png)

Branchement donc sur le pin 10 
![](assets/posts/2024-07-17-Hardware-CTF/36.png)


Puis écoute d'une communication à l'analyseur logique : 

Les "bits" de synchro du début nous montrent comment reconnaitre un long (-) et un court (.) :  
![](assets/posts/2024-07-17-Hardware-CTF/37.png)
Entièreté de la communication :   
![](assets/posts/2024-07-17-Hardware-CTF/38.png)

puis décodage : 
```
-.-. ----- ..- -.-. ----- ..- ..--.- - ..- ..--.- ...- ...-- ..- -..- ..--.- ...- ----- .. .-. ...-- ..--.- -- ----- -. ..--.- -- ----- .-. ..... ...-- ..--.- ..--..
```

Ce qui donne en décodant le morse : 
```
C0UC0U_TU_V3UX_V0IR3_M0N_M0R53_?
```

## Exercice 7 : Le sens de la vie 

Lorsque l'on lance l'exercice 7 : 
![](assets/posts/2024-07-17-Hardware-CTF/39.png)  

Après de nombreuse manipulation, je comprend que si l'on mappe certain pin entre eux, le binaire change et donc le résultat en décimal change : 
par exemple le CVV et la 4 :   
![](assets/posts/2024-07-17-Hardware-CTF/40.png)  
  
![](assets/posts/2024-07-17-Hardware-CTF/41.png)  
l'exercice 7 nous fais comprendre par sa phrase : "Connais tu la réponse ultime ?" 
qu'il attend le nombre 42 : Essayons donc de lui donner. 

Lorsqu'on mappe les pins : VCC + 3 
![](assets/posts/2024-07-17-Hardware-CTF/42.png)  
![](assets/posts/2024-07-17-Hardware-CTF/43.png)
````
The flag is :F33L_Th3_P0w3r_0f_B1n@rY
```
