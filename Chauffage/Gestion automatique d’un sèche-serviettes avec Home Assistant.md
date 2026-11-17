# 🚿 Gestion automatique d’un sèche-serviettes avec Home Assistant

> Régulation de la température, programmation Confort/Éco, gestion de l’humidité et arrêt automatique à l’ouverture des fenêtres.

## 📑 Sommaire

- [Présentation](#-présentation)
- [Fonctionnement](#-fonctionnement)
- [Capteurs et équipements](#-capteurs-et-équipements)
- [Installation](#-installation)
- [Code 1 — Configuration des capteurs et de la régulation](#-code-1--configuration-des-capteurs-et-de-la-régulation)
- [Code 2 — Automatisation d’allumage](#-code-2--automatisation-dallumage)
- [Code 3 — Automatisation d’extinction](#-code-3--automatisation-dextinction)
- [Personnalisation et vérifications](#-personnalisation-et-vérifications)

## 🏠 Présentation

Ce projet permet de piloter un sèche-serviettes électrique avec **Home Assistant** en fonction de la température mesurée dans la salle de bain, des horaires, du taux d’humidité et de deux contacts de fenêtre.

**Tout le projet est documenté dans ce seul `README.md`.** Les blocs YAML ci-dessous sont à copier dans Home Assistant aux emplacements indiqués ; il n’est pas nécessaire de publier un fichier `.yaml` distinct sur GitHub.

## ⚙️ Fonctionnement

### 🕒 Horaires

| Jours | Mode Confort | Mode Éco |
|---|---|---|
| Lundi à vendredi | 06:00–09:00 et 17:00–20:30 | En dehors de ces plages |
| Samedi et dimanche | 08:00–11:00 et 17:00–20:30 | En dehors de ces plages |

### 🌡️ Régulation de la température

| Situation | Démarrage de la demande de chauffage | Arrêt de la demande |
|---|---|---|
| Mode Confort | Sous **20 °C** | À partir de **22 °C** |
| Mode Éco | Sous **17 °C** | À partir de **19 °C** |
| Mode Éco avec humidité élevée | Sous **22 °C** | À partir de **22 °C** |

Les modes Confort et Éco utilisent une **hystérésis** : une fois le chauffage demandé, la demande reste active jusqu’au seuil haut. En mode Éco avec humidité élevée, le code demande directement le chauffage tant que la température reste inférieure à 22 °C.

### 💧 Humidité

- Le capteur d’humidité s’active au-dessus de **70 %**.
- Il conserve son état actif tant que l’humidité reste au moins à **65 %**.
- Une fois la condition de désactivation atteinte, un délai de **30 minutes** est appliqué avant son extinction.
- Si le logement est en mode Éco et que le capteur d’humidité est actif, le chauffage peut fonctionner jusqu’à **22 °C**.

### 🪟 Sécurité des fenêtres

La demande de chauffage est désactivée dès qu’**au moins une des deux fenêtres** n’est plus détectée fermée. L’automatisation d’extinction coupe alors le relais.

## 🔌 Capteurs et équipements

Les identifiants ci-dessous correspondent à l’installation d’origine. **Adaptez-les à votre installation** avant utilisation.

| Rôle | Entité Home Assistant |
|---|---|
| Température ambiante | `sensor.a4c138fcf2e6_ble_temperature_a4c138fcf2e6` |
| Humidité ambiante | `sensor.ble_humidity_a4c138fcf2e6` |
| Fenêtre gauche | `binary_sensor.capteur_de_fenetre_salle_de_bain_gauche_contact` |
| Fenêtre droite | `binary_sensor.capteur_de_fenetre_salle_de_bain_droite_contact` |
| Relais du sèche-serviettes | `switch.switch_salle_de_bain_seche_serviette` |
| Mode calculé | `sensor.chauffage_salle_de_bain_mode` |
| Humidité élevée calculée | `binary_sensor.salle_de_bain_humidite` |
| Demande de chauffage calculée | `binary_sensor.seche_serviettes_automatique` |

## 🛠️ Installation

1. Vérifiez que vos capteurs de température, d’humidité et de fenêtre ainsi que le relais fonctionnent dans Home Assistant.
2. Copiez le **code 1** dans votre configuration Home Assistant. Il est structuré pour être utilisé comme contenu d’un package, par exemple dans `/config/packages/salle_de_bain_chauffage.yaml`, si les packages sont activés. **Ce fichier est créé sur votre instance Home Assistant, pas dans le dépôt GitHub.**
3. Vérifiez la configuration YAML, puis rechargez les entités concernées ou redémarrez Home Assistant selon votre installation.
4. Dans **Paramètres → Automatisations et scènes**, créez une automatisation et collez le **code 2** dans l’éditeur YAML.
5. Créez une seconde automatisation et collez le **code 3** dans son éditeur YAML.
6. Testez le changement de mode, l’ouverture de chaque fenêtre et les seuils de température et d’humidité.

## 📄 Code 1 — Configuration des capteurs et de la régulation

Ce code correspond à la configuration principale, avec **20/22 °C pour le mode Confort**.

```yaml
# ==========================================================
# SALLE DE BAIN - GESTION AUTOMATIQUE DU SECHE-SERVIETTES
# ==========================================================

template:
  # --------------------------------------------------------
  # 1. GESTION DES HORAIRES CONFORT / ECO
  # --------------------------------------------------------
  - sensor:
      - name: "Chauffage salle de bain mode"
        unique_id: chauffage_salle_de_bain_mode
        icon: mdi:shower
        state: >
          {% set heure = now().strftime('%H:%M') %}
          {% if now().isoweekday() in [6, 7] %}
            {% if ("08:00" <= heure < "11:00") or ("17:00" <= heure < "20:30") %}
              Mode Confort
            {% else %}
              Mode Eco
            {% endif %}
          {% else %}
            {% if ("06:00" <= heure < "09:00") or ("17:00" <= heure < "20:30") %}
              Mode Confort
            {% else %}
              Mode Eco
            {% endif %}
          {% endif %}

  # --------------------------------------------------------
  # 2. DETECTION DE L'HUMIDITE
  # --------------------------------------------------------
  - binary_sensor:
      - name: "Salle de bain humidité"
        unique_id: binary_salle_de_bain_humidite
        icon: mdi:water-percent-alert

        availability: >
          {{ is_number(states('sensor.ble_humidity_a4c138fcf2e6')) }}

        state: >
          {% set humidite = states('sensor.ble_humidity_a4c138fcf2e6') | float(none) %}
          {% set actif = this.state == 'on' %}
          {{ humidite is not none and (humidite > 70 or (humidite >= 65 and actif)) }}

        delay_off: "00:30:00"

  # --------------------------------------------------------
  # 3. REGULATION AUTOMATIQUE DU SECHE-SERVIETTES
  # --------------------------------------------------------
  - binary_sensor:
      - name: "Sèche serviettes automatique"
        unique_id: binary_seche_serviette_automatique
        icon: mdi:shower

        state: >
          {# Température de la pièce #}
          {% set temperature = states('sensor.a4c138fcf2e6_ble_temperature_a4c138fcf2e6') | float(none) %}

          {# Seuils Confort / Eco #}
          {% set temp_confort_min = 20.0 %}
          {% set temp_confort_max = 22.0 %}
          {% set temp_eco_min = 17.0 %}
          {% set temp_eco_max = 19.0 %}

          {# Vérification des fenêtres #}
          {% set fenetre_gauche = is_state('binary_sensor.capteur_de_fenetre_salle_de_bain_gauche_contact', 'off') %}
          {% set fenetre_droite = is_state('binary_sensor.capteur_de_fenetre_salle_de_bain_droite_contact', 'off') %}

          {# Mode programmé #}
          {% set mode_confort = is_state('sensor.chauffage_salle_de_bain_mode', 'Mode Confort') %}
          {% set mode_eco = is_state('sensor.chauffage_salle_de_bain_mode', 'Mode Eco') %}

          {# Activation Confort en cas d'humidité élevée en Eco #}
          {% set humidite_active = is_state('binary_sensor.salle_de_bain_humidite', 'on') %}
          {% set confort_actif = mode_confort or (mode_eco and humidite_active) %}

          {# Hystérésis #}
          {% set etat_precedent = this.state == 'on' %}

          {% if not fenetre_gauche or not fenetre_droite %}
            false
          {% elif temperature is none %}
            false
          {% elif mode_eco and humidite_active %}
            {{ temperature < temp_confort_max }}
          {% elif mode_confort %}
            {{ temperature < temp_confort_min or
              (temperature < temp_confort_max and etat_precedent) }}
          {% elif mode_eco %}
            {{ temperature < temp_eco_min or
              (temperature < temp_eco_max and etat_precedent) }}
          {% else %}
            false
          {% endif %}
```

## 🔥 Code 2 — Automatisation d’allumage

Cette automatisation allume le relais uniquement lorsque la demande de chauffage est active, que les deux fenêtres sont fermées et que le relais est éteint. Elle vérifie également les conditions chaque minute et au démarrage de Home Assistant.

```yaml
alias: 'Salle de bain : Allumer sèche-serviettes'
description: ''
triggers:
  - trigger: state
    entity_id: binary_sensor.seche_serviettes_automatique
    to: 'on'
  - trigger: state
    entity_id:
      - binary_sensor.capteur_de_fenetre_salle_de_bain_gauche_contact
      - binary_sensor.capteur_de_fenetre_salle_de_bain_droite_contact
    to: 'off'
  - trigger: homeassistant
    event: start
  - trigger: time_pattern
    minutes: /1
conditions:
  - condition: state
    entity_id: binary_sensor.seche_serviettes_automatique
    state: 'on'
  - condition: state
    entity_id: binary_sensor.capteur_de_fenetre_salle_de_bain_gauche_contact
    state: 'off'
  - condition: state
    entity_id: binary_sensor.capteur_de_fenetre_salle_de_bain_droite_contact
    state: 'off'
  - condition: state
    entity_id: switch.switch_salle_de_bain_seche_serviette
    state: 'off'
actions:
  - action: switch.turn_on
    target:
      entity_id: switch.switch_salle_de_bain_seche_serviette
mode: single
```

## 🛑 Code 3 — Automatisation d’extinction

Cette automatisation éteint le relais lorsque la demande de chauffage passe à `off`, lorsqu’une fenêtre s’ouvre ou au démarrage de Home Assistant si les conditions de chauffage ne sont pas réunies.

```yaml
alias: 'Salle de bain : Eteindre sèche-serviettes'
description: ''
triggers:
  - trigger: state
    entity_id: binary_sensor.seche_serviettes_automatique
    to: 'off'
  - trigger: state
    entity_id:
      - binary_sensor.capteur_de_fenetre_salle_de_bain_gauche_contact
      - binary_sensor.capteur_de_fenetre_salle_de_bain_droite_contact
    to: 'on'
  - trigger: homeassistant
    event: start
conditions:
  - condition: template
    value_template: |
      {{ not is_state('binary_sensor.seche_serviettes_automatique', 'on')
         or not is_state('binary_sensor.capteur_de_fenetre_salle_de_bain_gauche_contact', 'off')
         or not is_state('binary_sensor.capteur_de_fenetre_salle_de_bain_droite_contact', 'off') }}
actions:
  - action: switch.turn_off
    target:
      entity_id: switch.switch_salle_de_bain_seche_serviette
mode: single
```

## 🔎 Personnalisation et vérifications

- **Entités :** remplacez les identifiants des capteurs et du relais par ceux de votre installation.
- **Températures :** ajustez les quatre variables `temp_confort_min`, `temp_confort_max`, `temp_eco_min` et `temp_eco_max`.
- **Horaires :** modifiez les plages horaires dans le premier capteur `sensor`.
- **Fenêtres :** le code suppose que `off` signifie **fermée** et `on` signifie **ouverte**.
- **Humidité :** le seuil de démarrage est strictement supérieur à 70 %, et la temporisation de désactivation est de 30 minutes.
- **Attention :** la branche Éco + humidité compare directement la température à 22 °C, sans hystérésis spécifique autour de ce seuil.

> **Sécurité :** vérifiez que le relais et son câblage sont adaptés à la puissance du sèche-serviettes. Une automatisation logicielle ne remplace pas les protections électriques et thermiques de l’appareil.
