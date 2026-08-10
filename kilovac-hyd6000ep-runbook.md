# Kilovac EV200 — Contacteur batterie LiFePO4 → SOFAR HYD6000EP

Suivi de projet — commande d'un contacteur TE Kilovac EV200AAANA entre le + batterie LiFePO4 48 V et la borne BAT+ de l'onduleur hybride SOFAR HYD6000EP, piloté depuis Home Assistant avec sécurité matérielle JK-BMS. Repris et complété le 10 août 2026 à partir du contexte du 12 juillet 2026.

## Topologie électrique

| Élément | Rôle |
|---|---|
| Batterie LiFePO4 48 V | Source à isoler |
| TE Kilovac EV200AAANA | Contacteur haute puissance, bobine 12 V DC |
| SOFAR HYD6000EP | Onduleur hybride, borne BAT+ |
| JK-BMS (DRY2) | Sécurité en série sur la commande bobine |
| Shelly Plus 1 (192.168.1.54) | Commande HA de la bobine (relais sec) |
| Home Assistant | 192.168.1.42 |

## Décisions retenues

- Commande directe du JK-BMS (DRY1/DRY2) depuis ESPHome **abandonnée** : les triggers `emergency`, `emergency_button_trigger`, `dry_contact_alarm_intermittent` du composant `syssi/esphome-jk-bms` n'ont pas fermé les contacts ; les valeurs `DRY Trigger Val` / `DRY Release Val` ne sont pas exposées par ce composant.
- **Shelly Plus 1** retenu comme commande HA de la bobine EV200. **Pas de Shelly Plus 1PM, pas de SONOFF MINIR4** (écartés le 12 juillet 2026).
- JK-BMS DRY2 conservé **en série**, comme sécurité matérielle indépendante de HA.
- Intégration HA du Shelly Plus 1 : **intégration native Shelly** (découverte locale mDNS), pas de commande via MQTT — le MQTT du Shelly n'est pas nécessaire pour ce device côté HA. (Décidé le 10 août 2026.)

## Câblage cible

```text
Alimentation 12 V DC +
  -> fusible
  -> Shelly Plus 1 borne I

Shelly Plus 1 borne O
  -> JK-BMS DRY2 borne COM2

JK-BMS DRY2 borne S2
  -> fil rouge bobine Kilovac EV200

Fil noir bobine Kilovac EV200
  -> alimentation 12 V DC - / 0 V
```

## État Home Assistant à conserver tant que la chaîne n'est pas validée

- `input_boolean.kilovac_hyd6000ep_autorisation` : **OFF**
- Ne pas lancer `script.kilovac_hyd6000ep_fermer_apres_precharge`
- Ne pas lancer `script.kilovac_hyd6000ep_reboot_sofar`

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

## Checklist de reprise

- [x] 1. Intégrer le Shelly Plus 1 dans HA (intégration native, découverte mDNS depuis `192.168.1.54`)
- [ ] 2. Identifier son entité `switch.*` et la renommer clairement (ex. `switch.kilovac_contacteur`)
- [ ] 3. Régler l'état au démarrage du relais sur **OFF** (paramètre Shelly « Switch on power up » → Always OFF, pas « Restore last state ») — critique pour ne pas refermer le contacteur après une coupure secteur
- [ ] 4. Tester les I/O du Shelly au multimètre (borne I / borne O), sans la bobine
- [ ] 5. Tester le 12 V commuté par le Shelly, toujours sans bobine
- [ ] 6. Ajouter le JK-BMS DRY2 en série et retester toute la chaîne
- [ ] 7. Modifier `script.kilovac_hyd6000ep_fermer_apres_precharge` et `script.kilovac_hyd6000ep_reboot_sofar` pour piloter `switch.<shelly>` au lieu de l'ancienne cible JK-BMS directe
- [ ] 8. Ne raccorder la bobine EV200 qu'après validation complète des étapes précédentes

## Pièges déjà rencontrés

> **`syssi/esphome-jk-bms` n'expose pas tout.** Les triggers DRY1/DRY2 existent dans le composant, mais pas les valeurs `DRY Trigger Val` / `DRY Release Val` nécessaires pour piloter la sortie sèche de façon fiable. D'où l'abandon de la commande directe JK-BMS au profit du Shelly Plus 1.

## Références

- Document complet (poste HA) : `/config/backup_toolkit/KILOVAC_HYD6000EP.md`
- Sauvegarde HA : `/config/backups/codex-ha-config/ha-config-20260712-192023.zip`

---

*Établi le 10 août 2026 à partir du contexte de reprise du 12 juillet 2026. À mettre à jour à chaque étape de la checklist.*
