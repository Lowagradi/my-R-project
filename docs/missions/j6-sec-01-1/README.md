# TJAZ — J6-SEC-01.1 : corrections ciblées de sécurité (développement isolé)

> **Dépôt de travail exclusif :** [Lowagradi/my-R-project](https://github.com/Lowagradi/my-R-project)  
> **Branche de mission :** `mission/j6-sec-01-1-isolated` · PR **distincte** d'EW-01  
> **Référence technique source, lecture seule :** [TJAZ PR #28 — J6-SEC-01](https://github.com/Lowagradi/TJAZ/pull/28), HEAD audité `c1478b8b0656ff6ffa289a4f34906135fba729e3`  
> **Revue des défauts :** [commentaire d'audit J6-SEC-01.1](https://github.com/Lowagradi/TJAZ/pull/28#issuecomment-6657033615)  
> **Autre chantier isolé, sans mélange :** [EW-01 — my-R-project PR #1](https://github.com/Lowagradi/my-R-project/pull/1)

## 0. Contrat de la mission : ne surtout pas toucher TJAZ

L'utilisateur veut que **tout développement et toute vérification préalable se déroulent dans `Lowagradi/my-R-project`**. Tu peux consulter le dépôt TJAZ **en lecture seule** ; tu ne peux y effectuer **aucun commit, aucun push, aucune branche, aucune PR, aucun changement de fichiers**. La copie/portage dans TJAZ ne sera envisagé qu'après audit indépendant et nouvelle autorisation explicite du propriétaire.

**Ce dépôt isolé était initialement vide.** Les classes `SimulationEngine`, `CommsSecurity`, `CommsNetwork`, `LocalTacticalKnowledge`, `ActorKnowledge`, ainsi que leurs tests Godot **n'y sont pas déjà installés**. Il faut donc préparer un **correctif portable minimal** et un banc d'essais Godot 4.7.2 indépendant, sans cloner ni importer tout le jeu TJAZ.

**Ne pas coder EW-01 ici.** EW-01 possède déjà sa branche `mission/ew-01-isolated` et sa PR #1. J6-SEC-01.1 part de `main` et reste totalement séparée.

**Interdictions :** pas de merge dans `main` d'aucun dépôt, aucun APK/export Android, aucun déploiement, aucune édition de workflows GitHub, aucune suppression de travail existant, aucun `reset --hard`, `clean` ou force-push. Ne pas envoyer d'e-mail ni solliciter de notifications.

## 1. Portée exacte : DEUX anomalies, pas une refonte de J6

### Défaut 1 — Un ordre ouvert peut être accepté sans autorité

**Chemin vérifié par lecture du code TJAZ audité :**

1. `SimulationEngine._validate_command(SEND_ORDER)` et `_run_send_order` vérifient la plateforme **émettrice** et l'acteur qui la commande, mais pas systématiquement le droit de commander l'unité **destinataire**.
2. `CommsSecurity.validate(m, world, now)` retourne un succès lorsque `m.security.is_empty()` et que la cible ne requiert pas de profil protégé. Le contrôle `trusted[target]` et celui de destination se trouvent dans le parcours protégé.
3. `SimulationEngine._deliver_message` peut alors écrire l'ordre ouvert dans `ActorKnowledge.received_orders` et le refléter chez l'acteur propriétaire de B.
4. `_order_execution_gate` contrôle l'existence de l'ordre reçu pour B, pas indépendamment l'autorité d'A.

**Scénario de reproduction** : pays/acteur A contrôle navire A. Pays/acteur B contrôle navire B. Communications B ouvertes. A expédie un `SEND_ORDER` valide depuis son propre navire vers B via la voie COMMS existante. Si transport arrive, l'ordre d'A ne doit **jamais** conférer un droit de commander/exécuter aux forces B sans une délégation explicite de B.

**Correctif attendu** :

- **Séparer strictement** confidentialité, authentification technique et **autorité de commandement**. L'absence de chiffrement ne supprime jamais le contrôle d'autorité.
- Pour tout message de nature opérationnelle (`Kind.ORDER` **ou** payload `type=order`, y compris les tentatives de confusion de type), vérifier avant insertion dans `received_orders` une autorité fondée sur les propriétaires réels / relations de commandement ou sur une délégation **explicite et vérifiable**. Le destinataire réel et la cible demandée doivent concorder.
- Une communication étrangère non autorisée peut être **émise, détectée et transportée**, mais demeure un ordre **rejeté**, jamais exécutable et sans ACK d'acceptation.
- Conserver le parcours legacy **ouvert et autorisé** entre unités appartenant au même acteur ainsi que les cas de délégation explicitement prévus. Les commandes `authority=technical|scenario` ne doivent pas introduire une faille jouable : elles restent limitées à leurs contextes légitimes préexistants.
- Une **clé compromise** ou un message techniquement authentifié ne transfère pas l'autorité, y compris sur un appareil étranger.
- L'identité déclarée dans le message ouvert n'est **pas** une preuve cryptographique ; les règles d'autorité doivent être explicites, fondées sur les objets et actions déjà contrôlés par le moteur.

### Défaut 2 — Rapport marqué « livré » alors que sa sécurité a été refusée

**Chemin vérifié par lecture du code TJAZ audité :**

1. `SimulationEngine._deliver_message` met `SimMessage.status=DELIVERED` et appelle `_note_local_report_status(m)`.
2. **Ensuite seulement** `comms_security.validate(m,...)` peut rejeter le message et sortir sans appeler `ActorKnowledge.remember_observation` ni délivrer le rapport à J2.
3. La mémoire locale peut donc considérer une observation `delivered` alors qu'aucune connaissance n'a été acceptée.

**Scénario de reproduction** : capteur C détecte un signal localement, crée un rapport automatique ou explicite, le transporte vers son commandement ; le transport arrive, mais le profil/clé de réception invalide l'application. Le rapport ne doit ni apparaître chez J2, ni devenir à tort un succès applicatif local.

**Correctif attendu** :

- Distinguer clairement **transport arrivé** et **rapport accepté par l'application / ActorKnowledge**.
- Préserver si possible `SimMessage.status=DELIVERED` pour la réception physique, mais exposer explicitement `security_result` et un état local `rejected/failed/held` (ou équivalent contractuel) si l'application refuse. Ne jamais marquer `RS_DELIVERED` ni `last_delivered_obs_id` en cas de refus.
- Si l'état de rejet exige une nouvelle tentative, la politique doit être bornée et conserver l'EMCON, les fenêtres AUTO, les plafonds, les priorités et la protection anti-feedback SIGINT-02. Ne pas provoquer de boucle de rapports de sécurité. Aucun renseignement ne doit passer à J2 avant acceptation réelle.
- Un nouveau message légitime avec la clé corrigée doit pouvoir être délivré **une fois**, sans dupliquer ni réinventer une acquisition ; préserver `acquired_at_s` et enregistrer `received_at_s` de la véritable acceptation.
- Ne pas traiter les rejets comme des anomalies globales magiquement visibles par J6 ; l'incident doit rester local et ne devenir connaissance commandement qu'après un rapport réellement livré.

## 2. Méthode imposée à Codex

1. **Auditer sans écrire dans TJAZ** les chemins pertinents : `scripts/sim/engine/simulation_engine.gd`, `scripts/sim/comms/comms_security.gd`, `scripts/sim/comms/message.gd`, `scripts/sim/knowledge/local_tactical_knowledge.gd`, `scripts/sim/knowledge/actor_knowledge.gd`, `tests/test_j6_sec_01.gd` et contrats des ordres. Vérifier si la version publique TJAZ a changé depuis le SHA de référence ; documenter la différence, sans écriture.
2. Reproduire séparément les **deux erreurs sur un banc de test minimal**, avec fixtures `TEST_ONLY` et ordre de phases `envoi → transport → validation → connaissance → exécution`. Capturer une trace « avant correctif », si reproductible avec le modèle fidèle aux contrats audités.
3. Concevoir la correction **la plus étroite** possible sous une forme isolée et portable. Privilégier une fonction indépendante de décision d'autorité et une fonction d'application/accusé de livraison qui ne modifient pas les statuts trop tôt. Ne pas copier l'intégralité de `SimulationEngine` ou tout le dépôt dans `my-R-project`.
4. Ajouter des tests « après correctif » qui prouvent les résultats et l'absence de régression des cas legacy. Conserver des identifiants stables, des données déterministes et des structures sérialisables en JSON.
5. Préparer un **plan de portage précis**, indiquant les fichiers et fonctions TJAZ qu'il faudra changer plus tard et les tests à exécuter **dans TJAZ après autorisation**. Ne pas faire ce portage maintenant.

La mission n'est pas « terminée dans TJAZ » parce que le prototype isolé passe les tests. L'état demandé est : **correctif conçu et testé de manière indépendante, intégration TJAZ en attente d'audit**.

## 3. Batterie de tests obligatoire

Pour l'autorité :
- A/acteur A vers B/acteur B **ouvert et non délégué** : transport livré, ordre rejeté, aucune entrée `received_orders`, aucun ordre exécutable.
- A vers B **même propriétaire** : fonctionnement legacy préservé, reçu/exécutable dans les conditions existantes.
- A vers B **délégation explicite vérifiée** : fonctionnement autorisé selon un contrat borné ; absence de délégation, révocation ou propriétaire non concordant : refus.
- A vers B **protégé** mais non autorisé : refus ; une clé valide n'accorde pas à elle seule la délégation.
- Altération `Kind.REPORT` + `payload.type=order`, mauvaise destination, acteur émetteur non propriétaire : refus.
- Scénario technique/scénario existant légitime : pas de régression ni contournement des autorités jouables.

Pour la livraison de renseignement :
- Rapport SIGINT transport `DELIVERED`, sécurité `rejected` : aucune observation, aucun rapport J2, état local ≠ `RS_DELIVERED`, pas de `last_delivered_obs_id`.
- Rapport transport `DELIVERED`, sécurité acceptée : observation J2 reçue une fois, statut local `RS_DELIVERED`, dates d'acquisition/réception cohérentes.
- Réparation de clé et **nouvel envoi autorisé** : comportement correct sans duplication infinie ni réécriture du temps d'acquisition.
- EMCON, panne, absence de route, HOLD, AUTO borné, répétition sur plusieurs ticks : pas de livraison magique, pas de boucle auto.
- Sauvegarde/reprise JSON dans chaque phase ; résultats déterministes avant/après restauration.
- Acteurs joueur et IA : mêmes règles de sécurité/autorité, sans exception joueur.

**Si Godot 4.7.2 est disponible**, créer un projet Godot minimal avec tests headless ; inclure commandes et logs. Sinon, le signaler et ne pas prétendre avoir exécuté les tests. Aucun APK.

## 4. Livrables et validation

- Code du correctif **dans ce dépôt isolé**, avec petit module portable et stubs clairement marqués `TEST_ONLY`.
- `docs/missions/j6-sec-01-1/IMPLEMENTATION_REPORT.md` : bugs reproduits ou non, choix, variantes, preuves, limites, tests exécutés (versus seulement envisagés).
- `docs/missions/j6-sec-01-1/PORTING_PLAN.md` : patch conceptuel fichier/fonction par fichier/fonction, préconditions, risques, règles de compatibilité, plan de tests complets Godot 4.7.2 à appliquer à TJAZ **après autorisation**.
- Traces et manifeste de tests réellement effectués, plus SHA du code.
- Code en branche `mission/j6-sec-01-1-isolated` ou PR de code dérivée ciblant cette branche ; jamais fusion vers main.

**Critères de refus :** nouvelle autorité étrangère par messages ouverts ; faux rapport localement « livré » ; effacement de la connaissance historique ; fuite d'identifiants, clés ou texte dans J2 ; nouvelles boucles de transmissions ; refonte COMMS/J6/OBS inutile ; affirmation de compatibilité TJAZ sans test dans TJAZ ; **toute écriture dans TJAZ**.

**Arrêter le travail après publication dans my-R-project.** Le propriétaire demandera l'audit indépendant avant tout portage vers TJAZ. Pas d'envoi d'e-mail, de notification volontaire ni de déploiement.
