# Intégration Modbus onduleur DEYE — Gruissan

Suivi de projet — intégration de l'onduleur hybride **DEYE SUN-6K-SG05LP1-EU-AM2-P** dans Home Assistant, via une passerelle RS485→Modbus TCP. Le Growatt est déjà intégré via SolaX Modbus (HACS) ; ce document couvre l'ajout du DEYE en complément. Établi le 10 août 2026.

## Contexte

- Onduleur cible : **DEYE SUN-6K-SG05LP1-EU-AM2-P**.
- Matériel de liaison : passerelle Modbus TCP / RS485 (déjà en possession de l'utilisateur).
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

## Configuration de la passerelle (référence : EBYTE NA111)

Paramètres utilisés dans la doc du projet, à adapter selon le modèle de passerelle réel :

- Work mode : **TCP server**
- Local port : **502**
- Baud rate : **9600**, Data bit 8, Parity NONE, Stop bit 1
- MODBUS TCP to RTU : **Open**
- Target IP / Target port : sans effet en mode TCP server, à ignorer

## Configuration Home Assistant

Entièrement via l'UI, pas de YAML :

1. Réglages → Appareils et services → **Ajouter une intégration**
2. Rechercher **Solarmodbus**
3. Mode : **Modbus TCP (Ethernet / LAN)**
4. Renseigner :
   - `host` : IP de la passerelle
   - `port` : `502`
   - `slave_id` : `1` (valeur par défaut de la doc — à confirmer si l'onduleur a un ID Modbus différent)
   - `model` : **`deye_hybrid`**
5. Envoyer — les entités apparaissent automatiquement.

## Checklist

- [ ] Confirmer le brochage RS485 exact du SUN-6K-SG05LP1-EU-AM2-P (étiquette/manuel, peut différer du SG05LP1-EU-SM2-P documenté)
- [ ] Câbler l'onduleur → passerelle (A/B)
- [ ] Configurer la passerelle (TCP server, port 502, 9600-8-N-1, Modbus TCP to RTU: Open)
- [ ] Installer `comdif/ha-solarmodbus` dans `custom_components`
- [ ] Ajouter l'intégration via l'UI (mode TCP, host, port 502, slave_id, modèle `deye_hybrid`)
- [ ] Vérifier que les entités créées correspondent à des valeurs cohérentes (SOC, puissance, tension réseau, etc.)
- [ ] Une fois validé : ajouter les badges de statut ONDULEUR sur le synoptique Kilovac (voir `kilovac-batterie-gruissan-runbook.md`, checklist item 9)

## Références

- Dépôt : `github.com/comdif/ha-solarmodbus`
- Doc de configuration (PDF fourni dans le dépôt) : `solarmodbus.pdf` — utilise le Deye SG05LP1-EU-SM2-P comme modèle de référence pour tout le guide de câblage et configuration.

---

*Établi le 10 août 2026. À mettre à jour au fil de l'intégration.*
