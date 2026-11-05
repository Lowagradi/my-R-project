# EW-01 — Guerre électronique (développement isolé pour TJAZ)

> **Dépôt de développement :** [Lowagradi/my-R-project](https://github.com/Lowagradi/my-R-project)  
> **Branche de mission :** `mission/ew-01-isolated`  
> **Destination future, PAS destination de cette mission :** [Lowagradi/TJAZ](https://github.com/Lowagradi/TJAZ)  
> **Référence du cahier de mission original :** [TJAZ — EW-01, PR #29](https://github.com/Lowagradi/TJAZ/pull/29) · [README original](https://github.com/Lowagradi/TJAZ/blob/docs/ew-01-mission/docs/missions/ew-01/README.md)  
> **Dépendances techniques dans TJAZ (lecture uniquement) :** [OBS-01 PR #26](https://github.com/Lowagradi/TJAZ/pull/26), [SIGINT-02 PR #27](https://github.com/Lowagradi/TJAZ/pull/27), [J6-SEC-01 PR #28](https://github.com/Lowagradi/TJAZ/pull/28).  
> **Statut :** cahier d'implémentation et tests dans un dépôt isolé ; **pas une intégration à TJAZ**.

## 0 — Règles de travail essentielles

**L'utilisateur veut deux phases clairement séparées.**

1. **Phase A, maintenant :** concevoir, coder et tester EW-01 **uniquement dans `my-R-project`**, sur une branche dédiée et dans une PR de ce dépôt. Ce repo était initialement vide : les modules TJAZ, ses classes Godot, ses tests et ses sauvegardes **n'y sont pas présents**. Ne prétends pas qu'ils existent. Construis un **prototype indépendant Godot 4.7.2**, réutilisable, avec de petites interfaces/adaptateurs et un environnement de test minimal. Référence le code TJAZ public **en lecture seulement** pour ses contrats ; n'y effectue aucune écriture.
2. **Phase B, ultérieure :** le propriétaire demandera à ChatGPT de **réviser et vérifier** le code et les tests terminés. **Ne copie, ne synchronise, ne fusionne et n'ouvre aucune PR dans TJAZ** avant cette revue et son autorisation explicite. La réussite des tests isolés n'équivaut jamais à une intégration réussie dans le vrai simulateur.

**Interdictions :** aucune modification du dépôt `Lowagradi/TJAZ`, aucune fusion dans `main` dans l'un ou l'autre dépôt, aucun APK, aucun export Android, aucun déploiement, aucune modification de workflows GitHub, aucun accès à des secrets ni à des services payants. N'exécute pas `git reset --hard`, `git clean` ou force-push pour effacer le travail existant.

**Dépendance en attente dans TJAZ :** le correctif J6-SEC-01.1 d'autorité de commande et de livraison effective des rapports est demandé dans [l'audit PR #28](https://github.com/Lowagradi/TJAZ/pull/28#issuecomment-6657033615). Il **bloque l'intégration future à TJAZ**, mais **pas le développement du prototype EW autonome ici**, puisque celui-ci ne modifie pas J6. Documenter les hypothèses d'intégration et ne pas simuler J6 comme déjà corrigé.

## 1 — Vision gameplay

Dans TJAZ, le joueur et les États IA doivent pouvoir **surveiller passivement**, **émettre**, **brouiller**, **cesser l'émission** et réagir aux perturbations.

Un brouilleur actif peut rendre un réseau ou un capteur moins efficace, mais son émission peut aussi être détectée. **Brouiller n'implique pas devenir invisible ou connaître la source adverse.** Les effets dépendent du **phénomène physique abstrait**, jamais de l'identité diplomatique de l'émetteur ou du destinataire.

La chaîne voulue reste :
```text
Plateforme + équipement EW opérationnel
            ↓ activation (JAM / LISTEN / OFF)
Phénomène EW réel, spatial, temporel, observable
            ├→ perturbation des liaisons RF et capteurs compatibles
            └→ éventuelle détection SIGINT (contact anonyme)
                                      ↓ mémoire tactique locale
                               rapport via communications
                                      ↓ réception effective
                                   J2 / J6
```
Une unité peut observer localement sans que son commandement connaisse immédiatement l'information ; un brouillage peut interrompre la remontée. Le comportement est identique pour joueur/IA, unités alliées/adverses et neutres, à équipement et milieu égaux.

## 2 — Architecture : prototype compatible sans fausse intégration

Le vrai TJAZ a déjà `EquipmentDef` de catégorie **D/SUBSYSTEM**, `EquipFitCap`, `EquipCompatibility`, `CommsDevice`, `CommsNetwork`, `LinkAssessor`, `SensorDevice`, `ObservationEngine`, `LocalTacticalKnowledge`, `ActorKnowledge`, `CommsSecurity` et `SimulationEngine`. **Ne recrée pas des moteurs concurrents dans TJAZ.**

Dans **ce dépôt isolé**, comme ces classes sont absentes, créer uniquement le **minimum de modèles stubs/adaptateurs** nécessaires pour exécuter des tests reproductibles. Marquer clairement tout stub `TEST_ONLY`. Séparer :
- **Noyau EW portable** : état/capacités des brouilleurs, émission d'événements EW, calcul de perturbation, activation/arrêt, contre-mesures, sérialisation déterministe.
- **Interfaces d'intégration attendues** : demande abstraite de qualité de liaison/capteur ; retour d'un effet réversible sans mutation permanente ; événement RF observable ; contexte d'identité et de propriétaire réservé à l'autorisation des commandes.
- **Fixtures** : faux navires, radios, capteurs, routes et acteurs, strictement destinés aux tests. Les tests doivent décrire **ce qu'ils prouvent et ce qu'ils ne prouvent pas**.

**Ne pas copier tout le projet TJAZ** dans ce repo. Si le portage nécessite un contrat supplémentaire, documenter précisément la dépendance par chemin de fichier et fonction, et proposer un adaptateur lors de la revue future.

## 3 — Périmètre EW-01

### A. Équipements et activation

Un équipement de guerre électronique est un **sous-système installable** sur une plateforme (navire, avion, base terrestre). Capacités déclarées :
- bandes/canaux pris en charge ;
- portée, puissance et qualité **abstraites** ;
- directionnalité optionnelle ;
- état, intégrité, énergie disponible, panne, EMCON ;
- modes `OFF`, `LISTEN`, `JAM` ;
- date d'activation, restrictions et éventuelles conséquences opérationnelles légères.

Une plateforme ou un matériel incapable d'émettre ne doit jamais produire de perturbation. L'écoute passive est distincte de l'émission active. Seules les plateformes autorisées peuvent changer leurs équipements. Pas de catalogue industriel complet ni de deuxième économie.

### B. Effets physiques abstraits et réversibles

Calculer à partir de sources EW **effectivement actives** :
- distance et bande compatible ;
- puissance, qualité, état, énergie ;
- environnement et direction simplifiée ;
- modificateur de qualité de réception RF ;
- baisse éventuelle de portée utile, augmentation de perte/latence, ou échec d'acquisition du capteur.

**Ne pas** coder de pénalité permanente dans le matériel victime, ni un malus global « pays ennemi ». L'effet disparaît quand la source cesse, tombe en panne ou sort de portée. Définir les règles de combinaison de plusieurs brouilleurs (bornes, plafonds, ordre stable). Une station peut affecter ses propres radios ou celles de ses alliés si le milieu le justifie.

Pour l'intégration future, distinguer disponibilité de canal/transport de l'authenticité et du secret du message : **EW perturbe COMMS/OBS, mais ne révoque pas les clés J6 et ne décrypte rien**.

### C. Signature observable du brouilleur

Lorsqu'un brouilleur émet réellement, produire un **événement RF observable** : bande, activité, classe, intensité abstraite, temps, empreinte physique privée de l'émetteur. Un récepteur SIGINT compatible peut détecter cette émission avec les mêmes limites de portée et de qualité que les autres signaux.

Le résultat public ne reçoit **jamais** d'identifiant réel d'émetteur, de position de vérité, de clé ni de contenu de message. Une localisation est possible seulement via une mesure directionnelle puis recoupement imparfait ; sinon `position_available=false`.

L'arrêt/EMCON n'efface pas les renseignements déjà observés et transmis, mais arrête les **nouvelles émissions**.

### D. Contre-mesures

Démontrer au minimum :
- changement de bande/canal lorsque le vrai équipement le permet ;
- matériel de secours compatible ;
- arrêt/réduction d'émission ou retrait de la zone ;
- résistance/filtrage explicitement décrits.

Aucune contre-mesure parfaite. Documenter les sacrifices (portée, délai, énergie, détectabilité, capacité).

### E. Commandement et renseignements

Une perte de communications crée incertitude et délai, pas un faux accusé de réception. Une observation EW est d'abord **locale**. Seul un rapport livré peut rejoindre les ports J2/J6. Pour le prototype isolé, construire un **transport factice à états** `PENDING / IN_TRANSIT / DELIVERED / FAILED` et une mémoire observateur/commandement de démonstration, **sans prétendre valider le moteur COMMS/J6 de TJAZ**.

Les identités et propriétaires ne jouent que pour l'autorisation des actions, jamais pour sélectionner qui est brouillé.

## 4 — Scénarios d'acceptation obligatoires

1. **Quatre navires A/B/C/D** : A envoie un ordre protégé à B ; C brouille ; D écoute et peut détecter l'émission de C sans connaître automatiquement son identité, sa position exacte ou le texte de l'ordre. L'arrêt/panne de C rétablit la liaison si rien d'autre ne la bloque.
2. Brouillage réellement effectif sur bande compatible et portée adéquate ; aucun effet si bande incompatible, brouilleur éteint/sans énergie/en panne/hors portée. WIRE n'est pas brouillable par RF ; SATCOM/SUBMARINE restent hors modèle détaillé tant que la physique nécessaire n'est pas implémentée.
3. Source alliée, ennemie ou neutre : **mêmes effets** à conditions physiques égales. Tester aussi le brouillage de ses propres communications.
4. Contre-mesure canal/radio de secours : restauration possible uniquement si la nouvelle paire d'équipements et le canal sont compatibles ; pas de contournement d'EMCON ni de sécurité.
5. Récepteur passif : signal de brouillage observable ; résultat anonyme, avec position absente/estimation incertaine ; aucune cible engageable automatiquement.
6. Rapport sous HOLD / lien coupé : l'observation reste locale ; connaissance du commandement inchangée avant une livraison effective.
7. Effets réversibles après arrêt, déplacement, perte d'énergie, panne et restauration ; cas limites géographiques et temporels.
8. Multibrouilleurs / multicapteurs / simulation longue : limites CPU/mémoire, pas de croissance infinie, déterminisme et reprise JSON.
9. Préparer des tests **d'adaptateurs** montrant les points où le vrai `LinkAssessor`, `ObservationEngine`, `CommsNetwork` et `EquipmentDef` seront raccordés **plus tard** ; ne pas marquer ces intégrations comme passées avant leur exécution dans TJAZ.
10. Si le moteur Godot 4.7.2 est disponible, créer un projet Godot minimal et exécuter les suites headless. Publier commandes exactes, manifestes, échecs et limites. Ne pas exiger ni construire d'APK.

## 5 — Performance et sauvegardes

Priorité mobile-first : pas de balayage `toutes unités × tous brouilleurs × tous capteurs` à chaque frame. Quand aucune source EW n'est active, coût incrémental négligeable. Avec brouillage, utiliser une sélection spatiale ou une indexation locale, des mises à jour au tick, des caches invalidés aux transitions, des bornes sur les événements et des évaluations seulement lorsque nécessaires.

Profilage PC : latence de tick, coût du calcul EW, taille des événements et mémoire, cas dense. **Aucune affirmation sur Android** sans tests sur Android. Sauvegarder les sources EW, modes, temps et les compteurs nécessaires ; même résultat avant/après JSON, y compris avec ordre d'itération stable.

## 6 — Ce qui reste hors mission

- Nouvelle cryptographie, cryptanalyse, contrôle J6 de clés ou cyber offensif ;
- physique détaillée de radar, sonar, infrarouge, optique et orbites (OBS-02) ;
- catalogue industriel complet de brouilleurs, coûts et réparation des équipements ([post-it #25 TJAZ](https://github.com/Lowagradi/TJAZ/issues/25)) ;
- analyse stratégique J2 et alertes autonomes (J2-ANALYSIS-01) ;
- nouveaux assets 2D, visuels NMCC, carte 3D ;
- intégration au vrai dépôt TJAZ, APK et déploiement.

## 7 — Livrables avant audit et éventuel portage

Dans `my-R-project`, livrer :
1. un module EW Godot testable, autonome et documenté, sans changement TJAZ ;
2. les fixtures/scénarios A/B/C/D, tests positifs/négatifs et mesures de performance ;
3. `docs/missions/ew-01/IMPLEMENTATION_REPORT.md` avec architecture, inventaire fichiers, hypothèses, ce qui est **réellement testé en isolé**, écarts connus, commande Godot, traces et SHA ;
4. `docs/missions/ew-01/PORTING_PLAN.md` décrivant précisément chaque module ou adaptateur futur, les classes TJAZ à réutiliser et les risques de conflit. **Un plan, pas une copie.**
5. une PR **dans ce dépôt** seulement, ciblant `mission/ew-01-isolated` si une branche de code dérivée est utilisée, ou commits dans cette branche de mission en conservant la PR documentaire ouverte. Aucun merge.

### Conditions d'acceptation

Refuser une solution qui : invente une position/identité de brouilleur sans acquisition, utilise des identifiants cachés pour produire le renseignement d'un acteur, donne des dégâts/brouillages permanents, ignore des radios/états matériels, modifie la sécurité J6, ne distingue pas la validation isolée de l'intégration réelle, ou écrit quoi que ce soit dans TJAZ.

**À la fin :** donner le lien PR de `my-R-project`, les commits, les résultats et le rapport. **S'arrêter ensuite.** Attendre que le propriétaire demande une revue indépendante avant tout portage vers TJAZ.
