# Hayward pool heater MQTT bridge

This is a little project I've been working for a while now. I'm the owner of a Hayward pool heater (*Trevium HP55TR, which is also the same exact model of Hayward energy line pro* )
This heat pump use a controller called **PC1000**.

Following pool heaters have been tested:
- **Trevium HP55TR** heat pump that is using a **PC1000** controller.
- **Hayward energy line pro** heat pump that is using a **PC1000** controller.
- **MONO 50 Basic** heat pump that is using a controller named **CC203** (if the online manual is correct)
- **Majestic** heat pump (Hayward white label) that is using a **PC1001** controller.
- **CPAC111** heat pump (Hayward) that is using a **PC1001** controller.
- **OASIS X19/24** heat pumps (Oasis) that is using a **PC1001** controller. These are common in Australia.
I have decoded the data using a small logic sniffer.

The last version of the sketch can now **received current parameters** and **send command** to the heatpump.
This version of the sketch is working on a **wemos d1 mini** (using the arduino IDE with **arduino core** installed)

Shematic

You have to connect the `NET` pin of the PC1000 controller to your `D5` pin of the wemos d1 via a *bidirectional* level shifter, and connect the PC1000 `GND` to the `GND` of your esp8266.
The 5v <-> 3.3v level shifter is mandatory because the esp8266 is not 5V tolerant, and the heatpump **controller is not working with 3.3v**.

On the **PC1001** board, you can connect the Wemos on `+5V` and `GND` using the connector **CN16**, then connect `NET`, `+5V` and `GND` to a *bi-directional logic level converter* (high voltage side) and, on the other side (low voltage side), connect the Wemos `+3.3V`, `GND` and `D5`. 

**MQTT topics**

Data will be published on your MQTT server every few seconds using this topics:

- `pool/power`  (true / false)
- `pool/mode` (heat / cool)
- `pool/automatic_mode` (true / false) Automatic = heat or cold according to the programmed temp and the out temperature
- `pool/temp_in`  (temperature `in` in celcius)
- `pool/temp_out`  (temperature `out` in celcius)
- `pool/temp_prog`  (programmed temperature in celcius)
- `pool/wifi_rssi`  (Wifi received Signal Strength Indication)

You will be able to change the settings via this topics:

- `pool/set_power_on`  (NULL msg)
- `pool/set_power_off` (NULL msg)
- `pool/set_mode` (HEAT/COOL/AUTO)
- `pool/set_temp`  (temperature in celcius. You could set half degree. Ex: 27.5)

---------

![Demo with a node red UI flow](https://raw.githubusercontent.com/njanik/hayward-pool-heater-mqtt/master/20200523_111808.jpg)



Special thx to the french arduino community, and especially to plode.
Also to this github users: @jruibarroso and @marcphilibert for adding temperature in and wifi rssi data.

[Whole reverse engineering topic (in french)](https://forum.arduino.cc/index.php?topic=258722.0)


**Home Assistant Usage**
```
mqtt:
  binary_sensor:
    - name: Pool Heater Status
      state_topic: pool/power
      payload_on: 1
      payload_off: 0
  sensor:
    - name: Pool Heater Temp In
      state_topic: pool/temp_in
      device_class: temperature
      unit_of_measurement: "°C"
    - name: Pool Heater Temp Out
      state_topic: pool/temp_out
      device_class: temperature
      unit_of_measurement: "°C"
    - name: Pool Heater Target
      state_topic: pool/temp_prog
      device_class: temperature
      unit_of_measurement: "°C"
    - name: Pool Heater Mode
      state_topic: pool/mode
    - name: Pool Heater Wifi
      state_topic: pool/wifi_rssi
      device_class: signal_strength
      unit_of_measurement: "rssi"
  number:
    - name: Pool Heater Target
      command_topic: pool/set_temp
      min: 18
      max: 33
      state_topic: pool/temp_prog
      device_class: temperature
      unit_of_measurement: "°C"
      mode: slider
      step: 0.5

```

The power switch itself is best controlled via a template switch

```
switch:
  - platform: template
    switches:
      pool_heater:
        friendly_name: Pool Heater
        value_template: "{{ is_state('binary_sensor.pool_heater_status', 'on') }}"
        turn_on:
          service: mqtt.publish
          data:
            topic: pool/set_power_on
        turn_off:
          service: mqtt.publish
          data:
            topic: pool/set_power_off
```

