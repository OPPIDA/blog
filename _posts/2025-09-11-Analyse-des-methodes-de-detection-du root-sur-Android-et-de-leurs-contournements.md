---
layout: post
author:
  - RDE
title: Analyse des méthodes de détection du root sur Android et de leurs contournements
date: 2025-09-11
categories:
  - Articles
  - Mobile
image: assets/posts/2025-09-11-Magisk-Root-Android/banniere.png
title_color: "#ffffff"
---

<script src="https://unpkg.com/mermaid@10/dist/mermaid.min.js"></script>
<script>
  mermaid.initialize({ startOnLoad: true });
</script>

# Analyse des méthodes de détection du root sur Android et de leurs contournements

## Introduction

Lors d'une récente évaluation mobile sur une application Android, nous avons été confrontés à plusieurs mécanismes détectant toute modification du système.  
De nombreux modules permettant de contourner ces protections sont facilement accessibles sur Internet, mais leur fonctionnement est rarement expliqué.
Le but de cet article est de décortiquer les mécanismes de détection de l'intégrité et de montrer comment ils peuvent être contournés.
## Magisk

### Introduction

Android étant un système fermé, il ne met pas l’user root à disposition de l'utilisateur. En conséquence, une grande partie du système est inaccessible. Pour pallier ce problème, la communauté a créé de nombreux utilitaires permettant de "rooter" un téléphone. Ces logiciels modifient Android afin de donner accès à l’user root. Le plus connu et le plus utilisé d’entre eux est Magisk.
L'accès root est utile dans de nombreux cas il permet de customiser le système en profondeur, et dans notre cas il est indispensable pour correctement évaluer une application.

### Fonctionnement

Avant de continuer, il est nécessaire de comprendre le fonctionnement de Magisk. Je vais expliquer brièvement son fonctionnement. Magisk est un root dit "systemless" c'est-à-dire qu'il ne va pas directement modifier le système Android. Son fonctionnement repose uniquement sur la modification de la partition de boot, où il remplace le processus d’initialisation standard par le sien.
Lors du démarrage du téléphone Magisk va créer une partition tmpfs, cette partition va contenir tous les fichiers nécessaires à son fonctionnement.
Magisk va par la suite créer un miroir du système classique dans un emplacement spécifique, cela va lui permettre de garder une copie système propre.
Magisk monte ensuite les fichiers nécessaires au root par-dessus ceux du système. L’OS et les applications voient les versions modifiées des fichiers, alors que la partition d’origine reste inchangée. 
Ce processus est appelé **Magic Mount**: il est possible grâce au bind mount qui permet à Linux de créer une vue alternative de fichier ou de dossier.

### Zygisk

Zygisk est un composant essentiel dans Magisk, c'est grâce à ce composant que les modules sont facilement intégrables à Magisk.
Pour comprendre comment Zygisk fonctionne, il est nécessaire de rappeler comment les applications Android fonctionnent. 
Toutes les applications Android fonctionnent dans des machines virtuelles, originellement appelées Dalvik. Il s'agit de VM java retravaillées afin qu'elles soient adaptées au système Android. Depuis la version 5 d'Android, Dalvik a été remplacée par Android Runtime plus couramment appelé ART. 
Dans un but d'optimisation, Android va créer une VM ART lors du démarrage pour chaque nouvelle application un fork de cette VM va être créé. 
C'est le processus Zygote qui va s'occuper de cette tâche, il va démarrer la VM mère et se mettre en écoute. Lors de la création d'un nouveau processus, il va s'occuper de réaliser le fork et de fournir la VM mère au processus.
![zygote](assets/posts/2025-09-11-Magisk-Root-Android/zygote-schema.png)
Comme son nom peut le laisser penser, Zygisk vient de Zygote + Magisk. Ce composant de Magisk va permettre d'automatiser l'injection de code directement dans Zygote. 
De plus Magisk, offre la possibilité aux utilisateurs d'installer des modules qui vont utiliser cette fonctionnalité.
Pour réaliser cela, Magisk va remplacer le binaire Zygote par son propre exécutable. Il va ensuite configurer des variables d'environnement et notamment **LDPRELOAD** afin d'exécuter une librairie "loader" dans le processus de Zygote. Une fois chargée, cette librairie expose un point d’entrée
permettant aux autres applications d’injecter du code dans Zygote. Ce loader étant chargé au tout début de l'exécution de Zygote, tous les forks vont hériter de ce code.   

### MagiskHide / Denylist

Magisk intégrait par défaut un utilitaire nommé MagiskHide, il permettait de cacher le root aux applications. Comme décrit précédemment, les modifications sont appliquées grâce à un système de mount. Lorsqu’il doit cacher le root à une application, MagiskHide démonte les
fichiers modifiés du point de vue de cette application, lui donnant ainsi l’illusion d’un système
Android classique. De plus la partition tmpfs citée précédemment qui contient les binaires et les configurations de Magisk n'est pas accessible depuis les applications.  
Cette fonctionnalité a été enlevée depuis la version 24 de Magisk, elle a été remplacée par Denylist. Bien que cachant le root, cet utilitaire fonctionne de manière beaucoup plus passive que MagiskHide. Lorsqu'une application est ajoutée dans la denylist Magisk, va se "désactiver". Cela signifie qu'il ne vient rien injecter dedans et va aussi refuser tous accès au binaire "su". Mais contrairement à MagiskHide, il ne va pas activement chercher à se cacher. 
Comme expliqué précédemment, le loader Zygisk est injecté dans la VM mère et est par conséquent hérité par toutes les VM filles. Après chaque fork, Zygisk vérifie si l’application figure dans la liste d’exclusion. Si c’est le cas, il interrompt le processus d’initialisation et nettoie l’environnement, de manière à ne pas s’exécuter dans les applications configurées dans cette liste.

### Magisk Alpha

Comme décrit précédemment, Magisk a une volonté de retirer les possibilités de cacher le root. Le développeur de Magisk a été recruté dans l'équipe de sécurité Android de Google en 2021. Il paraît donc logique qu'il ait abandonné toute volonté de cacher Magisk aux sécurités mises en place par son employeur. 
En contrepartie, la version 24 citée précédemment est aussi la version ou Zygisk a fait son apparition permettant ainsi à la communauté de reprendre le flambeau de MagiskHide.
Hormis Zygisk, des développeurs ont aussi créé des forks de Magisk le rendant plus discret? C'est notamment le cas de Magisk Alpha qui est un des forks les plus connus et utilisés.
Bien que closed-sources, quelques différences avec Magisk Canary(nom donné à Magisk post version 24) sont connues.
Alpha réintroduit plusieurs des fonctionnalités enlevées dans la version 24. Il réintègre notamment MagiskHide, mais au lieu d'être un module, il l’intègre directement dans son noyau, le rendant ainsi plus dur à contourner.
Il patche aussi différemment la partition boot. Alpha injecte plus de binaires que Canary. Il est également plus agressif sur plusieurs points. Par exemple, là où Canary ajoute simplement une règle à SELinux, Alpha injecte plusieurs règles qui lui permettent de se cacher plus facilement.


## Play integrity

Le Play Integrity est le remplaçant de SafetyNet. Il a pour but de détecter tout risque pour les applications présentes sur le téléphone. Il peut notamment identifier si une application a été modifiée, si le téléphone est émulé ou si des applications dangereuses sont installées sur l’appareil.
Dans notre cas, c'est la vérification de l'intégrité du téléphone qui va nous intéresser. N'importe quelle application peut demander au Play Integrity de scanner son téléphone afin de vérifier si le support est sécurisé.
![api-overview](assets/posts/2025-09-11-Magisk-Root-Android/play-integrity-api-overview.png)
Play Integrity évalue trois niveaux d’intégrité :
* **MEETS_BASIC_INTEGRITY** : le niveau le plus bas, qui vérifie uniquement si l’environnement est risqué (par exemple un émulateur ou un téléphone rooté).
* **MEETS_DEVICE_INTEGRITY** : vérifie que le système est correctement configuré avec Play Protect, que le bootloader est verrouillé, qu’aucune trace de root n’est présente et qu’il s’agit d’une ROM officielle.
* **MEETS_STRONG_INTEGRITY** : le niveau le plus élevé, qui vérifie en profondeur l’intégrité du téléphone. Il nécessite un appareil à jour et disposant des derniers correctifs de sécurité.

Pour le strong integrity, le téléphone doit posséder une sécurité matérielle telle que le **TEE** (_Trusted Execution Environement_) ou un **SecureElement**. Cette partie sécurisée du téléphone contient une paire de clés cryptographiques générées par les constructeurs de téléphone.
Lorsqu’une attestation est demandée, le **TEE** génère un certificat contenant plusieurs informations sur le téléphone : état du bootloader, niveau de correctif de sécurité et root of trust. Le root of trust est une structure de données qui décrit l’état de la chaîne de démarrage (bootloader déverrouillé, état du VerifiedBoot et empreinte de la clé racine). 
Ce certificat va être signé avec la clé privée des constructeurs et ensuite envoyé sur les serveurs de Google, qui va ensuite retourner le verdict.
Toute la sécurité du Play integrity repose sur cette paire de clés connues uniquement de Google et des constructeurs.

| Niveau                | Vérifie quoi ?                                      | Exemple de blocage         |
|-----------------------|------------------------------------------------------|----------------------------|
| Basic integrity       | Environnement à risque (émulateur, root détecté)    | Emulateur, device root     |
| Device integrity      | Bootloader verrouillé, ROM officielle, pas de traces | Téléphone root/modifie     |
| Strong integrity      | Attestation TEE + clé OEM                           | Toute modification du système |

## DexGuard

[DexGuard](https://www.guardsquare.com/dexguard) est un **RASP** (_Runtime Application Self-Protection_) développé par Guardsquare.
Il offre notamment une obfuscation avancée ainsi qu’un chiffrement des chaînes de caractères et des ressources. Il intègre également plusieurs mécanismes de sécurité tels que l’anti-tamper, l’anti-root et l’anti-debug.
L’application que nous avons évaluée, en plus d’utiliser Play Integrity, intégrait également DexGuard pour renforcer sa protection.
Étant une solution propriétaire, son fonctionnement exact n’est pas connu, mais elle est réputée robuste et difficile à contourner.
Lors de notre évaluation, même avec les trois niveaux d’intégrité validés, DexGuard posait encore problème et a nécessité des recherches approfondies pour être contournées.
## Contournement des sécurités

Maintenant que le fonctionnement de ces protections est compris, nous allons étudier comment elles sont contournées. La communauté Android étant très active, une multitude de modules permettant de les bypasser sont disponibles. Dans cette partie, nous verrons comment ils permettent de contourner la majorité des sécurités présentes sur Android.

1. Zygisk Next
[Zygisk Next](https://github.com/Dr-TSNG/ZygiskNext) est un module remplaçant le module Zygisk nativement intégré dans Magisk. Bien que parfaitement fonctionnel, Zygisk laisse des traces qui sont relativement faciles à détecter.
C'est en partie pour cela que Zygisk Next a été créée.
Ne pouvant pas s’injecter lors de la phase de boot comme Zygisk, Zygisk Next s’injecte dans le processus Zygote déjà en cours d’exécution.
N'étant pas open source on ne connaît pas la méthode dont il s'injecte.

2. Shamiko et Zygisk Assistant
Dans la communauté du root, deux modules reviennent principalement quand l'on souhaite cacher les traces du root. 
Le premier et sûrement le plus connu est [Shamiko](https://github.com/LSPosed/LSPosed.github.io/releases). Il est développé par la même équipe que le célèbre framework [LSPosed](https://github.com/LSPosed/LSPosed). 
Le second est [Zygisk Assistant](https://github.com/snake-4/Zygisk-Assistant), il est légèrement moins réputé, mais a l'avantage d'être totalement open source contrairement à Shamiko.
Ces deux modules vont essayer de cacher toutes traces de root et de la présence de Zygisk. Pour mettre en œuvre cette protection, ils utilisent une multitude de techniques que je vais rapidement détailler.
Ils reprennent le fonctionnement de Denylist, mais corrigent certaines de ses lacunes.
En s'injectant dans le processus de l'application, ils vont être capable d'intercepter les appels systèmes(open(), stat(), access(), etc) et modifier leur retour.
Ils modifient les logs et les contextes de SELinux pour effacer les traces de Magisk.
Ils nettoient également en profondeur les traces résiduelles des montages de Magisk. Par exemple, ils analysent le fichier _/proc/self/mounts_ afin de s’assurer qu’aucune trace de Magisk n’est présente.

3. Play integrity fork et Trickystore 
Ces deux modules vont servir à modifier le verdict du play integrity, comme cité précédemment le play integrity comporte trois niveaux d'intégrité, le basic, le device et le strong.

[Play integrity fork](https://github.com/osm0sis/PlayIntegrityFork) va agir uniquement sur les deux premiers niveaux.
Pour altérer le verdict Play integrity fork dit PIF va agir sur deux points.
Il va tout d'abord injecter du code afin de modifier le build.prop qui est un fichier contenant la configuration du système. Ici le module va se concentrer principalement sur les informations du système tel que Build.FINGERPRINT, Build.MODEL, Build.BRAND. Le but de cette modification est de faire passer le téléphone pour un téléphone non modifié, le plus souvent il va se faire passer pour un Google Pixel.
La seconde modification apportée est la modification des propriétés systèmes. Comme précédemment afin de leurrer le play integrity il va modifier les valeurs ro.boot.verifiedbootstate, ro.boot.flash.locked, ro.build.tags. Ces valeurs vont simuler que le bootloader est verrouillé et qu'aucune modification n’a eu lieu sur le système.

Le niveau d’intégrité le plus élevé repose sur une clé constructeur protégée par le matériel. À ce jour, aucun contournement fiable de ce mécanisme n'est connu publiquement. Le seul moyen de le valider est de reproduire le processus.
[Trickystore](https://github.com/5ec1cff/TrickyStore) va s'occuper de générer une attestation et de la signer, il va reproduire exactement le même procédé qu'Android va réaliser dans le TEE. Pour pouvoir réaliser cela, Trickystore doit posséder une clé valide. Heureusement pour nous des clés sont régulièrement leaks par les constructeurs permettant ainsi temporairement de pouvoir réaliser des attestations valides.

<pre class="mermaid">
sequenceDiagram
    participant App
    participant Trickystore
    participant Google

    App->>Trickystore: Demande d'attestation
    Trickystore->>Trickystore: Création d'un certificat avec une clé leakées
    Trickystore->>Google: Envoi du certificat
    Google-->>App: Verdict (validé or rejeté si la clé est bannie)
</pre>

Bien évidemment en contrepartie, Google va régulièrement bannir toutes les clés leaks rendant impossible la validation du strong_integrity jusqu’au prochain leak.
Trickystore ne fournit pas les clés leaks. Plusieurs moyens plus ou moins officiels permettent de récupérer des clés leaks, ces moyens ne seront bien évidement pas détaillés ici.

4. Contournement de Dexguard
Un dernier effort est nécessaire pour contourner DexGuard, le principal problème étant l’impossibilité d’identifier précisément ce qui provoquait la détection.
L'application [Native Detector](https://github.com/reveny/Android-Native-Root-Detector) va rechercher tous les problèmes dans l'environnement, il a été très utile pour bypasser Dexguard. 

<img src="assets/posts/2025-09-11-Magisk-Root-Android/Native-Detector.jpg" alt="Native-Detector" height="150" />


Deux soucis ont été remontés par l'application, le premier était qu'il arrivait à détecter une injection dans Zygote. Une simple mise à jour de Zygisk Next a permis de résoudre le souci. 
Le second est un problème dans le hash du boot, ce problème est aussi très simple à résoudre. Trickystore permet de modifier cet hash, il suffit de configurer le bon hash de boot pour résoudre ce souci.
Malgré les deux problèmes corrigés, Dexguard détectait toujours le root.
Après plusieurs tentatives infructueuses, la cause de la détection par DexGuard a enfin été identifiée. Il venait du manager de root de Magisk, malgré le fait que ce dernier ait été renommé et que son nom de package soit modifié, le **RASP** le détectait toujours. La simple suppression de ce manager a permis de résoudre le problème.
## Le futur

Conscient des problèmes liés aux fuites de clés, Google a déjà conçu une alternative qui rendrait Trickystore et les anciennes méthodes inopérantes.
Cette nouvelle approche, appelée **RKP** (_Remote Key Provisioning_), devrait être mise en œuvre et devenir obligatoire à partir du premier trimestre 2026.
Les clés constructeurs disparaissent et sont remplacées par une clé appelée **UDS** (_Unique Device Secret_). Cette clé est inscrite physiquement dans le processeur, ne peut pas être exportée et reste inaccessible, même pour le **TEE**, pourtant la zone la plus sécurisée de l’environnement Android. Cette clé physique va être utilisée pour dériver une paire de clés asymétriques éphémères qui seront stockées dans le **TEE** ou la **Strongbox** si le téléphone en a une à disposition. 
Cette paire est générée lors du premier démarrage du téléphone, mais le système peut décider à tout moment d’en dériver une nouvelle si nécessaire.
Pour attester de son intégrité, le téléphone génère une attestation appelée **DICE** (_Device Identifier Composition Engine_). Sans entrer dans les détails, il s’agit d’un mécanisme qui reflète l’état du système en s’appuyant à la fois sur des éléments matériels et logiciels, ce qui le rend extrêmement difficile, voire impossible, à reproduire.
Avec la clé publique et l’attestation **DICE**, le téléphone génère un **CSR** (_Certificate Signing Request_) qui les intègre.
Le **CSR** est envoyé aux serveurs de Google. S’il est jugé valide, il est signé par leur clé puis renvoyé au téléphone.
Ce certificat, valable quelques mois, est ensuite utilisé à chaque demande de vérification de l’intégrité du téléphone.

<p align="center">
<pre class="mermaid">
graph TD
    UDS[Clé unique dans le CPU] --> Ephemeral[Dérivation d'une paire de clés éphémères puis sauvegardées dans le TEE ou la StrongBox]
    Ephemeral --> DICE[Récupération de l'état du système via DICE]
    DICE --> CSR[CSR avec la clé publique et DICE]
    CSR --> Google[Validation sur les serveurs de Google]
    Google --> Cert[Renvoi du certificat signé]
    Cert --> Use[Le certificat atteste de l'état du système lors des demandes du play integrity]
</pre>
</p>
