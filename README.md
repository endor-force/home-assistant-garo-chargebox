# Garo Chargebox in Home Assistant
Details for setting up Garo Chargebox in Homeassistant

I set up this integration purely with the rest and curl a long time back and thought i would share it with you.
Texts are in Swedish, i think you will figure it out :)

![image|311x500](content/lovelace-screenshot.png) 

It is using the following parts:

* Command line switch to control availability on/off for the chargebox.. if you don't trust your neighbours then you can schedule or automate the availability of charging from the box.
* Rest sensor to collect the data from the box as attributes (one call to collect all stats)
* Template sensors to calculate and make the collected data representable and enumerate statuses.
* Lovelace cards with dynamic image that changes color based on status.
* Utility sensors that collect and calculate stats for usage per month and day..


*Note:* for chargebox with software version earlier than v1.3.1 you need to format the urls differently, update of the box is recommended:
```
http://192.168.xxx.yyy:2222/rest/chargebox/mode
http://192.168.xxx.yyy:2222/rest/chargebox/status
```

Switch to control the state/availability for the box (update with your IP or hostname accordingly): 
Below format is confirmed for v 1.3.1 and later.

```
switch:
  - platform: command_line
    switches:
#Laddbox mode
      laddbox_mode:
          command_on: '/usr/bin/curl -X POST http://192.168.xxx.yyy:8080/servlet/rest/chargebox/mode/ALWAYS_ON'
          command_off: '/usr/bin/curl -X POST http://192.168.xxx.yyy:8080/servlet/rest/chargebox/mode/ALWAYS_OFF'
          command_state: '/usr/bin/curl -X GET http://192.168.xxx.yyy:8080/servlet/rest/chargebox/status'
          value_template: '{{ value_json.mode == "ALWAYS_ON" }}'
```

The sensor with attributes:

```
sensor:
  - platform: rest
    name: Laddbox
    resource: http://192.168.xxx.yyy:8080/servlet/rest/chargebox/status
    json_attributes:
        - connector
        - mode
        - currentLimit
        - pilotLevel
        - powerMode
        - currentChargingCurrent
        - currentChargingPower
        - accSessionEnergy
        - latestReading
        - currentTemperature
        - sessionStartValue
        - nrOfPhases
    value_template: '{{ value_json.connector }}'
```

Template sensors to calculate and enumerate statuses:
```
sensor:
  - platform: template
    sensors:
      #Laddbox templates
      laddbox_chargingcurrent:
        friendly_name: 'Aktuell laddström'
        unit_of_measurement: 'A'
        value_template: >
          {% set state = state_attr('sensor.laddbox', 'currentChargingCurrent')/1000 | round(1) %}
          {% if state >= 0 %}
            {{state}}
          {% else %}
           0,0
          {% endif %}
      laddbox_chargingpower:
        friendly_name: 'Aktuell förbrukning'
        unit_of_measurement: 'kW'
        value_template: >
          {% set state = state_attr('sensor.laddbox', 'currentChargingPower')/1000 | round(1) %}
          {% if state >= 0 %}
            {{state}}
          {% else %}
           0,0
          {% endif %}
      laddbox_sessionenergy:
        friendly_name: 'Laddat sedan anslutning'
        value_template: '{{ state_attr("sensor.laddbox", "accSessionEnergy")/1000 | round(1) }}'
        unit_of_measurement: 'kWh'
      laddbox_totalconsumption:
        friendly_name: 'Total förbrukning'
        value_template: '{{ state_attr("sensor.laddbox", "latestReading")/1000 | round(1) }}'
        unit_of_measurement: 'kWh'
      laddbox_temp:
        friendly_name: 'Laddbox temp'
        value_template: '{{ state_attr("sensor.laddbox", "currentTemperature") }}'
        unit_of_measurement: '°C'
      laddbox_chargerstatus:
        friendly_name: 'Laddbox status'
        value_template: >
          {# Enumerate and give the chargebox translation texts. #}
          {%-
            set box_status = {
              'CHARGING_FINISHED' : 'Klar',
              'CHARGING_CANCELLED' : 'Avbruten',
              'CHARGING_PAUSED' : 'Pausad av lastbalanserare',
              'NOT_CONNECTED' : 'Ej ansluten',
              'CONNECTED' : 'Ansluten',
              'SEARCH_COMM' : 'Ansluten - söker',
              'CHARGING' : 'Laddar',
              'RCD_FAULT' : 'Skyddsbrytare utlöst',
              'OVERHEAT' : 'Överhettad, begränsad',
              'CRITICAL_TEMPERATURE' : 'Kritisk temperatur',
              'INITIALIZATION' : 'Initierar',
              'CABLE_FAULT' : 'Kabelfel',
              'LOCK_FAULT' : 'Låsfel',
              'CONTACTOR_FAULT' : 'Kontaktfel',
              'VENT_FAULT' : 'Ventileringsfel',
              'DC_ERROR' : 'DC-fel',
              'DISABLED' : 'Inaktiverad',
              'UNAVAILABLE' : 'Laddning pausad (Master)',
              'REMOTE_DISABLED' : 'Externt deaktiverad',
              'DC_HARDWARE' : 'DC hårdvarufel',
              'CP_FAULT' : 'Kommunikationsfel (CP-signal)',
              'CP_SHORTED' : 'Kommunikationsfel (CP-kortsluten)'
            }
          -%}

          {# Get the box status from the attribute connector #}
          {% set status = state_attr('sensor.laddbox','connector') %}

          {#- Check if the status is translated and return the translated text.
          If a translation is not found then status text will be raw status text. -#}
          {% if box_status.get(status) %}
            {{box_status.get(status)}}
          {% else %}
            {{status}}
          {% endif %}
```

Utility meters:
```
utility_meter:
  laddning_bil_manad:
    source: sensor.laddbox_totalconsumption
    cycle: monthly
  laddning_bil_daglig:
    source: sensor.laddbox_totalconsumption
    cycle: daily
```

Lovelace page:

```
- title: Car
    id: car
    icon: mdi:car
    panel: false
    cards:
      - type: vertical-stack
        cards:
          - type: picture-entity
            name: Laddbox
            entity: sensor.laddbox_chargerstatus
            state_image:
                "Klar": /local/laddbox_gron.png
                "Avbruten": /local/laddbox_gron.png
                "Pausad av lastbalanserare": /local/laddbox_gron.png
                "Ej ansluten": /local/laddbox_gron.png
                "Ansluten": /local/laddbox_gron.png
                "Ansluten - söker": /local/laddbox_gron.png
                "Laddar": /local/laddbox_bla.png
                "Skyddsbrytare utlöst": /local/laddbox_rod.png
                "Överhettad, begränsad": /local/laddbox_bla.png
                "Kritisk temperatur": /local/laddbox_rod.png
                "Initierar": /local/laddbox_gron.png
                "Kabelfel": /local/laddbox_rod.png
                "Låsfel": /local/laddbox_rod.png
                "Kontaktfel": /local/laddbox_rod.png
                "Ventileringsfel": /local/laddbox_rod.png
                "DC-fel": /local/laddbox_rod.png
                "Inaktiverad": /local/laddbox_off.png
                "Laddning pausad (Master)" : /local/laddbox_bla.png
                "Externt deaktiverad" : /local/laddbox_bla.png
                "DC hårdvarufel" : /local/laddbox_rod.png
                "Kommunikationsfel (CP-signal)" : /local/laddbox_rod.png
                "Kommunikationsfel (CP-kortsluten)" : /local/laddbox_rod.png
                "None": /local/laddbox_off.png
          - type: entities
            entities:
              - entity: sensor.laddbox_chargingpower
                name: Aktuell förbrukning
                icon: mdi:resistor
              - entity: sensor.laddbox_chargingcurrent
                name: Aktuell laddström
                icon: mdi:flash-circle
              - entity: sensor.laddbox_sessionenergy
                name: Laddat sedan anslutning
                icon: mdi:clock-start
              - entity: sensor.laddbox_temp
                name: Temperatur
                icon: mdi:thermometer
              - entity: sensor.laddbox_totalconsumption
                name: Total förbrukning
                icon: mdi:gauge
              - entity: switch.laddbox_mode
                name: Laddbox tillgänglig
          - type: entities
            entities:
              - entity: sensor.laddning_bil_daglig
                name: Förbrukat idag
              - entity: sensor.laddning_bil_manad
                name: Laddat denna månad
```
