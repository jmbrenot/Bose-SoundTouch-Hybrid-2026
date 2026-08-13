# Intégration Modbus onduleur DEYE — Gruissan

Suivi de projet — intégration de l'onduleur hybride **DEYE SUN-6K-SG05LP1-EU-AM2-P** dans Home Assistant, via une passerelle RS485→Modbus TCP. Le Growatt est déjà intégré via SolaX Modbus (HACS) ; ce document couvre l'ajout du DEYE en complément. Établi le 10 août 2026.

## Checklist Jour J (à suivre dans l'ordre)

1. **Sauvegarde HA avant de commencer** — Réglages → Système → Sauvegardes, ou bouton « Lancer » de la carte Sauvegarde complète (vue Système du dashboard SCADA, `script.sauvegarde_complete_demarrer_dashboard`). Utile vu que `comdif/ha-solarmodbus` est un projet jeune avec accès Modbus en écriture.
2. **Vérifier le brochage RS485 réel** de l'onduleur (étiquette/manuel du SUN-6K-SG05LP1-EU-AM2-P) contre le tableau de la section « Câblage » ci-dessous — ne pas supposer qu'il est identique au modèle SM2 documenté par le projet.
3. **Câbler** : Onduleur RS485-A → passerelle borne A, Onduleur RS485-B → passerelle borne B.
4. **Régler l'ID Modbus de l'onduleur sur `4`** (menu Modbus/RS485 de l'écran ou de l'appli DEYE).
5. **Fixer l'IP de la passerelle** `192.168.1.170` — réservation DHCP sur le routeur, ou IP statique via l'onglet « Local IP Config » de la passerelle.
6. **Installer `comdif/ha-solarmodbus`** — SSH sur HA (add-on Terminal & SSH), coller :
   ```bash
   mkdir -p /config/custom_components/solarmodbus && curl -fsSL https://github.com/comdif/ha-solarmodbus/archive/refs/heads/main.zip -o /tmp/sm.zip && unzip -o /tmp/sm.zip -d /tmp && mv /tmp/ha-solarmodbus-main/solarmodbus/* /config/custom_components/solarmodbus/ && ha core restart
   ```
7. **Ajouter l'intégration** : Réglages → Appareils et services → Ajouter une intégration → *Solarmodbus* → mode **Modbus TCP (Ethernet / LAN)** → `host` `192.168.1.170`, `port` `502`, `slave_id` `4`, `model` `deye_hybrid` → Envoyer.
8. **Vérifier les entités créées** (SOC batterie, puissance, tension réseau…) contre les valeurs affichées sur l'écran/l'appli de l'onduleur.
9. **Si tout est bon** : cocher la checklist détaillée plus bas, compléter le journal, puis passer à l'ajout des badges ONDULEUR sur le synoptique Kilovac (`kilovac-batterie-gruissan-runbook.md`, item 9).
10. **Si ça bloque** : voir la réserve en section « Décision retenue » — se replier sur l'intégration `modbus:` native de HA si le projet a un problème bloquant.

## Contexte

- Onduleur cible : **DEYE SUN-6K-SG05LP1-EU-AM2-P**.
- Matériel de liaison : passerelle **USR-TCP232-410S** (USR IOT), nom de module `INVERT-TCP232-410S`, IP `192.168.1.170`, firmware V8.0.12.000000.0000, MAC `D4-AD-20-83-E7-FE`. Confirmé le 10 août 2026 via l'interface web du module.
- Le sélecteur `PORT Status: RS232` sur la page « Current Status » est un sélecteur d'affichage de statistiques par port, pas une indication du mode actif — l'onglet **RS485** dédié montre une configuration déjà en place et cohérente (voir ci-dessous), donc pas de bascule nécessaire de ce côté.
- Intégration existante pour le Growatt : `wills106/homeassistant-solax-modbus` (HACS), avec un merge local automatisé (`script.solax_modbus_update_hacs_then_merge` dans `scripts.yaml`).

## Options évaluées

| Option | Verdict | Raison |
|---|---|---|
| `wills106/homeassistant-solax-modbus` (déjà utilisé pour le Growatt) | ❌ Écarté | Ne supporte pas Deye/Sunsynk — vérifié sur le dépôt : Solis, Growatt, Sofar, SolaX, Solinteg/SRNE/Swatten (WIP) uniquement. |
| `StephanJoubert/home_assistant_solarman` | ❌ Écarté | Supporte bien Deye/Sunsynk/SolArk, mais parle le protocole propriétaire Solarman via le boîtier WiFi/LAN du fabricant — pas conçu pour une passerelle Modbus TCP générique. |
| `kbialek/deye-inverter-mqtt` | Solution de repli | Microservice Docker, mode `mbtcp` = Modbus/TCP standard, mais modèles Deye documentés comme supportés = SG01LP1/SG02LP1 (SG05LP1 non confirmé explicitement), nécessite un plugin MQTT séparé pour la découverte HA. |
| Intégration `modbus:` native HA (YAML, registres manuels) | Solution de repli | Zéro dépendance tierce, garanti compatible avec n'importe quelle passerelle Modbus TCP — déjà utilisé pour les 9 micro-onduleurs Hoymiles côté Gruissan. Demande de recopier les registres depuis la doc Modbus publique Deye. |
| **`comdif/ha-solarmodbus`** | **✅ Retenu** | Modbus TCP standard testé avec passerelle tierce (EBYTE NA111), **document de référence du projet utilise littéralement le modèle Deye SG05LP1-EU-SM2-P** — quasi identique au SUN-6K-SG05LP1-EU-AM2-P (seul le suffixe AM2/SM2 diffère). Configuration 100% UI, aucun registre à définir manuellement. |

## Décision retenue

**`comdif/ha-solarmodbus`**, en mode Modbus TCP, profil de modèle `deye_hybrid`.

Réserve à garder en tête : projet jeune (2 étoiles GitHub au 10 août 2026, un seul mainteneur), pas de release versionnée, installation via script qui télécharge et exécute du code sur l'hôte HA, avec accès Modbus en écriture à l'onduleur (pas juste lecture). Pas de garantie de maintenance à long terme — si le projet montre des signes d'abandon ou un bug bloquant, se replier sur `modbus:` natif (registres à extraire de la doc Modbus publique Deye).

## Installation

### Sur HAOS (via l'add-on Terminal & SSH)

1. Réglages → Modules complémentaires → Installer *Terminal & SSH* si pas déjà fait.
2. Ouvrir le terminal, coller :
   ```bash
   mkdir -p /config/custom_components/solarmodbus && curl -fsSL https://github.com/comdif/ha-solarmodbus/archive/refs/heads/main.zip -o /tmp/sm.zip && unzip -o /tmp/sm.zip -d /tmp && mv /tmp/ha-solarmodbus-main/solarmodbus/* /config/custom_components/solarmodbus/ && ha core restart
   ```
3. HA redémarre automatiquement.

### Autres installations (venv, conteneur, etc.)

Copier le dossier `solarmodbus` du dépôt dans `/config/custom_components/solarmodbus/` via SFTP, puis redémarrer HA.

## Câblage — à vérifier avant de brancher

La doc du projet documente le brochage RJ45 du **SG05LP1-EU-SM2-P** (modèle de référence, pas forcément identique au vôtre) :

| Pin RJ45 | Signal | Couleur (câblage B) |
|---|---|---|
| 1 | sunspec-485_B | Blanc/Orange |
| 2 | sunspec-485_A | Orange |
| 3, 6 | GND_sunspec-485 | — |
| 7 | sunspec-485_A | Blanc/Marron |
| 8 | sunspec-485_B | Marron |

Choix retenu dans la doc : Pin 1 (Blanc/Orange) = RS485-A, Pin 2 (Orange) = RS485-B (paire 1-2, la paire 7-8 fonctionne aussi).

**⚠️ Le suffixe du modèle diffère (AM2 vs SM2) — confirmer ce brochage sur l'étiquette ou le manuel exact du SUN-6K-SG05LP1-EU-AM2-P avant de câbler**, le projet précise explicitement que ce n'est pas universel selon les modèles Deye/Sunsynk (bloc à vis, connecteur RS485 dédié, port CAN/RS485 partagé, ou pas de RJ45 Modbus du tout selon les variantes).

Câblage générique vers la passerelle : `Onduleur RS485-A → Passerelle A`, `Onduleur RS485-B → Passerelle B`.

## Configuration de la passerelle

### Matériel réel : USR-TCP232-410S (192.168.1.170)

La doc du projet `comdif/ha-solarmodbus` utilise un EBYTE NA111 comme référence, mais la passerelle réelle côté Gruissan est un **USR-TCP232-410S**. Onglet **RS485** vérifié le 10 août 2026 — la configuration Socket A est **déjà en place et correcte** :

| Paramètre | Valeur constatée | Statut |
|---|---|---|
| Baud Rate | 9600 | ✅ Standard Deye/Modbus |
| Data bit / Parity / Stop bit | 8 / None / 1 | ✅ 8-N-1 standard |
| Flow ctrl | NONE | ✅ |
| Socket A — Work Mode | **TCP Server** | ✅ |
| Socket A — Protocole | **ModbusTCP** (menu déroulant à côté de Work Mode) | ✅ — c'est l'équivalent de la conversion Modbus TCP↔RTU de la doc de référence |
| Socket A — Local Port Number | **502** | ✅ Correspond à ce qu'attend `comdif/ha-solarmodbus` |
| Socket A — Modbus Poll | Activé, timeout 200 ms | ✅ |
| Socket A — TCP Server MAX Sockets | 8, Up to MAX: KICK | OK par défaut |
| Socket B | WorkMode: NONE | Inutilisé, normal |

**Rien à changer sur cette page.** Reste à vérifier : `Local IP Config` (IP fixe plutôt que bail DHCP) et `RS232` (s'assurer qu'aucun autre appareil ne dépend de ce port sur la même passerelle, sans impact sur RS485).

### Paramètres cibles (pour référence / autre matériel)

- Work mode : **TCP server**
- Local port : **502**
- Baud rate : **9600**, Data bit 8, Parity NONE, Stop bit 1
- Conversion Modbus TCP ↔ RTU : **activée** (ici via le protocole « ModbusTCP » du Work Mode)

## Configuration Home Assistant

Entièrement via l'UI, pas de YAML :

1. Réglages → Appareils et services → **Ajouter une intégration**
2. Rechercher **Solarmodbus**
3. Mode : **Modbus TCP (Ethernet / LAN)**
4. Renseigner :
   - `host` : `192.168.1.170` (USR-TCP232-410S)
   - `port` : `502`
   - `slave_id` : **`4`** (choisi par l'utilisateur — doit être réglé à l'identique côté onduleur, voir menu Modbus/RS485 du DEYE ; le Growatt et tout autre appareil Modbus sur le même bus doivent avoir un ID différent)
   - `model` : **`deye_hybrid`**
5. Envoyer — les entités apparaissent automatiquement.

## Dépannage — erreur au premier démarrage (12-13 août 2026)

### Symptômes

- Écran de config Solarmodbus : « Unknown error occurred » (générique, pas exploitable seul).
- `nc -zv 192.168.1.170 502` → **open** : le réseau et la passerelle sont OK, le problème est plus haut dans la pile.
- Log HA (`config/system_log`, filtré sur `solarmodbus`) :
  - `WARNING [homeassistant.util.loop] Detected blocking call to listdir ... custom_components/solarmodbus/config_flow.py, line 46` — bug de qualité du code (appel bloquant `os.listdir` dans une coroutine async), signalé à l'auteur du composant mais pas forcément la cause de l'échec final.
  - `ERROR [homeassistant.config_entries] Setup of config entry 'Solarmodbus TCP (192.168.1.170)' ... cancelled` → traceback se terminant par `asyncio.exceptions.CancelledError` pendant `coordinator.py:130 _async_update_data`, c'est-à-dire un **timeout** lors du tout premier rafraîchissement de données.
  - Juste après : `[custom_components.solarmodbus.coordinator] Exception reading block 192-192: 'NoneType' object has no attribute 'recv'` — le client Modbus TCP interne est `None`/déconnecté au moment de lire, cohérent avec une connexion coupée après le timeout précédent.

### Cause probable : deux intégrations sur le même bus RS485 en même temps

Le log montre que **l'intégration `Solarman` (StephanJoubert, celle écartée dès le départ) était encore active en parallèle**, avec sa propre entrée de config (`Deye Gruissan`) qui échoue elle aussi : `[custom_components.solarman] ... 'NoneType' object has no attribute 'parser'; Retrying in 600 seconds`.

RS485 est half-duplex : un seul maître Modbus peut parler au bus à la fois. Deux intégrations qui pollent le même onduleur via la même passerelle en même temps peuvent se percuter et provoquer exactement ce type d'erreurs de connexion qui semblent aléatoires.

Timeout de la passerelle (`Response Timeout: 200 ms` sur Socket A) probablement trop court pour un aller-retour RS485 à 9600 bauds sur un bloc de registres, encore plus avec de la contention sur le bus.

### Plan de résolution

1. **Supprimer complètement l'intégration Solarman** (Réglages → Appareils et services → carte Solarman → ⋮ → Supprimer) — elle ne doit pas rester active, elle a été écartée dès le départ et contend pour le même bus.
2. **Augmenter le timeout Modbus de la passerelle** : onglet RS485 → Socket A → `Response Timeout` `200` → `1000` ms.
3. **Supprimer puis recréer l'intégration Solarmodbus** (pas un simple reload) avec les mêmes paramètres (host `192.168.1.170`, port `502`, slave_id `4`, modèle `deye_hybrid`).
4. **Si `NoneType has no attribute 'recv'` persiste** malgré l'absence de Solarman et le timeout augmenté : probable bug du coordinator de `comdif/ha-solarmodbus` (projet jeune, peu testé) — se replier sur l'intégration `modbus:` native de HA (registres à extraire de la doc Modbus publique Deye), comme prévu en réserve.

## Pivot vers `modbus:` natif (13 août 2026)

### Constat

Une fois la contention Solarman résolue, l'entrée Solarmodbus se crée sans erreur, mais **les valeurs lues sont physiquement aberrantes** (SOC batterie à 0 % dans HA contre 69 % réel sur l'écran DEYE, tension batterie à 0 V, températures à 400 °C / -100 °C, énergies journalières à plusieurs milliers de kWh pour un onduleur 6 kW).

En comparant `custom_components/solarmodbus/inverter_definitions/deye_hybrid.yaml` (fourni par l'utilisateur) aux adresses de registres généralement documentées pour les onduleurs hybrides Deye, les adresses elles-mêmes (SOC à `0x00B8`, tension batterie à `0x00B7`, etc.) semblent correctes. Toutes les valeurs fausses proviennent de registres situés dans le **deuxième bloc de lecture groupée** du projet (`0x0096`-`0x00F9`, 100 registres en une seule requête Modbus) — ce qui pointe vers un **bug de découpage/indexation** dans le code de l'intégration au moment de retranscrire ce bloc en valeurs individuelles, pas vers une table de registres fausse.

### Décision

Plutôt que de dépendre d'un correctif dans un projet jeune et peu testé, **construction directe de la même lecture via l'intégration `modbus:` native de HA**, en reprenant telles quelles les adresses de `deye_hybrid.yaml` (converties de hexadécimal en décimal). Ça évite complètement le bug de `comdif/ha-solarmodbus` puisque HA lit et décode les registres lui-même, sans passer par son code.

Fichier complet mis à jour : [`gruissan-configuration-merged.yaml`](./gruissan-configuration-merged.yaml) — nouveau hub `modbus:` nommé `deye` (type `tcp`, host `192.168.1.170`, port `502`) ajouté **dans la clé `modbus:` existante**, juste après le hub `hoymiles`, avec 32 capteurs couvrant le solaire, la batterie, le réseau, la charge et l'état onduleur. Diff vérifié : uniquement des ajouts (352 lignes), rien de supprimé ni modifié ailleurs.

### Points à vérifier après déploiement

- **Champs 32 bits (`Production Totale`, `Batterie Charge/Décharge Totale`, `Energie Vendue Totale`)** : l'ordre des mots (word order) entre les deux registres de chaque paire n'est pas confirmé. Si une valeur totale semble absurde (proche de 0 ou énorme), ajouter `swap: word` au capteur concerné.
- **`DEYE Statut Fonctionnement (code)`** et **`DEYE Statut Reseau (code)`** : exposés en code numérique brut (0-4 et 0-1 respectivement), pas encore traduits en texte. Table de correspondance en commentaire dans le fichier ; à transformer en template sensor si besoin d'un texte lisible.
- **`Total Energy Bought`** de la définition d'origine a été exclu : ses deux registres (`0x004E`, `0x0050`) ne sont pas consécutifs dans le fichier source (ça saute `0x004F` = Grid Frequency), ce qui sent l'erreur dans le fichier d'origine — pas assez fiable pour être repris tel quel.
- **⚠️ Contention potentielle** : une fois `modbus:` natif validé, **désinstaller/désactiver l'intégration Solarmodbus** (Réglages → Appareils et services → Solarmodbus → ⋮ → Supprimer) — sinon on recrée exactement le même problème de contention RS485 qui a bloqué Solarman/Solarmodbus au début, cette fois entre `modbus:` natif et Solarmodbus.

## Alternative testée en parallèle : découper le bloc de lecture dans `deye_hybrid.yaml`

Plutôt que de dupliquer toute la lecture dans `modbus:` natif, une piste plus légère : le bloc fautif `0x0096`-`0x00F9` lit 100 registres en une seule requête Modbus — sur un lien RS485 9600 bauds via passerelle, une lecture aussi large peut se corrompre/tronquer sans erreur explicite, ce qui expliquerait des valeurs fausses sans exception levée (plutôt qu'un bug d'indexation Python).

Fichier prêt à déposer : [`deye_hybrid-patched.yaml`](./deye_hybrid-patched.yaml) — à copier vers `/config/custom_components/solarmodbus/inverter_definitions/deye_hybrid.yaml` (écrase l'original). Seule la section `requests:` change : le bloc de 100 registres est coupé en 4 blocs de 25 (`0x0096-0x00AE`, `0x00AF-0x00C7`, `0x00C8-0x00E0`, `0x00E1-0x00F9`). Diff vérifié contre l'original fourni par l'utilisateur : uniquement la section `requests:` touchée.

Si ça corrige les valeurs : garder Solarmodbus, supprimer le hub `modbus:` natif ajouté en secours (redondant). Si ça ne suffit pas : le `modbus:` natif déjà en place reste la solution de repli, il suffit de retirer Solarmodbus.

### Résultat (13 août 2026) : le patch ne corrige rien

Testé avec Solarmodbus seul (hub `modbus:` natif désactivé temporairement, donc pas de contention possible cette fois). Les valeurs sont restées **identiques au chiffre près** à l'échec initial (Battery SOC 0 %, Battery Voltage 0 V, Battery Temperature -100,0 °C, AC Temperature 400,0 °C, Daily Battery Charge 630,10 kWh, Daily Energy Sold 5000,00 kWh).

Ça élimine l'hypothèse « lecture RS485 trop large corrompue » : le découpage en blocs de 25 registres n'a rien changé, donc ce n'est pas une question de taille de requête. C'est un vrai bug dans le code de `comdif/ha-solarmodbus` (probablement dans `coordinator.py`, pas dans les définitions YAML), indépendant de la façon dont les registres sont regroupés pour la lecture. Pas creusé plus loin — projet jeune, peu de retour sur investissement à continuer à patcher à l'aveugle sans le code source du coordinator.

**Décision finale : abandon de `comdif/ha-solarmodbus`, le `modbus:` natif devient la solution retenue**, pas juste une solution de repli. Le hub `deye` a été réactivé dans `gruissan-configuration-merged.yaml` (commit `44a9e23`).

## Checklist

- [x] Passerelle identifiée : USR-TCP232-410S, IP `192.168.1.170` (confirmé le 10 août 2026)
- [x] Passerelle configurée côté RS485/Socket A : TCP Server + ModbusTCP, port 502, 9600-8-N-1, Modbus Poll activé (constaté déjà en place le 10 août 2026)
- [x] Brochage RS485 confirmé en pratique (câblage fait, onduleur démarré et raccordé)
- [x] Onduleur câblé → passerelle (A/B)
- [x] ID Modbus de l'onduleur DEYE réglé sur **4**
- [x] Onduleur démarré et raccordé à la passerelle USR `192.168.1.170`
- [x] `comdif/ha-solarmodbus` installé dans `custom_components` (12/13 août 2026)
- [x] Intégration ajoutée via l'UI (host `192.168.1.170`, port 502, slave_id 4, modèle `deye_hybrid`) — échec au premier refresh, voir « Dépannage » ci-dessus
- [x] Supprimer l'intégration Solarman (contention sur le bus RS485) — confirmé absente le 13 août 2026, HA redémarré ensuite
- [x] Augmenter le timeout Modbus de la passerelle (200 ms → 1000 ms) — confirmé le 13 août 2026 sur l'onglet RS485
- [x] Recréer proprement l'intégration Solarmodbus — **succès le 13 août 2026**, entrée « Solarmodbus Device » créée, plus d'erreur de setup. Confirme que la contention avec Solarman était bien la cause du blocage initial.
- [ ] Fixer l'IP de la passerelle (`192.168.1.170`) côté routeur/DHCP pour éviter qu'elle change (à confirmer — pas explicitement vérifié)
- [x] Diagnostic des valeurs aberrantes confirmé : SOC HA à 0 % contre 69 % réel — bug de découpage du bloc de registres côté `comdif/ha-solarmodbus`, pas une table de registres fausse
- [x] Hub `modbus:` natif « deye » construit dans `gruissan-configuration-merged.yaml` (32 capteurs, adresses reprises de `deye_hybrid.yaml`) — validé par parseur YAML, diff = ajouts uniquement
- [x] Testé `deye_hybrid-patched.yaml` (blocs de 25 registres) avec Solarmodbus seul, sans contention — **valeurs identiques à l'échec initial au chiffre près**, hypothèse « lecture RS485 trop large » écartée
- [x] **Décision finale** : abandon de `comdif/ha-solarmodbus` (bug interne, pas une histoire de registres ni de taille de bloc), `modbus:` natif retenu comme solution définitive. Hub `deye` réactivé dans `gruissan-configuration-merged.yaml`.
- [x] **Supprimer complètement l'intégration Solarmodbus** — testé le 13 août, confirmé retiré
- [x] Table de registres validée trouvée : `VMrenato/homeassistant-deye-tcan485-esphome`, testée sur un SUN-6K-SG05LP1-EU-AM2-P réel identique au nôtre. Confirme la plupart des adresses de `deye_hybrid.yaml`, mais corrige `Total Grid Import`/`Total Grid Export` (registres simples 78/81, pas des paires 32 bits) et confirme l'ordre **mot faible d'abord** pour les compteurs 32 bits.
- [x] Cause probable identifiée dans leur doc de dépannage : requêtes Modbus trop rapprochées pour un bus RS485 lent (9600 bauds) font atterrir les valeurs sur le mauvais capteur — symptôme identique au nôtre. Ajout de `message_wait_milliseconds: 100` au hub pour espacer les requêtes.
- [x] Hub `deye` entièrement reconstruit dans `gruissan-configuration-merged.yaml` à partir de cette table validée (29 capteurs + 1 binary_sensor « Reseau Connecte »), avec `swap: word` sur les 4 compteurs 32 bits et correction des deux champs Total Grid Import/Export
- [ ] Déployer `gruissan-configuration-merged.yaml` (hub `deye` reconstruit) sur l'hôte HA et redémarrer
- [ ] Vérifier les nouvelles entités `DEYE *` contre l'écran/l'appli de l'onduleur (SOC, tension batterie en priorité)
- [ ] Une fois validé : ajouter les badges de statut ONDULEUR sur le synoptique Kilovac (voir `kilovac-batterie-gruissan-runbook.md`, checklist item 9)

## Références

- Dépôt intégration abandonnée : `github.com/comdif/ha-solarmodbus`
- **Table de registres retenue** : `github.com/VMrenato/homeassistant-deye-tcan485-esphome` (MIT), validée sur un SUN-6K-SG05LP1-EU-AM2-P réel — voir `deye-register-spec.md` dans ce dépôt (archive complète : README, registres, câblage, dépannage). Elle-même dérivée de `StephanJoubert/home_assistant_solarman` (`deye_hybrid.yaml`) et recroisée avec `slipx06/Sunsynk-Home-Assistant-Dash`.
- Pas de documentation officielle Deye consultée — deyeinverter.com et les moteurs de recherche généralistes sont inaccessibles depuis cette session.

## Journal

### 10 août 2026
- Passerelle réelle identifiée : USR-TCP232-410S (pas l'EBYTE NA111 de la doc de référence), IP `192.168.1.170`, firmware V8.0.12.
- Onglet RS485 vérifié : Socket A déjà configuré correctement (TCP Server + ModbusTCP, port 502, 9600-8-N-1, Modbus Poll activé). Le `PORT Status: RS232` vu sur la page Current Status était un sélecteur d'affichage, pas le mode actif — fausse alerte corrigée.
- Reste : câblage physique onduleur → passerelle, vérification du brochage RJ45 exact du modèle AM2, IP fixe, puis installation/config de `comdif/ha-solarmodbus` côté HA.
- Décision : ID Modbus de l'onduleur DEYE fixé à **4** (à régler côté onduleur ET côté intégration HA — les deux doivent correspondre).
- Ajout d'une checklist « Jour J » consolidée en tête de document (recherche externe sur le brochage AM2 tentée mais bloquée par la politique réseau de la sandbox — pas d'info supplémentaire trouvée, réserve maintenue). Intégration prévue le lendemain.

### 11 août 2026
- Étapes 1 à 5 de la checklist Jour J faites : onduleur DEYE câblé, démarré, ID Modbus réglé sur `4`, raccordé à la passerelle USR-TCP232-410S (`192.168.1.170`).
- Reste : confirmer l'IP fixe côté passerelle/routeur, installer `comdif/ha-solarmodbus`, ajouter l'intégration côté HA (host `192.168.1.170`, port 502, slave_id 4, modèle `deye_hybrid`), puis vérifier les entités.

### 12-13 août 2026
- `comdif/ha-solarmodbus` installé et intégration ajoutée, mais échec au premier refresh (« Unknown error occurred » puis, dans les logs, timeout suivi de `'NoneType' object has no attribute 'recv'`).
- Diagnostic : l'intégration `Solarman` (écartée dès le départ) était restée active en parallèle et contend probablement pour le même bus RS485 half-duplex — voir section « Dépannage ». Plan : supprimer Solarman, augmenter le timeout Modbus de la passerelle à 1000 ms, recréer proprement l'entrée Solarmodbus.
- Tentative de log frais renvoyée deux fois identique (même hash) avant qu'un vrai nouveau log confirme que Solarman tournait toujours (`Deye Gruissan` en retry jusqu'à 600s) au moment de l'échec Solarmodbus — la suppression n'avait pas encore été effective.
- Solarman confirmé supprimé de la liste des intégrations, HA redémarré, Solarmodbus recréé : **setup réussi, plus d'erreur**. Diagnostic de contention confirmé.
- Nouveau problème constaté : valeurs d'entités physiquement aberrantes (températures 400°C / -100°C, tension batterie à 0V avec courant non nul, énergies journalières à 5000/630 kWh pour un onduleur 6 kW) — probable décalage de registres entre le profil `deye_hybrid` générique et le modèle réel. À vérifier contre l'écran/l'appli DEYE.
- SOC réel confirmé à 69 % (écran/appli DEYE) contre 0 % dans HA — écart confirmé, pas un arrondi.
- Fichier `deye_hybrid.yaml` du projet fourni et analysé : adresses de registres cohérentes avec la doc Deye généralement citée, mais toutes les valeurs fausses tombent dans le même bloc de lecture groupée (100 registres en une requête) — bug de découpage côté `comdif/ha-solarmodbus`, pas une table de registres erronée.
- Pivot décidé : construction d'un hub `modbus:` natif HA (« deye », 32 capteurs) directement à partir des adresses de `deye_hybrid.yaml`, dans `gruissan-configuration-merged.yaml`. Contourne le bug sans dépendre du projet tiers.
- Alternative testée en parallèle : `deye_hybrid-patched.yaml` découpant le bloc de 100 registres fautif en 4 blocs de 25, sur l'hypothèse d'une lecture RS485 trop large plutôt qu'un bug Python — moins de duplication si ça fonctionne, garde l'organisation par appareil de Solarmodbus.
- Premier test des deux en même temps (natif + Solarmodbus reconfiguré après redémarrage) : échec total des deux côtés (`No response received after 3 retries` sur toutes les adresses natives, même bug `NoneType`/`recv` côté Solarmodbus) — contention confirmée une deuxième fois. Hub `deye` commenté temporairement pour isoler le test.
- Test isolé de `deye_hybrid-patched.yaml` avec Solarmodbus seul (natif désactivé) : **valeurs identiques au chiffre près à l'échec initial**. Hypothèse « lecture RS485 trop large » écartée — bug interne à `comdif/ha-solarmodbus`, indépendant de la taille des blocs.
- **Décision finale** : abandon de `comdif/ha-solarmodbus`, `modbus:` natif retenu comme solution définitive (pas seulement une réserve). Hub `deye` réactivé dans `gruissan-configuration-merged.yaml`. Prochaine étape : supprimer Solarmodbus pour de bon, déployer, redémarrer, vérifier les entités `DEYE *`.

- Recherche d'une table de registres validée : trouvé `VMrenato/homeassistant-deye-tcan485-esphome`, testé sur un SUN-6K-SG05LP1-EU-AM2-P réel (notre modèle exact). Confirme la plupart des adresses de `deye_hybrid.yaml`, corrige `Total Grid Import`/`Total Grid Export` (registres simples 78/81, à tort combinés en 32 bits), confirme l'ordre mot faible d'abord pour les compteurs 32 bits.
- Leur doc de dépannage décrit noir sur blanc notre symptôme (« values landing on the wrong sensor, shifted by one », causé par des requêtes Modbus trop rapprochées sur un bus RS485 lent) — cause probable des deux échecs précédents (Solarmodbus ET premier essai natif), indépendamment des adresses de registres.
- Hub `deye` reconstruit intégralement à partir de cette table (29 capteurs + 1 binary_sensor), `swap: word` sur les 4 compteurs 32 bits, `message_wait_milliseconds: 100` ajouté pour espacer les requêtes.
- Archive complète (README, registres, câblage, dépannage de ce dépôt) sauvegardée dans `deye-register-spec.md` — pas la doc officielle Deye (fabricant et moteurs de recherche inaccessibles depuis cette session), mais une source validée sur le matériel exact.

---

*Établi le 10 août 2026. À mettre à jour au fil de l'intégration.*
