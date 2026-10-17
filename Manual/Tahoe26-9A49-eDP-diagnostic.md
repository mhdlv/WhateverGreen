# Tahoe 26 — eDP DDI A / Pipe A, Intel 8086:9A49

## Périmètre et résultat

- Machine étudiée : Dell Latitude 3520, Core i5-1135G7, Iris Xe Tiger Lake PCI `8086:9A49`.
- Système étudié : macOS Tahoe 26 ; sortie interne eDP sur DDI A / Pipe A.
- Base : `tahoe-9a49-framebuffer-dev`, commit `c2f8d17ad156b0f667a0fc4e0e01284dbb860693`.
- Branche de travail : `tahoe-9a49-edp-diag-bounded`.
- Fichiers examinés : `WhateverGreen/kern_igfx.cpp` et `WhateverGreen/kern_igfx.hpp`.
- Aucune cause matérielle précise n'est démontrée sans capture du démarrage défaillant sur cette machine.
- Le changement améliore l'observation ; il ne prétend pas réparer l'écran noir.
- Aucun accès au Dell, à Windows, à l'EFI ou au SSD interne n'est effectué par ce travail.

## Analyse de la perte d'image

`TigerLakeRegisterMap` adapte les accès d'un framebuffer Ice Lake au moteur d'affichage Tiger Lake.
Le chemin eDP historique est remappé vers le transcoder A ; les registres de training DDI A sont
remappés de `0x64040/0x64044` vers `0x60540/0x60544`.
Les sélections DDI dans `TRANS_DDI_FUNC_CTL` et `TRANS_CLK_SEL` changent également d'encodage.
La cohérence entre training, clock, transcoder, buffer DDI, alimentation de dalle et PWM est donc centrale.

Le code de base décrit une écriture de zéro dans `TRANS_CLK_SEL_A` par `LightUpEDP`, après le training.
Sur le chemin Ice Lake eDP, cette sélection peut être inutile ; sur Tiger Lake, elle retire l'horloge
au transcoder A. Le correctif existant `SynthPreEnable` vise déjà cette différence.
Cette explication est un suspect étayé par la logique du code, pas une preuve du défaut actuel.
Il faut observer les valeurs réellement transmises et leur ordre autour de l'extinction.
Les écritures `TRANS_DDI_FUNC_CTL_A`, `TRANS_CONF_A` et `DDI_BUF_CTL_A` peuvent aussi expliquer
une absence de scanout alors qu'un rétroéclairage reste présent.
Les commentaires de tests antérieurs de la branche sont des indications historiques ; leurs journaux
bruts n'ont pas été fournis pour cette analyse et ne sont pas assimilés à une nouvelle mesure.

Le sous-module s'active avec un masque `igfxtglmap` non nul et un modèle CPU Tiger Lake `0x8C/0x8D`.
`-igfxtgldiag` seul ne l'active pas. Le code de base n'impose pas une restriction PCI `9A49`
ou Tahoe 26 : ce rapport étudie uniquement la machine et le système demandés.
Aucun changement de masque ni de garde matériel n'est introduit ici.

## Link training : ce que les valeurs démontrent

- `DP_TP_CTL_A`, bit 31 : commande d'activation du générateur de transmission.
- Ses bits 10:8 à `3` : sélection de transmission normale, pas résultat d'entraînement du récepteur.
- Une désactivation peut appartenir à un modeset ou à une extinction normale ; elle ne prouve pas un échec.
- `DP_TP_STATUS_A` est un état côté source ; il ne remplace pas les états AUX/DPCD du récepteur.
- La réussite Clock Recovery / Channel Equalization nécessite les états de lien DPCD par lane et l'alignement.
- L'absence de lecture `DP_TP_STATUS` dans le journal ne démontre ni réussite ni échec du lien.

Le diagnostic observe les lectures `DP_TP_STATUS` déjà réalisées par le pilote, sans nouvelle lecture MMIO
et sans transaction AUX ajoutée. Il enregistre le premier état observé et ses changements.
Il ne vérifie donc pas lui-même le training du panneau et ne peut pas exclure un échec côté récepteur.
La correction du commentaire « normal = trained » évite cette conclusion excessive.

## PWM : séparer signal vidéo et rétroéclairage

Les registres existants sont `BXT_BLC_PWM_CTL1` (`0xC8250`), période/fréquence PWM (`0xC8254`)
et duty (`0xC8258`). La période et le duty firmware sont déjà lus au premier accès MMIO.
Le diagnostic réutilise ces valeurs ; il n'ajoute pas de snapshot matériel.
Un duty nul transmis explique une commande d'extinction du rétroéclairage, mais pas nécessairement
la perte du flux vidéo ; un duty non nul ne prouve pas qu'une lumière est effectivement produite.
Il faut aussi considérer le bit d'activation PWM et l'alimentation de la dalle (`PP_CONTROL`, `0xC7204`).

`ScaleBacklight` conserve la période firmware, convertit l'échelle du pilote et laisse passer le blanking.
La restauration de duty sur réactivation existe déjà dans la branche de base ; elle est désormais signalée
séparément d'une écriture duty directe du pilote. `KeepBacklight` conserve également son comportement.
Les champs `requested` et `value` distinguent valeur du pilote et valeur transmise à l'accesseur d'origine.
`value` n'est pas une relecture après écriture ni une preuve que le matériel a accepté la programmation.
La correction du commentaire `ScaleBacklight` décrit son comportement existant sans le modifier.

## Lecture de la prochaine capture

| Observation corrélée à la perte d'image | Interprétation limitée aux accès observés |
| --- | --- |
| Stage 13 : `requested=0`, `value=0` alors que la pipe reste commandée active | Une suppression de la sélection d'horloge A a été transmise ; vérifier que le chemin de traduction attendu est effectivement actif. |
| Stage 13 : `requested=0`, `value=10000000` | Le garde existant contre la suppression de l'horloge a intercepté cette requête. Cela ne prouve pas la stabilité du reste du pipeline. |
| Stage 7 sans restauration 8/11 observée ensuite | Le duty a été mis à zéro ; une trace complète serait nécessaire pour attribuer l'extinction au PWM. |
| Stage 9 sans bit 31, ou stage 10 avec période nulle | Une commande de désactivation PWM ou de période nulle a été transmise ; vérifier la suite avant de conclure. |
| Stage 4 seul | Le pilote a sélectionné le pattern normal ; l'état CR/EQ du récepteur demeure inconnu. |

Le stage 5 décrit une commande d'activation de pipe, pas une confirmation du bit d'état actif
ou du défilement des frames. Le duty dépend de `driverDutyMax()` et de `igfxtglblmax` : une mauvaise
échelle peut expliquer une image très sombre. Une période firmware nulle empêche la conversion
habituelle et fait utiliser le repli existant ; elle est maintenant visible au stage 17.
Toute absence d'événement peut également provenir de la saturation ou de la perte de logs.

## Modification minimale du diagnostic opt-in

L'ancien diagnostic utilisait un ensemble mutable de champs partagés et une publication différée.
Une nouvelle transition pouvait écraser la précédente avant son impression ; le budget n'était pas atomique.
Le changement utilise 64 emplacements préalloués, immuables après publication, sans réutilisation.
Chaque producteur tente une seule réservation atomique : contention ou capacité atteinte perd l'événement.
Il n'existe ni boucle d'attente, ni sommeil, ni allocation par événement dans les hooks MMIO.
Une barrière atomique publie l'événement ; un worker différé consomme chaque emplacement au plus une fois.
Le worker balaie au plus 64 emplacements par invocation et imprime les événements publiés via `SYSLOG`.
L'observation reste best effort : saturation, contention, arrêt du système ou perte de logs peuvent tronquer la trace.

Le préfixe est `TGLDIAG`, avec séquence, `mach_absolute_time`, stage, adresse physique, demande et valeur.
Le temps est un compteur monotone brut, pas une date ni une durée directement exprimée en nanosecondes.
L'ordre de réservation donne un indice de séquence ; impression et chronologie peuvent différer entre CPU.
Le tampon n'est jamais réarmé ; les filtres de changements d'état restent best effort en concurrence.
Le message d'activation indique si le worker a pu être alloué ; sinon le diagnostic reste inactif.
Aucune écriture MMIO expérimentale, lecture MMIO supplémentaire, publication IORegistry/NVRAM
ou écriture de fichier n'est ajoutée par ce diagnostic.
Les mécanismes optionnels préexistants du sous-module restent inchangés et ne sont pas activés par ce travail.

| Stage | Observation |
| --- | --- |
| 1 | Duty firmware déjà lu au premier accès |
| 2 | `DP_TP_CTL_A` activé avec un pattern autre que normal |
| 3 | `DP_TP_CTL_A` désactivé |
| 4 | `DP_TP_CTL_A` activé, pattern normal sélectionné |
| 5 | `TRANS_CONF_A` : commande d'activation |
| 6 | `TRANS_CONF_A` : commande de désactivation |
| 7 | Duty nul transmis |
| 8 | Duty non nul transmis, y compris conservation firmware |
| 9 | Écriture contrôle PWM |
| 10 | Écriture période PWM effectivement transmise |
| 11 | Duty restauré par le comportement préexistant |
| 12 | Premier `DP_TP_STATUS_A` lu ou changement de sa valeur brute |
| 13 | Écriture sélection d'horloge transcoder A |
| 14 | Écriture fonction DDI du transcoder A |
| 15 | Écriture contrôle buffer DDI A |
| 16 | Écriture contrôle alimentation dalle |
| 17 | Période PWM firmware déjà lue au premier accès |
| 18 | Écriture synthétisée existante de sélection d'horloge A |
| 19 | Écriture synthétisée existante de fonction DDI A |

Les stages 13/14 décrivent le chemin du pilote ; les stages 18/19 distinguent `prepareTranscoderA`.
Pour les écritures synthétisées et les lectures observées, `requested` est égal à `value`.
Les événements 1/17 portent sur la première observation, pas sur une capture garantie avant tout pilote.
Le journal décrit les accès observés ; il n'enrichit pas la trace par des lectures actives de registres.

## Validation et limites

- Revue du diff : changements limités au diagnostic et à deux commentaires de comportement existant.
- `git diff --check` : réussi.
- Test hôte des fonctions extraites `noteDiagnostic` / `diagnosticThread` avec atomiques acquire/release
  simulées : mode désactivé, allocation absente, saturation, appels fusionnés, et 100 exécutions avec
  8 producteurs / 3 consommateurs ; au plus 64 événements cohérents et uniques, sans réutilisation.
- Rejeu comparatif base / patch : 17 cas, 10 701 accès MMIO simulés ordonnés et 2 005 retours de lecture
  identiques. Couvre blanking et restauration PWM, fréquence zéro, conservation firmware, échelle,
  training/préactivation, horloge zéro et chemins PLL. Snapshot initial, persistance et matériel exclus.
- CI du code : commit `1fb086c4fd78f308a238b11689db88626eb6f4f2`,
  [run 39073106150](https://github.com/mhdlv/WhateverGreen/actions/runs/39073106150).
- [Build 117514635393](https://github.com/mhdlv/WhateverGreen/actions/runs/39073106150/job/117514635393) :
  `xcodebuild -jobs 1 -configuration Debug` **success** ;
  `xcodebuild -jobs 1 -configuration Release` **success** ; dépôt de l'artefact `Artifacts` **success**.
- [Analyse Clang 117514635263](https://github.com/mhdlv/WhateverGreen/actions/runs/39073106150/job/117514635263) :
  configurations Debug et Release **success**. Coverity et publication Release **skipped**, conformément
  aux conditions du workflow sur ce fork et sur un push de branche.
- Le SHA et les conclusions de chaque étape ont été vérifiés sur les pages publiques des jobs GitHub Actions.
  Les journaux complets et les binaires de l'artefact n'ont pas été téléchargés par l'agent.
- Le workflow existant utilise `macos-latest`, MacKernelSDK et Lilu ; il reste inchangé.
- Aucun démarrage macOS Tahoe sur le Dell n'a été réalisé dans cet environnement cloud.
- Une compilation réussie valide la construction du code ; elle ne valide ni l'affichage, ni le training,
  ni le PWM, ni la stabilité matérielle, ni le sommeil/réveil.

## Collecte et vérifications matérielles restant à réaliser

Aucune installation ni modification de démarrage n'est prévue ici. `-igfxtgldiag` est uniquement
un paramètre disponible pour un éventuel essai externe distinct et autorisé ultérieurement.
Dans une session macOS existante déjà accessible, la lecture suivante peut extraire les messages disponibles :

```sh
sw_vers
uname -a
log show --last boot --style compact --predicate 'eventMessage CONTAINS "TGLDIAG"'
```

Ces commandes ne montent pas l'EFI, ne modifient pas le démarrage et n'écrivent pas de registres.
L'absence de messages dans le journal unifié ne prouve pas que les hooks n'ont pas été exécutés.
Conserver la version exacte de Tahoe, le SHA du kext et les arguments déjà en vigueur, sans les modifier.
Les observations encore nécessaires sont l'instant précis de disparition de l'image, la présence éventuelle
d'une image très faible sous éclairage externe, puis l'ordre des stages autour de cet instant.
Si les données existantes le permettent, comparer horloge/fonction/conf/port à la commande PWM et au duty.
Sans états DPCD existants du récepteur, conclure sur la réussite du training reste impossible.
Aucun nouveau registre, changement de masque, installation ou essai de sommeil/réveil n'est prescrit ici.

## Références de lecture

Les sources Linux i915 v6.6 servent à vérifier la sémantique ; elles ne prouvent pas l'état du Dell :

- [Registres et encodage DP_TP_CTL](https://github.com/torvalds/linux/blob/v6.6/drivers/gpu/drm/i915/i915_reg.h#L5613).
- [Préactivation Tiger Lake : horloge et fonction avant training](https://github.com/torvalds/linux/blob/v6.6/drivers/gpu/drm/i915/display/intel_ddi.c#L2578).
- [Séquence CR/EQ côté i915](https://github.com/torvalds/linux/blob/v6.6/drivers/gpu/drm/i915/display/intel_dp_link_training.c#L799).
- [Publication atomique OSCompareAndSwap](https://github.com/acidanthera/MacKernelSDK/blob/master/Headers/libkern/OSAtomic.h).
- [Contrat de remise en file des callbacks](https://github.com/apple-oss-distributions/xnu/blob/main/osfmk/kern/thread_call.c).
- [Définitions des états de lien DPCD](https://github.com/torvalds/linux/blob/v6.6/include/drm/display/drm_dp.h#L738).
