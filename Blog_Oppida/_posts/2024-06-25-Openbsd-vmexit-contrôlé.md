---
layout: post
title: "Openbsd vmexit contrôlé"
date: 2024-06-25
categories: Divers
background_image: /assets/maxresdefault.jpg
title_color: "#ffffff"
---




Cet article décrit comment générer un VM-exit contrôlé dans OpenBSD. Pour y parvenir, le code source du noyau doit être patché, notamment la partie qui gère les VM-exit. Un petit programme doit également être codé pour déclencher la raison du VM-exit ciblé.

### Contexte et Motivation

L'utilisation d'un fuzzer noyau actuel manque d'efficacité en raison de l'absence de couverture de code. Cette approche permettra d'implémenter un fuzzer guidé par couverture de code dans un environnement virtualisé, optimisant la récupération des traces d'exécution de chaque input transmis pour adapter les inputs mutés.

Le fuzzing guidé par couverture de code utilise l’instrumentation du programme pour retracer la couverture de code atteinte par chaque entrée transmise à une cible de fuzzing. Les moteurs de fuzzing utilisent ces informations pour muter les entrées afin de maximiser la couverture.

### VM-Exit et VM-Exit Contrôlé

Un VM-exit permet à l'hyperviseur de reprendre le contrôle de la VM lorsqu'un événement spécifique se produit, passant du mode VMX non-root au mode VMX root. Un VM-exit contrôlé est un VM-exit où l'événement déclencheur est choisi et sa gestion modifiée pour permettre l’exécution de code spécifique.

### Méthodologie

1. **Hyperviseur** : Acquérir une compréhension de base de l'architecture des hyperviseurs.
2. **VM-exit** : Comprendre la notion d’un VM-exit.
3. **Analyse** : Analyser les fichiers concernant la virtualisation pour identifier où est implémentée la gestion des raisons d’un VM-exit.
4. **POC** : Décrire l’étape de la POC (Proof of Concept) pour démontrer qu’un VM-exit contrôlé est possible dans OpenBSD.

### VMX

Les extensions de machine virtuelle (VMX) définissent la prise en charge au niveau du processeur pour les machines virtuelles basées sur les processeurs IA-32.

Deux principales classes de logiciel sont supportées :
1. **Virtual Machine Monitors (VMM)** : Un VMM contrôle entièrement le(s) processeur(s) ainsi que les autres matériels de la plateforme de l’hôte.
2. **Guest software** : Chaque machine virtuelle (VM) est un environnement logiciel invité qui fonctionne indépendamment des autres machines virtuelles.

### Opération VMX

La prise en charge de la virtualisation par le processeur est assurée par une forme d’opération du processeur appelée **VMX operation**.

Il existe deux types d'opérations VMX :
- **VMX root operation** : Un VMM s’exécute en mode VMX root.
- **VMX non-root operation** : Le logiciel invité s’exécute en mode VMX non-root.

Les transitions entre le fonctionnement en mode VMX root et VMX non-root sont appelées **transitions VMX** :
- **VM-entry** : Transition entre une opération en mode VMX root et une opération en mode VMX non-root.
- **VM-exits** : Transition entre une opération en mode VMX non-root et une opération en mode VMX root.

### Cycle VMM

Le schéma ci-dessous illustre le cycle d’un VMM et de son logiciel invité, ainsi que les interactions entre eux :
- VMM entre en fonctionnement VMX en exécutant une instruction VMX appelée **VMXON**.
- Utilisation de **VM-entry** pour passer de VMM vers le logiciel invité.
- Utilisation de **VM-exit** pour transférer le contrôle à un point d’entrée spécifié par VMM.

### Virtual Machine Control Structure (VMCS)

Les opérations et transitions VMX sont contrôlées par une **Virtual Machine Control Structure (VMCS)**.

La gestion de l’accès à la VMCS est assurée par un élément de l’état du processeur connu sous le nom de **VMCS Pointer**. Le VMM attribue une VMCS distincte à chaque VM qu’il gère.

### Direct Execution et Binary Translation

- **Direct Execution** : Méthode dans laquelle les instructions ou les demandes de l’utilisateur sont exécutées directement sur le matériel physique sans aucune modification.
- **Binary Translation** : Méthode utilisée pour exécuter des instructions ou des demandes du système d’exploitation invité qui ne sont pas supportées par le matériel physique.
- **Hypercall** : Demande spécifique faite par le système d’exploitation invité au VMM pour effectuer une opération nécessitant une intervention du VMM.

### Analyse du répertoire /sys d’OpenBSD

Les fichiers intéressants pour la gestion des raisons d’un VM-exit sont `vmmvar.h` et `vmm_machdep.c`.

Dans `/usr/src/sys/arch/amd64/include/vmmvar.h`, les constantes des raisons d’un VM-exit sont définies :

```c
/* VMX: Basic Exit Reasons */  
#define VMX_EXIT_NMI             0  
#define VMX_EXIT_EXTINT          1  
#define VMX_EXIT_TRIPLE_FAULT    2  
#define VMX_EXIT_INIT            3  
#define VMX_EXIT_SIPI            4  
...
#define VMX_EXIT_INVPCID         58  
#define VMX_EXIT_VMFUNC          59  
#define VMX_EXIT_RDSEED          61  
#define VMX_EXIT_XSAVES          63  
#define VMX_EXIT_XRSTORS         64
```

La fonction qui implémente la gestion des raisons d’un VM-exit est présente dans le fichier /usr/src/sys/arch/amd64/amd64/vmm_machdep.c, elle se nomme vmx_handle_exit() :

```c
int  
vmx_handle_exit(struct vcpu *vcpu)  
{  
    uint64_t exit_reason, rflags, istate;  
    int update_rip, ret = 0;  
  
    update_rip = 0;  
    exit_reason = vcpu->vc_gueststate.vg_exit_reason;  
    rflags = vcpu->vc_gueststate.vg_rflags;  
  
    switch (exit_reason) {  
    case VMX_EXIT_INT_WINDOW:  
       if (!(rflags & PSL_I)) {  
          DPRINTF("%s: impossible interrupt window exit "  
              "config\n", __func__);  
          ret = EINVAL;  
          break;  
       }  
  
       ret = EAGAIN;  
       update_rip = 0;  
       break;  
    case VMX_EXIT_EPT_VIOLATION:  
       ret = vmx_handle_np_fault(vcpu);  
       break;  
    case VMX_EXIT_CPUID:  
       ret = vmm_handle_cpuid(vcpu);  
       update_rip = 1;  
       break;  
    ...
    case VMX_EXIT_MWAIT:
    ...
```

POC
Création de la VM test dans OpenBSD

Créer un espace disque :

sh

# vmctl create -s <TAILLE>G <NOM_DISQUE>.img

Démarrer la VM à partir du fichier ISO de l’OS souhaité :

sh

# vmctl start -m <TAILLE_MEMOIRE>G -L -i 1 -r <CHEMIN_ISO> -d <DISQUE_CHEMIN> <NOM_VM>

Afficher l’état des VMs :

sh

# vmctl show

S’attacher à la console de la VM :

sh

# vmctl console <NOM_VM>

Arrêter la VM :

sh

# vmctl stop <NOM_VM>

Choix de la raison du VM-exit

La raison CPUID semble la plus appropriée à déclencher :

c

#include <sys/syslog.h>

    case VMX_EXIT_CPUID:
        ret = vmm_handle_cpuid(vcpu);
        update_rip = 1;
        log(LOG_INFO, "Its won in CPUID vm exit reason !!!!");
        break;

Reconstruire le noyau patché

Pour modifier le nom du noyau :

sh

# cd /usr/src/sys/arch/$(machine)/conf
# cp GENERIC.MP <NOM_NOYAU>.MP
# config <NOM_NOYAU>.MP

Pour reconstruire et installer :

sh

# cd ../compile/<NOM_NOYAU>.MP
# make obj
# make config
# make && make install

Coder un programme dans la VM test pour déclencher la raison du VM-exit CPUID

Voici un petit programme pour déclencher la raison du VM-exit CPUID :

c

#include <stdio.h>
#include <cpuid.h>

int
main()
{
    uint32_t eax, ebx, ecx, edx;
    printf("Setting up CPUID\n");
    __cpuid(0, eax, ebx, ecx, edx);
    printf("CPUID instruction successfully executed\n");
    
    return 0;
