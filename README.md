# tasmota_rules_collection
Collection of useful Tasmota (firmware for ESP8266 / ESP32 devices) commands and rules.

Here are some useful commands and rules to customize [Tasmota firmware](https://tasmota.github.io/) from the [Tasmota project](https://tasmota.app/) for various use cases with ESP8266/ESP32 based devices (see [list of compatible devices](https://templates.blakadder.com/).

I use the Wemos Mini D1 ([micro-USB](https://www.berrybase.de/d1-mini-esp8266-entwicklungsboard?c=306)/[USB-C](https://www.berrybase.de/d1-mini-esp8266-entwicklungsboard-usb-c?c=306)) a lot, with add-ons like the [Relais Shield](https://www.berrybase.de/relais-shield-fuer-d1-mini?c=306). For difficult Wifi environments, there's a [version with external antenna](https://www.berrybase.de/d1-mini-pro-esp8266-entwicklungsboard-mit-u.fl-anschluss-set-mit-antenne?c=306) available, too.

Timer
=====
To toggle a relay or light off after a specific time being switched on, use PulseTime:
```
PulseTime<x> 1..111ms / ((112..64900)-100)s
```
If the webinterface's timers are used to toggle on, make sure to set the timezone once in the console, e.g. 
```
Timezone 99
```
for Germany (DE).

For inverted operation, i.e. switched off by any trigger and toggled on after a specific time, specify
```
PowerOnState 5
```
once in the console.


Flip-Flop
=========
Warmwater circulation
---------------------

```
rule1
  on power1#state=1 do 
    backlog power1 %value%; ruletimer1 240 
  endon 
  on rules#timer=1 do 
    power1 off
  endon 
  on power1#state=0 do 
    backlog power1 %value%; ruletimer2 1200 
  endon 
  on rules#timer=2 do 
    power1 on
  endon
  ON system#boot DO
    power1 2
  ENDON
```
Activate rule with ```rule1 1``` in the console.

Turn on for 2min, 30min off
---------------------------
rule1
  ON power1#state=1 DO
    backlog power1 %value%; ruletimer1 120
  ENDON
  ON rules#timer=1 DO
    power1 off
  ENDON
  ON power1#state=0 DO
    backlog power1 %value%; ruletimer2 1800
  ENDON
  ON rules#timer=2 DO
    power1 on
  ENDON
  ON system#boot DO
    power1 2
  ENDON
rule1 1

### oder ###

# alle halbe Stunde schalten
rule1 on power1#state=1 do backlog power1 %value%; ruletimer1 1800 endon on rules#timer=1 do power1 off endon on power1#state=0 do backlog power1 %value%; ruletimer2 1800 endon on rules#timer=2 do power1 on endon  ON system#boot DO power1 2 ENDON
rule1 1
