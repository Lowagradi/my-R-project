# CONSIGNE CODEX — J6-SEC-01.1 (REPRISE / CORRECTIF ISOLÉ)

**Travaille uniquement dans** `Lowagradi/my-R-project` sur `mission/j6-sec-01-1-isolated`.

Lis en entier `docs/missions/j6-sec-01-1/README.md`.

La source de contexte technique est **TJAZ PR #28**, code audité au commit `c1478b8b0656ff6ffa289a4f34906135fba729e3`, accessible **en lecture seule**. Le dépôt actuel est volontairement isolé et ne contient pas les classes TJAZ. **N'écris jamais dans Lowagradi/TJAZ** et **ne travaille pas sur la branche EW-01**.

**Deux corrections seulement :**
1. L'autorité d'exécuter un ordre doit être validée même pour les ordres ouverts : un acteur A ne peut pas commander l'unité B d'un autre acteur sans délégation explicite. Une clé valide ne donne pas automatiquement autorité.
2. Quand le transport livre un rapport mais que sa sécurité le rejette, la mémoire locale ne doit **pas** le marquer remis à J2. Garder la distinction transport reçu / donnée acceptée. J2 ne reçoit rien avant validation.

**Marche à suivre :** inspecte la référence publique → produis des tests de reproduction → code des correctifs minimaux et réutilisables en Godot 4.7.2 avec fixtures TEST_ONLY → teste sauvegarde JSON, anti-rejeu, IA/joueur, autorité, HOLD/AUTO, refus/acceptation → publie traces et résultats dans `IMPLEMENTATION_REPORT.md` → prépare `PORTING_PLAN.md` sans effectuer le portage.

Pousse seulement dans `my-R-project`. Conserve la PR de mission ou ouvre une PR dérivée ici. Communique URL de la PR, SHA, nombre de tests passés et limites. **Arrête-toi ensuite pour audit indépendant.**

**Interdictions :** aucun commit/PR/copie dans TJAZ, aucun merge main, aucun APK/export Android, aucun déploiement, aucun changement de workflows, aucun envoi d'e-mail ni @mention de notification.
