---
layout: post
author: [Nicolas RODRIGUES]
title: "Analyse-application-mobile-securisee-2"
date: 2025-10-03
categories: [Articles, Mobile, Sécurité]
image: assets/posts/rasp-ios/banner.png // image de couverture de votre choix à mettre à mettre dans le dossier assets 
title_color: "#ffffffff"
---

# Analyse d'une application mobile sécurisée

## [PART 2] - Développement de notre premier Tweak

Dans la première partie, nous avons été confrontés à un premier mécanisme de protection permettant la détection d'environnement de jailbreak. Cette protection nous empêchait d'exécuter l'application cible dans un environnement de test.  Nous avons donc étudié ce mécanisme et identifié une méthode pour contourner la détection et exécuter l'application sur un téléphone jailbreak. A présent, l'objectif est d'analyser le comportement de l'application afin d'évaluer sa sécurité.

### Première contrainte rencontrée

Lors de l'exécution de l'application sur certains de nos téléphones jailbreak, une erreur apparait. 

![image-20250513100846263](assets/posts/rasp-ios/1.png)

<div style="text-align: center;">Erreur affiché sur l'application</div>

Après quelques recherches, on remarque que l'erreur provient du fait que nos iPhones jailbreak ne possèdent pas de code de verrouillage. L'application ne s'exécute correctement que sur des téléphones possédant un code. En effet, enregistrer des données sensibles sur un appareil sans verrouillage ou de code d'accès le rend plus vulnérable au vol. Cependant, cela pose problème lorsque l'on souhaite utiliser le jailbreak "Palera1n" basé sur l'exploit **checkm8**. En effet, ce dernier ne fonctionne pas si le téléphone possède un code de verrouillage activé. Toutes les options liées au SEP (Secure Enclave Protocole) ne sont pas utilisable.  

Pour identifier si un téléphone possède un code de verrouillage, Apple dispose d'un framework nommé **[LocalAuthentication](https://developer.apple.com/documentation/localauthentication)**. 

![image-20250513104745066](assets/posts/rasp-ios/2.png)

<div style="text-align: center;">Vérification de la présence d'un code par l'application</div>

Lors de l’analyse de l’application, un appel à la méthode [`canEvaluatePolicy(_:error:)`](https://developer.apple.com/documentation/localauthentication/lacontext/canevaluatepolicy(_:error:)) du framework *LocalAuthentication* est effectué afin de vérifier la présence d’un code de déverrouillage configuré sur l’appareil. Dans notre cas, cette méthode retourne la valeur **false**, indiquant l’absence d’un tel code. En conséquence, l’application interrompt son exécution et affiche l’erreur mentionnée précédemment.

Pour poursuivre nos tests, il est nécessaire d'exécuter cette application sur notre téléphone. Notre première approche a été d'utiliser Frida. En effet, en interceptant l'appel à la méthode canEvaluatePolicy(_:error:), nous pouvons renvoyer la valeur **true** pour simuler la présence d'un code de verrouillage sur le téléphone. Voici le script Frida que nous avons écrit à cet effet :

```js
if (ObjC.available) {
    var LAContext = ObjC.classes.LAContext;
    var canEvaluatePolicy = LAContext["- canEvaluatePolicy:error:"];

    Interceptor.attach(canEvaluatePolicy.implementation, {
        onEnter: function (args) {
            var policy = args[2].toInt32();
            console.log("[+] App called canEvaluatePolicy with policy: " + policy);
        },
        onLeave: function (retval) {
            console.log("[+] Forcing return value to TRUE (passcode/biometry active)");
            retval.replace(1); // 1 = true
        }
    });

    console.log("[+] Hook applied, waiting for the app to start...");

} else {
    console.log("[-] Objective-C Runtime is not available.");
}

```

Cependant, lors de son exécution, **Frida** crash avec le message d'erreur suivant.

![attach-appli-integ](assets/posts/rasp-ios/3.png)

<div style="text-align: center;">Message d'erreur affiché par Frida</div>

Nous avons compris que l'application est équipée d'un mécanisme de protection anti-hook efficace, ce qui nous empêche d'instrumenter l'application pour y injecter notre code. Après plusieurs tentatives infructueuses pour dissimuler notre action de hooking, notamment en utilisant un patch de Frida et diverses autres modifications, nous avons décidé d'explorer une solution alternative.

### Les Tweaks

Afin d'effectuer des hooks du système sans utiliser **Frida** tout en rendant l'instrumentation difficiles à détecter, nous avons exploré l'utilisation des **tweaks** sur iOS

Un **tweak** est une modification logicielle installée sur un appareil iOS jailbreaké. Il permet de changer le comportement du système, d'ajouter des fonctionnalités ou de modifier l'apparence de l'interface. Ces tweaks reposent sur des systèmes d'injection dynamique tels que **MobileSubstrate**, **ElleKit**, etc.

#### Fonctionnement de l'injection dynamique avec ElleKit

**ElleKit** est un loader moderne optimisé pour les versions récentes d'iOS. Il offre une injection fiable malgré les protections modernes, telle que [PAC](https://support.apple.com/en-mz/guide/security/sec8b776536b/1/web/1#sec0167b469d), et est principalement utilisé dans les jailbreaks rootless. 

Au démarrage du système, un daemon d'injection fourni par **ElleKit** est activé. Son rôle est alors de surveiller et d'injecter les tweaks dans les processus cibles. 

Lorsqu'un tweak est installé, il est généralement compilé sous forme de bibliothèque dynamique (`.dylib`). Il dispose également d'un fichier `.plist`  qui précise les cibles sur lesquelles il s'injecte, comme SpringBoard, UIKits, ou d'autres applications.  **ElleKit** injecte alors cette bibliothèque dans les processus cibles en se référant au fichier `.plist`.

![image-20250513172909780](assets/posts/rasp-ios/4.png)

<div style="text-align: center;">Fonctionnement de l'injection de tweak</div>

Par défaut, lorsqu'un processus est lancé, iOS utilise **dyld** (Dynamic Linker) pour charger les bibliothèques dynamiques dans l'espace du processus. **ElleKit** intervient à bas niveau en s'intégrant  au processus de chargement **dyld**. Il injecte sa bibliothèque `libellekit.dylib` dans la variable d'environnement [DYLD_INSERT_LIBRARIES](https://man.cx/dyld(1)), ce qui lui permet de surveiller et d'intercepter le lancements de chaque processus. Il représente l'équivalent de `LD_PRELOAD` sur Linux pour iOS.

L'utilisation de DYLD_INSERT_LIBRARIES est d'ailleurs une [méthode bien connue](https://theevilbit.github.io/posts/dyld_insert_libraries_dylib_injection_in_macos_osx_deep_dive/) pour injecter du code dans un processus.

Une fois le processus ciblé lancé via **launchd**, **ElleKit** injecte ses bibliothèques dynamiques (tweaks) dans l'espace mémoire du processus ciblé. Il modifie ensuite la table des symboles pour rediriger les appels des fonctions et méthodes ciblées vers les implémentations des tweaks. 

### Développement de notre tweak

Nous avons alors décidé de développer notre propre tweak afin d'intercepter et hooker l'appel de la fonction `canEvaluatePolicy(_:error:)` effectué par l'application. Pour ce faire,  le framework **[Theos](https://theos.dev/)** répond parfaitement à notre problématique.

**Theos** est un environnement de développement open-source utilisé principalement pour créer des **tweaks** (modifications système) sur les appareils iOS jailbreakés. Il permet de compiler, structurer et déployer du code Objective-C / Swift en `.dylib`, injectables dans le système ou dans des apps via des frameworks comme ElleKit ou libhooker. Il fonctionne autant sur macOS que sur Linux. 

![image-20250513175759300](assets/posts/rasp-ios/5.png)

<div style="text-align: center;">Choix divers de création de programme iOS via Theos</div>

De plus, il propose aussi son propre langage, le **Logos**, utilisé pour réaliser des hooks. 

Nous ne détaillerons pas comment installer **Theos** sur un ordinateur car il existe déjà un bon nombre d'articles sur ce sujet :

* https://theos.dev/docs/installation-linux
* https://medium.com/@bancarel.paul/jailbreak-create-your-first-ios-tweak-version-67e8159c53f5

La fonction `canEvaluatePolicy:` est définie dans la bibliothèque [LAContext.h](https://github.com/hughbe/macOS-iOS-headers/blob/master/macOS/Frameworks/LocalAuthentication.framework/LAContext.h#L150). Il est ainsi possible d'en extraire la structure.

**Theos** offre un format simple pour effectuer des actions. Le définisseur `%hook` permet d'indiquer la classe que l'on souhaite intercepter et personnaliser.  Il nous suffit donc de hooker `LAContext` afin de modifier le retour de la fonction qui nous intéresse.

Ainsi, le tweak suivant est crée : 

```objective-c
#import <UIKit/UIKit.h>
#import <LocalAuthentication/LAContext.h>

%hook LAContext

- (BOOL)canEvaluatePolicy:(LAPolicy)policy error:(NSError **)error {
    return YES; //Indique que le code de dévrouillage est présent
}

%end
```

<div style="text-align: center;">Tweak permettant de hook la valeur du code de déverrouillage</div>

Lors de l'interception de l'utilisation de `LAContext`, on force la méthode `canEvaluatePolicy:error:` à retourner la valeur "**YES**", ce qui fera pensé que le système possède un mécanisme d'authentification (FaceID ou Touch ID).  Ainsi, il contourne les vérifications réelles de disponibilité.

Pour que le tweak puisse s'exécuter correctement dans nos environnements, nous spécifions dans le Makefile les différentes caractéristiques liées à notre configuration, notamment qu'il s'agit d'un tweak rootless.

Enfin, pour cet exemple, nous avons décidé d'appliquer ce tweak à toutes les applications. C'est pourquoi nous avons choisi de cibler [UIKit](http://en.wikipedia.org/wiki/UIKit) dans notre fichier `.plist`. En effet il représente un framework de développement utilisés par presque toutes les applications. En le ciblant, notre tweak s'applique à pratiquement toutes les applications d'un coup.

```xml
<dict>
    <key>Filter</key>
    <dict>
        <key>Bundles</key>
        <array>
            <string>com.apple.UIKit</string>
        </array>
    </dict>
</dict>
```

<div style="text-align: center;">Fichier .plist du tweak</div>

Une fois compilé, le tweak peut être installé sur notre téléphone jailbreak. 

![tweak](assets/posts/rasp-ios/6.png)

<div style="text-align: center;">Tweak installé dans Sileo</div>

Ainsi, on peut constater que notre application se lance correctement depuis un téléphone jailbreaké avec **Palera1n** dépourvu de mécanisme de verrouillage. L'application n'a pas identifiée l'utilisation de notre tweak. 

### Conclusions 

Dans cette section, nous avons exploré le fonctionnement et le développement de tweaks permettant d’effectuer du hooking sur des fonctions d’une application cible. Cette approche constitue une alternative pertinente pour instrumenter des fonctions, en raison de sa furtivité et de son efficacité. En effet, nous avons constaté qu’elle permet de contourner certains mécanismes de détection mis en place pour contrer l’analyse dynamique, contrairement à d’autres techniques plus classiques.

L’intégralité du tweak développé dans le cadre de cet article est disponible sur le [GitHub d’Oppida](https://github.com/oppida), afin de faciliter sa reproduction et son adaptation.


Dans la prochaine partie, nous verrons comment approfondir encore davantage l’analyse de notre application cible en allant au-delà du simple hooking.


