# Kilovac EV200 — Contacteur batterie LiFePO4 → SOFAR HYD6000EP

Suivi de projet — commande d'un contacteur TE Kilovac EV200AAANA entre le + batterie LiFePO4 48 V et la borne BAT+ de l'onduleur hybride SOFAR HYD6000EP, piloté depuis Home Assistant. Repris et complété le 10 août 2026 à partir du contexte du 12 juillet 2026 ; pilotage aligné le même jour sur le schéma retenu à Auzeville (Shelly seul, sans JK-BMS dans la boucle de commande).

## Topologie électrique

| Élément | Rôle |
|---|---|
| Batterie LiFePO4 48 V | Source à isoler |
| TE Kilovac EV200AAANA | Contacteur haute puissance, bobine 12 V DC |
| SOFAR HYD6000EP | Onduleur hybride, borne BAT+ |
| Shelly Plus 1 (192.168.1.54) | Commande HA de la bobine (relais sec) — `switch.garage_shellyplus1_kilovac` |
| Home Assistant | 192.168.1.42 |

## Décisions retenues

- Commande directe du JK-BMS (DRY1/DRY2) depuis ESPHome **abandonnée** : les triggers `emergency`, `emergency_button_trigger`, `dry_contact_alarm_intermittent` du composant `syssi/esphome-jk-bms` n'ont pas fermé les contacts ; les valeurs `DRY Trigger Val` / `DRY Release Val` ne sont pas exposées par ce composant.
- **Shelly Plus 1** retenu comme commande HA de la bobine EV200. **Pas de Shelly Plus 1PM, pas de SONOFF MINIR4** (écartés le 12 juillet 2026).
- **JK-BMS DRY2 totalement retiré de la boucle de commande (câblage et logique) le 10 août 2026** : le contact sec ne fonctionne pas de façon fiable. **Seul le Shelly Plus 1 pilote directement la bobine du Kilovac** — pas de sécurité matérielle JK-BMS en série, pas de condition logicielle basée sur le JK-BMS.
- Intégration HA du Shelly Plus 1 : **intégration native Shelly** (découverte locale mDNS), pas de commande via MQTT — le MQTT du Shelly n'est pas nécessaire pour ce device côté HA. (Décidé le 10 août 2026.)
- Pilotage HA calqué sur le schéma d'Auzeville (`kilovac_batterie_auzeville_*`), repris sous une nouvelle famille d'entités **`kilovac_batterie_gruissan_*`** — sans confirmation automatique par tension (pas de capteur équivalent côté Gruissan) tant que la chaîne complète n'est pas validée physiquement.
- Les anciennes entités `kilovac_hyd6000ep_*` (input_boolean d'autorisation, scripts JK-BMS) sont **obsolètes** et remplacées par `kilovac_batterie_gruissan_*` ; elles peuvent être supprimées de la config HA une fois la nouvelle famille validée.

## Câblage cible

```text
Alimentation 12 V DC +
  -> fusible
  -> Shelly Plus 1 borne I

Shelly Plus 1 borne O
  -> fil rouge bobine Kilovac EV200

Fil noir bobine Kilovac EV200
  -> alimentation 12 V DC - / 0 V
```

Le JK-BMS DRY2 n'intervient plus dans ce câblage.

## État Home Assistant à conserver tant que la chaîne n'est pas validée

- `input_boolean.kilovac_batterie_gruissan_autorisation` : **OFF**
- Ne pas lancer `script.kilovac_batterie_gruissan_fermer`
- Les anciennes entités `input_boolean.kilovac_hyd6000ep_autorisation` / `script.kilovac_hyd6000ep_*` ne sont plus utilisées — ne pas les lancer non plus tant qu'elles n'ont pas été nettoyées de la config.

## Pilotage Home Assistant (famille `kilovac_batterie_gruissan_*`)

Calqué sur le bloc `kilovac_batterie_auzeville_*` du dashboard SCADA Auzeville, simplifié : pas de confirmation par capteur (le JK-BMS n'étant plus dans la chaîne, et aucun capteur de tension côté onduleur n'existe à Gruissan). Vérification physique (multimètre) obligatoire tant que la chaîne n'est pas validée bout en bout.

### Helper à créer

Réglages → Appareils et services → **Aides** → + → *Interrupteur (toggle)* :
- Nom : `Kilovac batterie Gruissan - autorisation`
- Entity ID résultant : `input_boolean.kilovac_batterie_gruissan_autorisation`

### Scripts (`scripts.yaml`, ou Réglages → Automatisations et scènes → Scripts → mode YAML)

```yaml
# CODEX KILOVAC BATTERIE GRUISSAN - DEBUT
kilovac_batterie_gruissan_autoriser_fermeture:
  alias: Kilovac batterie Gruissan - armer la fermeture
  icon: mdi:lock-open-check
  mode: single
  sequence:
    - action: input_boolean.turn_on
      target: {entity_id: input_boolean.kilovac_batterie_gruissan_autorisation}

kilovac_batterie_gruissan_interdire_fermeture:
  alias: Kilovac batterie Gruissan - ouvrir et désarmer
  icon: mdi:lock
  mode: restart
  sequence:
    - action: input_boolean.turn_off
      target: {entity_id: input_boolean.kilovac_batterie_gruissan_autorisation}
    - action: script.kilovac_batterie_gruissan_ouvrir

kilovac_batterie_gruissan_ouvrir:
  alias: Kilovac batterie Gruissan - ouvrir
  icon: mdi:electric-switch
  mode: restart
  sequence:
    - action: switch.turn_off
      target: {entity_id: switch.garage_shellyplus1_kilovac}
    - action: persistent_notification.create
      data:
        title: Kilovac batterie Gruissan
        message: >-
          Commande d'ouverture envoyée au Shelly. Aucune confirmation
          automatique disponible : vérifier physiquement l'état du
          contacteur.

kilovac_batterie_gruissan_fermer:
  alias: Kilovac batterie Gruissan - fermer
  icon: mdi:electric-switch-closed
  mode: single
  sequence:
    - condition: state
      entity_id: input_boolean.kilovac_batterie_gruissan_autorisation
      state: "on"
    - condition: template
      value_template: "{{ states('switch.garage_shellyplus1_kilovac') not in ['unknown', 'unavailable'] }}"
    - action: switch.turn_on
      target: {entity_id: switch.garage_shellyplus1_kilovac}
    - action: persistent_notification.create
      data:
        title: Kilovac batterie Gruissan
        message: >-
          Commande de fermeture envoyée au Shelly. Aucune confirmation
          automatique disponible : vérifier physiquement l'état du
          contacteur avant de considérer la fermeture effective.

kilovac_batterie_gruissan_etat:
  alias: Kilovac batterie Gruissan - afficher état
  icon: mdi:information-outline
  mode: single
  sequence:
    - action: persistent_notification.create
      data:
        title: Kilovac batterie Gruissan - état
        message: >
          Autorisation {{ states('input_boolean.kilovac_batterie_gruissan_autorisation') }},
          Shelly {{ states('switch.garage_shellyplus1_kilovac') }}.
          Aucune confirmation physique automatisée : vérifier au multimètre si besoin.
# CODEX KILOVAC BATTERIE GRUISSAN - FIN
```

### Carte dashboard (Lovelace)

Synoptique : `kilovac-batterie-gruissan-synoptique.svg` (dans ce dépôt) — copié dans `/config/www/` sur l'hôte HA Gruissan le 10 août 2026, référencé via `/local/kilovac-batterie-gruissan-synoptique.svg`.

**Emplacement dans le dashboard réel `SCADA ENERGETIQUE` :** insérer le bloc `vertical-stack` ci-dessous dans la colonne `center` de la vue `scada`, juste après la carte `entities` « Batterie Gruissan JKBMS » et avant le `vertical-stack` de la colonne `right`.

```yaml
- type: vertical-stack
  title: Kilovac batterie Gruissan
  cards:
    - type: picture-elements
      image: /local/kilovac-batterie-gruissan-synoptique.svg
      aspect_ratio: 60%
      elements:
        - type: conditional
          conditions:
            - entity: switch.garage_shellyplus1_kilovac
              state: 'on'
          elements:
            - type: icon
              icon: mdi:circle
              title: Shelly ON
              style: {left: 50%, top: 45%, color: '#18b85a', --mdc-icon-size: 44px, filter: 'drop-shadow(0 2px 4px rgba(0,0,0,.55))'}
            - type: icon
              icon: mdi:toggle-switch
              title: Shelly ON
              style: {left: 50%, top: 45%, color: '#ffffff', --mdc-icon-size: 25px}
        - type: conditional
          conditions:
            - entity: switch.garage_shellyplus1_kilovac
              state: 'off'
          elements:
            - type: icon
              icon: mdi:circle
              title: Shelly OFF
              style: {left: 50%, top: 45%, color: '#e53935', --mdc-icon-size: 44px, filter: 'drop-shadow(0 2px 4px rgba(0,0,0,.55))'}
            - type: icon
              icon: mdi:toggle-switch-off
              title: Shelly OFF
              style: {left: 50%, top: 45%, color: '#ffffff', --mdc-icon-size: 25px}
        - type: conditional
          conditions:
            - entity: switch.garage_shellyplus1_kilovac
              state: unavailable
          elements:
            - type: icon
              icon: mdi:circle
              title: Shelly indisponible
              style: {left: 50%, top: 45%, color: '#e53935', --mdc-icon-size: 44px, filter: 'drop-shadow(0 2px 4px rgba(0,0,0,.55))'}
            - type: icon
              icon: mdi:alert
              title: Shelly indisponible
              style: {left: 50%, top: 45%, color: '#ffffff', --mdc-icon-size: 25px}
        - type: conditional
          conditions:
            - entity: input_boolean.kilovac_batterie_gruissan_autorisation
              state: 'on'
          elements:
            - type: icon
              icon: mdi:circle
              title: Fermeture armée
              style: {left: 80%, top: 88%, color: '#18b85a', --mdc-icon-size: 40px, filter: 'drop-shadow(0 2px 4px rgba(0,0,0,.55))'}
            - type: icon
              icon: mdi:lock-open-check
              title: Fermeture armée
              style: {left: 80%, top: 88%, color: '#ffffff', --mdc-icon-size: 22px}
        - type: conditional
          conditions:
            - entity: input_boolean.kilovac_batterie_gruissan_autorisation
              state: 'off'
          elements:
            - type: icon
              icon: mdi:circle
              title: Fermeture désarmée
              style: {left: 80%, top: 88%, color: '#e53935', --mdc-icon-size: 40px, filter: 'drop-shadow(0 2px 4px rgba(0,0,0,.55))'}
            - type: icon
              icon: mdi:lock
              title: Fermeture désarmée
              style: {left: 80%, top: 88%, color: '#ffffff', --mdc-icon-size: 22px}
    - type: markdown
      content: >-
        **Commande Shelly :** {{ states('switch.garage_shellyplus1_kilovac') }} |
        **Autorisation :** {{ states('input_boolean.kilovac_batterie_gruissan_autorisation') }}


        *Pas de confirmation physique automatisée — le JK-BMS n'est plus utilisé dans cette chaîne. Vérifier manuellement l'état réel du contacteur.*
    - type: grid
      columns: 2
      square: false
      cards:
        - type: button
          name: Armer la fermeture
          icon: mdi:lock-open-check
          tap_action:
            action: call-service
            service: script.turn_on
            target: {entity_id: script.kilovac_batterie_gruissan_autoriser_fermeture}
            confirmation: {text: "Autoriser manuellement une future fermeture du Kilovac ?"}
          card_mod: {style: 'ha-state-icon { color: #20c968 !important; }'}
        - type: button
          name: Fermer le Kilovac
          icon: mdi:electric-switch
          tap_action:
            action: call-service
            service: script.turn_on
            target: {entity_id: script.kilovac_batterie_gruissan_fermer}
            confirmation: {text: "Fermer le Kilovac ? Aucune confirmation automatique : vérifiez physiquement."}
          card_mod: {style: 'ha-state-icon { color: #20c968 !important; }'}
        - type: button
          name: Ouvrir le Kilovac
          icon: mdi:electric-switch-closed
          tap_action:
            action: call-service
            service: script.turn_on
            target: {entity_id: script.kilovac_batterie_gruissan_ouvrir}
          card_mod: {style: 'ha-state-icon { color: #ef5350 !important; }'}
        - type: button
          name: Ouvrir et désarmer
          icon: mdi:lock
          tap_action:
            action: call-service
            service: script.turn_on
            target: {entity_id: script.kilovac_batterie_gruissan_interdire_fermeture}
          card_mod: {style: 'ha-state-icon { color: #ef5350 !important; }'}
```

## Journal

### 12 juillet 2026
- Tests DRY1/DRY2 au multimètre ; DRY2 configuré en `12 - Remote Control` dans l'appli JK-BMS.
- Commande directe ESPHome invalidée (voir Décisions retenues).
- Shelly Plus 1 et alimentation 12 V DC commandés, réception prévue le mardi suivant.

### 10 août 2026
- Shelly Plus 1 relié au réseau (`192.168.1.54`), page MQTT renseignée (prefix `shellyplus1-78ee4cc60a6c`, broker `192.168.1.42:1883`) — **non requis pour l'intégration retenue** ; à désactiver ou laisser inutilisé côté HA.
- Décision confirmée : intégration native Shelly dans HA plutôt que MQTT manuel.
- Reprise de la checklist à l'étape 1.
- Étape 1 faite : Shelly Plus 1 intégré dans HA (intégration native).
- Étape 2 faite : entité identifiée — `switch.garage_shellyplus1_kilovac`.
- Décision : abandon définitif du JK-BMS DRY2 dans la boucle de commande (contact sec non fiable) — seul le Shelly pilote la bobine. Câblage cible simplifié en conséquence.
- Pilotage Home Assistant écrit par analogie avec Auzeville : nouvelle famille d'entités `kilovac_batterie_gruissan_*` (scripts autoriser/interdire/ouvrir/fermer/état), sans confirmation automatique (pas de capteur de tension côté onduleur à Gruissan). Dashboard + synoptique SVG ajoutés au dépôt.

## Checklist de reprise

- [x] 1. Intégrer le Shelly Plus 1 dans HA (intégration native, découverte mDNS depuis `192.168.1.54`)
- [x] 2. Identifier son entité `switch.*` — `switch.garage_shellyplus1_kilovac`
- [ ] 3. Régler l'état au démarrage du relais sur **OFF** (paramètre Shelly « Switch on power up » → Always OFF, pas « Restore last state ») — critique pour ne pas refermer le contacteur après une coupure secteur
- [ ] 4. Tester les I/O du Shelly au multimètre (borne I / borne O), sans la bobine
- [ ] 5. Tester le 12 V commuté par le Shelly, toujours sans bobine
- [x] ~~6. Ajouter le JK-BMS DRY2 en série et retester toute la chaîne~~ — abandonné, DRY2 non fiable, retiré du câblage cible
- [x] 7. Scripts et dashboard Gruissan en place (famille `kilovac_batterie_gruissan_*`, calquée sur Auzeville, sans confirmation automatique) — helper `input_boolean.kilovac_batterie_gruissan_autorisation` restant à créer dans HA
- [ ] 8. Ne raccorder la bobine EV200 qu'après validation complète des étapes précédentes (câblage simplifié : Shelly → bobine directement, sans JK-BMS)

## Pièges déjà rencontrés

> **`syssi/esphome-jk-bms` n'expose pas tout.** Les triggers DRY1/DRY2 existent dans le composant, mais pas les valeurs `DRY Trigger Val` / `DRY Release Val` nécessaires pour piloter la sortie sèche de façon fiable. D'où l'abandon de la commande directe JK-BMS au profit du Shelly Plus 1.

> **Le contact sec JK-BMS reste un point de blocage même hors ESPHome.** Au-delà du problème de pilotage direct, le DRY2 s'est aussi révélé non fiable comme simple sécurité en série une fois câblé : décision du 10 août 2026 de le retirer entièrement de la chaîne plutôt que de continuer à en dépendre.

## Références

- Document complet (poste HA) : `/config/backup_toolkit/KILOVAC_HYD6000EP.md`
- Sauvegarde HA : `/config/backups/codex-ha-config/ha-config-20260712-192023.zip`

---

*Établi le 10 août 2026 à partir du contexte de reprise du 12 juillet 2026. À mettre à jour à chaque étape de la checklist.*
