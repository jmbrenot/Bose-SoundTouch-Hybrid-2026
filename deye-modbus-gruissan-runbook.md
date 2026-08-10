# Intégration Modbus onduleur DEYE — Gruissan

Suivi de projet — intégration de l'onduleur hybride **DEYE SUN-6K-SG05LP1-EU-AM2-P** dans Home Assistant, via une passerelle RS485→Modbus TCP. Le Growatt est déjà intégré via SolaX Modbus (HACS) ; ce document couvre l'ajout du DEYE en complément. Établi le 10 août 2026.

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
   - `slave_id` : `1` (valeur par défaut de la doc — à confirmer si l'onduleur a un ID Modbus différent)
   - `model` : **`deye_hybrid`**
5. Envoyer — les entités apparaissent automatiquement.

## Checklist

- [x] Passerelle identifiée : USR-TCP232-410S, IP `192.168.1.170` (confirmé le 10 août 2026)
- [x] Passerelle configurée côté RS485/Socket A : TCP Server + ModbusTCP, port 502, 9600-8-N-1, Modbus Poll activé (constaté déjà en place le 10 août 2026)
- [ ] Confirmer le brochage RS485 exact du SUN-6K-SG05LP1-EU-AM2-P (étiquette/manuel, peut différer du SG05LP1-EU-SM2-P documenté)
- [ ] Câbler l'onduleur → passerelle (A/B)
- [ ] Fixer l'IP de la passerelle (`192.168.1.170`) côté routeur/DHCP pour éviter qu'elle change
- [ ] Installer `comdif/ha-solarmodbus` dans `custom_components`
- [ ] Ajouter l'intégration via l'UI (mode TCP, host `192.168.1.170`, port 502, slave_id, modèle `deye_hybrid`)
- [ ] Vérifier que les entités créées correspondent à des valeurs cohérentes (SOC, puissance, tension réseau, etc.)
- [ ] Une fois validé : ajouter les badges de statut ONDULEUR sur le synoptique Kilovac (voir `kilovac-batterie-gruissan-runbook.md`, checklist item 9)

## Références

- Dépôt : `github.com/comdif/ha-solarmodbus`
- Doc de configuration (PDF fourni dans le dépôt) : `solarmodbus.pdf` — utilise le Deye SG05LP1-EU-SM2-P comme modèle de référence pour tout le guide de câblage et configuration.

## Journal

### 10 août 2026
- Passerelle réelle identifiée : USR-TCP232-410S (pas l'EBYTE NA111 de la doc de référence), IP `192.168.1.170`, firmware V8.0.12.
- Onglet RS485 vérifié : Socket A déjà configuré correctement (TCP Server + ModbusTCP, port 502, 9600-8-N-1, Modbus Poll activé). Le `PORT Status: RS232` vu sur la page Current Status était un sélecteur d'affichage, pas le mode actif — fausse alerte corrigée.
- Reste : câblage physique onduleur → passerelle, vérification du brochage RJ45 exact du modèle AM2, IP fixe, puis installation/config de `comdif/ha-solarmodbus` côté HA.

---

*Établi le 10 août 2026. À mettre à jour au fil de l'intégration.*
