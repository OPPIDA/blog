---
layout: post
author: [PLM]
title: "CTF - Hardware"
date: 2024-07-17
categories: CTF
background_image: assets/Tridroid-Banner.jpg
title_color: "#ffffff"
---

![](assets/posts/2024-07-17-Hardware-CTF/1_01.png)

Ce projet est réalisé par samuel.marrazzo (https://twitter.com/EnlargeYourGeek)


Repo Github : [CTF-Hardware](https://github.com/smarrazzo/CTF-Hardware)


Plusieurs devices sont possible pour heberger ce challenge, mais j'ai choisi une ATmega32U4
![](assets/posts/2024-07-17-Hardware-CTF/1.png)

## 1 ere étape - Soudure
![](assets/posts/2024-07-17-Hardware-CTF/2.png)

## 2 eme Etape - Flash + accès au menu

Flash :

```
python3 reset.py /dev/ttyACM0
```

```
avrdude "-Cavrdude.conf" -v -V -patmega32u4 -cavr109 "-P/dev/ttyACM1" -b57600 -D "-Uflash:w:CTF_Leonardo.hex:i"
```


L'accès au menu se fait par une connexion en serie sur le port USB de l'ATmega :

![[assets/posts/2024-07-17-Hardware-CTF/3.png]]
![[assets/posts/2024-07-17-Hardware-CTF/4.png]]

## Installation sur une breadbord : 
Une breadbord est simple a utiliser, tout les pins sont mappées... 

![[6.png]]

![[5.png]]

## Exercices 

### Exercice 1 :  Donne moi le flag !

En démarrant l'exercice 1, le flag est affiché à l'écran 
![](7.png)

```
USB_S3Ri@l_1s_T0_3@sY
```

### Exercice 2 :  Je crois que nous avons affaire à un serial killer!

![](8.png)

Cette phrase nous indique qu'il mapper les pins A0 sur **True** et A1 sur **False** :  
![](9.png)

![](10.png)

Le message : "Tu bluffes Martoni ! " apparait. 
Nous avons donc réussi à mapper les bon pins, mais le flag n'apparait pas.
L'énoncé nous parle de série, avec un Analyseur logique, il faut écouter TX et RX : 

![](11.png)

Lors de l'écoute, la capture permet de voir ceci : 

![](12.png)

Calcul du Baudrate avec l'extention "Baud Rate Estimate " 

![](13.png)

Ajustement de la configuration de l'analyse :

![](14.png)

Le flag apparait en ASSCI : 
![](15.png)

```
The flag is : S3ri@L_1S_FuN!\r\n
```

### Exercice 3 :  Mosi et Miso sont sur un bateau

![[Pasted image 20240530095416.png]]

L'énoncé indique MOSI / MISO, ce qui nous fait penser au SPI : 
Le bus SPI utilise quatre signaux logiques :
- **SCLK** => Serial Clock, Horloge (généré par le maître)
- **MOSI**  => Master Output, Slave Input (généré par le maître)
- **MISO** => Master Input, Slave Output (généré par l'esclave)
- **SS** => Slave Select (optionel)

 Dans la datasheet de l'ATemaga32 nous pouvons identifier les PINS du SPI : 

![[Pasted image 20240530101512.png]]


Avec l'analyseur logique : 

![[Pasted image 20240530105045.png]]

il y a bien 3 flux : 
- Le chanel 2 en rouge, nous identifions bien la clock (un signal régulier)
- Le chanel 0 en orange : MOSI  
- Le chanel 1 en blanc : MISO 


![[Pasted image 20240530105320.png]]

```
0x37 0x7A 0xBC 0xAF 0x27 0x1C
```

![[Pasted image 20240530105438.png]]


si nous copions donc l'entiéreté des data du flux : 
![[Pasted image 20240530105516.png]]


![[Pasted image 20240530105532.png]]

Nous n'avons malheureusement pas encore le mot de passe pour ouvrir cette archive :( 
peut être dans un futur exercice... 
### Exercice 4 : Appel moi maître !
#### Fonctionnement de l'I2C : 
I2C : Inter-Integrated Circuit 

Les échanges ont toujours lieu entre un maître et un ou plusieurs esclave(s). 

SCL = clock 
SDA = la data 
![[Pasted image 20240603110706.png]]


![[Pasted image 20240603104039.png]]

![[Pasted image 20240603104058.png]]

je suis longtemps resté bloqué sur ce chall jusqu'a ce que j'ai un hint du créateur du chall : 
![[Pasted image 20240603111147.png]]

je dois donc m'aider d'un autre exercice pour résoudre celui ci, 
Si l'on regarde donc l'exo d'après : 
![[Pasted image 20240603111239.png]]

nous avons bien à faire à un esclave, cela est donc cohérent.
![[Pasted image 20240603111412.png]]

Les pins de l'atmega32 correspondant à l'I2C sont donc les 2 et 3
2 pour la data et 3 pour la clock. 

Lorsque le maitre envoie ses commandes I2C : 
aucune réponse de slave ne lui est renvoyé donc 0x2A NAK :
![[Pasted image 20240603113022.png]]

mais par contre lorsqu'on connecte un autre atémega3 et qu'on le lance l'exo 5, on le passe donc en mode slave. 
L'échange est beaucoup plus important et l'on voit de la data passer : 

![[Pasted image 20240603131654.png]]

- Le câble SDA Maitre sur le SDA Escale (Data)
- Le câble SCL Maitre sur le SCL Maitre (Clock)
![[Pasted image 20240603132055.png]]

Durant la capture : 

![[Pasted image 20240603140133.png]]

Utilisation ensuite de python et unicode escape pour convertir notre data en GIF : 
![[Pasted image 20240603141736.png]]

nous pouvons donc maintenant décoder notre archive 7z avec ce password : 

![[Pasted image 20240603141926.png]]

```
SPI_1S_US3FU77
```

### Exercice 6 : Une histoire d'écoute 

![[Pasted image 20240604105937.png]]

Branchement donc sur le pin 10 
![[Pasted image 20240604110052.png]]


Puis écoute d'une communication à l'analyseur logique : 

Les "bits" de synchro du début nous montrent comment reconnaitre un long (-) et un court (.) :
![[Pasted image 20240604110202.png]]
Entièreté de la communication : 
![[Pasted image 20240604105831.png]]

puis décodage : 
```
-.-. ----- ..- -.-. ----- ..- ..--.- - ..- ..--.- ...- ...-- ..- -..- ..--.- ...- ----- .. .-. ...-- ..--.- -- ----- -. ..--.- -- ----- .-. ..... ...-- ..--.- ..--..
```

Ce qui donne en décodant le morse : 
```
C0UC0U_TU_V3UX_V0IR3_M0N_M0R53_?
```

### Exercice 7 : Le sens de la vie 

Lorsque l'on lance l'exercice 7 : 
![[Pasted image 20240716161514.png]]

Après de nombreuse manipulation, je comprend que si l'on mappe certain pin entre eux, le binaire change et donc le résultat en décimal change : 
par exemple le CVV et la 4 : 
![[Pasted image 20240716162534.png]]

![[Pasted image 20240716162208.png]]
l'exercice 7 nous fais comprendre par sa phrase : "Connais tu la réponse ultime ?" 
qu'il attend le nombre 42 : Essayons donc de lui donner. 

Lorsqu'on mappe les pins : VCC + 3 
![[Pasted image 20240716163327.png]]
![[Pasted image 20240716163003.png]]

````
The flag is :F33L_Th3_P0w3r_0f_B1n@rY
```
