# tasmota_rules_collection
Collection of useful Tasmota (firmware for ESP8266 / ESP32 devices) commands and [rules](https://tasmota.github.io/docs/Rules/). Many more can be found [here](https://tasmota.github.io/docs/Rules/#rule-cookbook). See [here](https://www.ei-ot.de/tasmota-rules-tutorial-mit-expressions-und-if-statement/) for more information on rules.

This collection contains some useful commands and rules to customize [Tasmota firmware](https://tasmota.github.io/) from the [Tasmota project](https://tasmota.app/) for various use cases with ESP8266/ESP32 based devices (see [list of compatible devices](https://templates.blakadder.com/).

I use the Wemos Mini D1 ([micro-USB](https://www.berrybase.de/d1-mini-esp8266-entwicklungsboard?c=306)/[USB-C](https://www.berrybase.de/d1-mini-esp8266-entwicklungsboard-usb-c?c=306)) a lot, with add-ons like the [Relais Shield](https://www.berrybase.de/relais-shield-fuer-d1-mini?c=306). For difficult Wifi environments, there's a [version with external antenna](https://www.berrybase.de/d1-mini-pro-esp8266-entwicklungsboard-mit-u.fl-anschluss-set-mit-antenne?c=306) available, too.

Setup
-----
To configure Tasmota for the Wemos D1 Mini, you need to flash the firmware from https://ota.tasmota.com/tasmota/release/ with
```
esptool --port /dev/ttyUSB0 write_flash --flash_size=4MB -fm dio 0 <e.g. tasmota-DE.bin>
```
on a Linux system in this case. Make sure to hold the reset button pressed while plugging in the USB cable and execute the above command right after releasing the reset button. After successfully flashing the device, connect to the new "tasmota-xxxx" wifi network with your mobile and complete the wifi setup to add the Wemos D1 Mini to your home network.

Now go into "Settings" in the Wemos D1 Mini webinterface and select "Device Settings"; change device type to "Generic(18)". You are now ready to start playing with your new IoT device.

Additional tools
----------------
To handle more than one Tasmota device with one central webinterface, you may want to give [TasmoAdmin](https://github.com/TasmoAdmin/TasmoAdmin) a try.

Timer
=====
To toggle a relay or light off after a specific time being switched on, use PulseTime:
```
PulseTime<x> 1..111ms / ((112..64900)-100)s
```
in Tasmota's console in the webinterface with \<x\> being the number of the relay/LED connected.

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
Turn on for ```Mem1``` seconds, turn off again for ```Mem2``` seconds and repeat that indefinitively.
Use case examples:
* Warm water circulation, turning on for 4 minutes and turning off for 20 miutes
```
Mem1 240
Mem2 1200
```
* Toggle every half hour
```
Mem1 1800
Mem2 1800
```
Copy and paste into the Tasmota console:
```
Mem1 240
```
```
Mem2 1200
```
```
Rule1
  On power1#state=1 Do 
    Backlog Power1 %value%; RuleTimer1 %Mem1%
  Endon 
  On Rules#Timer=1 Do
    Power1 off
  EndOn 
  On Power1#State=0 do 
    Backlog Power1 %value%; RuleTimer2 %Mem2%
  EndOn 
  On Rules#Timer=2 do 
    Power1 on
  Endon
  On System#Boot Do
    Power1 2
  EndOn
```
Activate rule with ```Rule1 1``` in the console. Using the variable of type ```Mem``` ensures they survive a reboot of the device. To change the timing, just re-enter ```Mem1 <x>``` and ```Mem2 <y>``` and restart the device.

Pulsing LED light (Dodow)
-------------------------
When turned on, the Dodow device throws a pulsing blue LED spot to the ceiling for you to breathe with the pulsing light and fall into sleep more easily.

Adjust the speed to your preferences: ```Mem1``` is the speed with which the light turns on, ```Mem2``` is the speed with which the light turns off.
```Mem3``` sets the time until the device turns itself off.

(Maybe required:
```Backlog SetOption0 0; SetOption15 1; LedPower 0; SetOption31 1```)

```
Mem1 4
```
```
Mem2 4
```
```
Mem3 480
```
```
rule1
  on Rules#Timer=1 do
    backlog speed %Mem2%; dimmer 0; delay %var2%; speed %Mem1%; dimmer 100; ruletimer1 %var1%
  endon
  on System#Boot do
    backlog  var1=Mem1/2; var2=Mem2/2*10; fade 1; ruletimer2 Mem3; ruletimer1 1
  endon
rule2
  on Rules#Timer=2 do
    backlog ruletimer1 0; dimmer 0; ruletimer2 30
  endon
```
Activate with
```
rule1 1
```
```
rule2 1
```
in the console.

Use ```Dimmer +```, ```Dimmer -``` with ```DimmerStep <steps>```; ```Dimmer >``` and ```Dimmer<``` to go 100% resp. 0%; read dimmer state with ```dimmer#state```. See https://github.com/arendst/Tasmota/discussions/12280

Traffic light
=============
Red, yellow and green LEDs are connected to D1, D2 and D3 respectively via resistors of ~22 ohms (GPIOs of Wemos D1 Mini have 3,3V so we don't need much to protect the LEDs).

Rule1 should look like this:
- on boot, trigger red to turn on for 10,5 seconds
- when red turns off, trigger yellow to turn on for 2,5 seconds
- when yellow turns off _and_ the light before has been red, trigger green to turn on for 8 seconds
- when green turns off, trigger yellow to turn on for 2,5 seconds
- when yellow turns off _and_ the light before yellow has been green, trigger red to turn on for 10,5 seconds

```
PulseTime1 105
```
```
PulseTime2 25
```
```
PulseTime3 80
```
```
Rule1
  On Power1#State=0 do
    Backlog Var1 0; Power2 1
  EndOn
  On Power2#State=0 Do
    If(%Var1%==1) Power1 1 Else Power3 1 EndIf
  EndOn
  On Power3#State=0 Do
    Backlog Var1 1; Power2 1
  EndOn
  On System#Boot Do
    Power1 1
  EndOn
```

Activate rule with ```Rule1 1``` in the console and power off/on or use the Wemos D1 Mini reset button to restart the device.


Trigger alarm when threshold is exceeded
========================================
An ESP8266 can measure an analog signal / voltage on the ```A0``` pin; to enable measuring voltage above 1V, use a voltage divider like [this](https://www.amazon.de/dp/B07G8WYHFJ) or simply build one using two resistors; see https://tasmota.github.io/docs/ADC , https://www.smarthome-tricks.de/esp8266/elektrische-spannung-messen/ , https://nerdiy.de/de_de/howto-espeasy-wemos-d1-mini-adc-an-eine-andere-maximalspannungen-anpassen/ and https://github.com/arendst/Tasmota/issues/7182 for more information.

Generic rules:
```
Mem1 <trigger level>
Var1 ALARM_OFF
Rule1 On [Sensor]#[ValueName]=%Mem1% do Power1 1 EndOn On [Sensor]#[ValueName]!=%Mem1% Do Power1 0 EndOn
Rule2 On Power2#State=0 Do Var1 %Mem1% EndOn On Power2#State=1 Do Var1 0 EndOn
```
To reset the alarm, use a virtual button by adding a "Relay" config on one of the free GPIOs:
```
WebButton1 Alarm <p> On/Off
```

Example with ```A0``` configured as ```ADC Input```, ```D1/GPIO5``` configured as ```Relay``` ```1``` and trigger set at 500:
```
Mem1 500
```
```
Var1 ALARM_OFF
```
```
Rule1
  On ANALOG#A0>%Mem1% Do
    If (%var1%=ALARM_OFF) Var1 ALARM_ON; Power1 1
  EndOn
```
```
Rule2
  On Power1#State=0 Do
    Var1 ALARM_OFF
  EndOn
  On Power1#State=1 Do
    Var1 ALARM_ON
  EndOn
```

Activate with 
```
Rule1 1
```
```
Rule2 1
```

Pulse counter
=============
Example use case is an analog water counter that short cuts two wires at every 1/1000 m³ (1l).
Counter2 is the counted amout in m³.
```
Rule1
  On Counter#C1=1000 Do
    Counter2 +1
  EndOn
  On Counter#C1>=1000 do
    Counter1 0
  EndOn
```
```
rule1 1
```

Distance / water level measurement
==================================
See https://favoss.de/smarte-wasserstandsanzeige-bauen/

Power-monitoring with Nous A1T power plug
=========================================
The Nous A1T comes preconfigured with Tasmota, but needs calibration; see https://tasmota.github.io/docs/Power-Monitoring-Calibration/ on how to do this.


More Tasmota projects
=====================

Reading SmartMeters with Tasmota and Hichi smart meter
------------------------------------------------------
Useful links:
* https://www.heise.de/tests/Ausprobiert-Guenstiger-IR-Lesekopf-fuer-Smart-Meter-mit-Tastmota-Firmware-7065559.html
* https://hessburg.de/tasmota-wifi-smartmeter-konfigurieren/
* https://ottelo.jimdofree.com/stromz%C3%A4hler-auslesen-tasmota/#1
* https://docs.google.com/document/d/1olSGZcaE_vkdNXzN_p0zGee61xKCcK3HJe3Al00xj2g/edit
* https://wiki.volkszaehler.org/hardware/channels/meters/power/edl-ehz/emh-ehz-h1#:~:text=Der%20EMH%20eHZ%2DH%20kann,die%20IR%2DSchnittstelle%20ausgelesen%20werden.
* https://youtu.be/VuXpzKetOhc
* https://www.smarthomejetzt.de/gaszaehler-mit-wemos-d1-mini-mit-reed-kontakt-pullup-widerstand-und-tasmota-smart-machen/
* https://homeitems.de/gas-und-wasserzaehler-mit-tasmota-digitalisieren/
