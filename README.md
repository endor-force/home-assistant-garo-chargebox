# Garo Chargebox in Home Assistant
An integration with Garo Chargebox without custom components.

Some texts are in Swedish, i think you will figure it out :)

![image](content/lovelace-screenshot.png)

It consists of the following parts:

* Rest command switches with dropdown to control availability on/off/schedule for the chargebox.
* Slider to control max charge current.
* Rest sensor to collect the data from the box as attributes (one call to collect all stats)
* Template sensors to calculate and make the collected data representable and enumerate statuses.
* Lovelace cards with dynamic image that changes color based on status.
* Utility sensors that collect and calculate stats for usage per month and day..


### How to add to your Home Assistant:

1. Download and put `www` and `garo_chargebox` in to your home assistant `config` folder.
2. Update the file `garo_chargebox/pkg_garo_chargebox.yaml` with the ip or hostname of your chargebox.
3. Verify the max charge current in your box. (You can see it in the main rest sensor attribute "switchCurrentLimit")
4. Add below in `configuration.yaml` in `homeassistant` `packages` area as below example:


```
homeassistant:

    packages:

      ### GARO CHARGEBOX ###
      garo_chargebox: !include garo_chargebox/pkg_garo_chargebox.yaml

```


4. Add the view or cards in to existing view in `ui-lovelace.yaml` if you are using Yaml mode:


    a) Add in views section (placement will determine order):
    ```
    views:
      - !include garo_chargebox/lovelace_view.yaml
    ```
    b) Or add the cards in existing view (example):
    ```
   - title: Car
     id: car
     icon: mdi:car
     panel: false
     cards:
       - !include garo_chargebox/lovelace_cards.yaml
    ```



*Note:* for chargebox with software version earlier than v1.3.1 you need to format the urls differently, update of the box is recommended:
```
http://192.168.xxx.yyy:2222/rest/chargebox/mode
http://192.168.xxx.yyy:2222/rest/chargebox/status
```
