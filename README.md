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

Coffee mill
-----------
- Using an external power plug, e.g. Shelly Plug S

```
# invert PulseTimer: when device is turned OFF, it will be turned on again after <PulseTime> (10 is 1s)
BACKLOG PowerOnState 5; MEM1 10; PulseTime 10; VAR1 0
```
```
# if device detects power of more than 10W, start a timer and turn off after <MEM1> seconds
RULE1
  ON Energy#Power>10 DO
    IF(%VAR1%==0)
      BACKLOG RuleTimer1 %MEM1%; VAR1=1; WebQuery http://ntfy.sh/<topic> POST [Title: Coffee mill] running for %MEM1% sec
    ENDIF
  ENDON
  ON Rules#Timer=1 DO
    BACKLOG Power1 OFF; VAR1=0; WebQuery http://ntfy.sh/<topic> POST [Title: Coffee mill] stopped 
  ENDON
  ON System#Boot Do
    VAR1=0
  EndOn
```

Activate with
```
RULE1 1
```

- Using a built-in relay

Turns off after <MEM1> seconds pressing the button (a high voltage detector like [this](https://www.amazon.de/dp/B08HQ7K14H) connected to GPIO0, configured as a switch), then waits 3 seconds before reset. Relay connected to GPIO2.

```
BACKLOG MEM1 14; PulseTime 30
```
```
RULE1
  ON Switch1#state=1 DO
    RuleTimer1 %MEM1%
  ENDON
  ON Rules#Timer=1 DO
    Power1 ON
  ENDON
```

Activate with
```
RULE1 1
```

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
Red, yellow and green LEDs are connected to D4, D3 and D2 respectively via resistors of ~22 ohms (GPIOs of Wemos D1 Mini have 3,3V so we don't need much to protect the LEDs).

In the configuration page of Tasmota, D4, D3 and D2 are set to ```Relay``` ```1```, ```Relay``` ```2``` and ```Relay``` ```3``` respectively.

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

A little bit more advanced version, closer to reality:
```
Rule1
  On Power1#State=0 do
    Backlog Var1 0; Power3 1
  EndOn
  On Power2#State=0 Do
    If(%Var1%==1) Backlog Power1 1; Delay 80; Power2 1
    EndIf
  EndOn
  On Power3#State=0 Do
    Backlog Var1 1; Power2 1
  EndOn
  On System#Boot Do
    Backlog Power1 1; Delay 10; Power2 1
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
    BACKLOG Var1 ALARM_OFF; WebQuery http://ntfy.sh/<topic> POST [Title: <title for alarm off>] <message for alarm off>
  EndOn
  On Power1#State=1 Do
   BACKLOG Var1 ALARM_ON; WebQuery http://ntfy.sh/<topic> POST [Title: <title for alarm on] <message for alarm on>
  EndOn
```
This example includes a notification via the free notification service ntfy.sh.

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

To make sure one short cut is ony counted once, you need to set the ```CounterDebounce``` variable to a reasonable value (in milliseconds):
```
CounterDebounce 500
```

Distance / water level measurement
==================================
See https://favoss.de/smarte-wasserstandsanzeige-bauen/

Power-monitoring with Nous A1T and A5T power plugs
==================================================
The Nous A1T and A5T come preconfigured with Tasmota, but need calibration; see https://tasmota.github.io/docs/Power-Monitoring-Calibration/ on how to do this.

In case A5T was reset, you may need to reconfigure the rules for the power buttons:

```
{"NAME":"NOUS A5T","GPIO":[0,3072,544,3104,0,259,0,0,225,226,224,0,35,4704],"FLAG":1,"BASE":18}
```
```
Rule1 on analog#a0<100 do break ON analog#a0<300 DO Power3 2 ENDON
Rule1 1
Rule2 on analog#a0<350 do break ON analog#a0<600 DO Power2 2 ENDON
Rule2 1
Rule3 on analog#a0<600 do break ON analog#a0<990 DO Power1 2 ENDON
Rule3 1
```

Simple cronjobs doing web requests (call an URL)
==================================================
E.g. toggle another Tasmota's relay ever minute:
```
RULE1 ON Time#Minute|1 DO
  WebQuery http://<ip>/cm?cmnd=Power%20TOGGLE
ENDON
```
```
RULE1 1
```

or start charging your car every day 185 minutes after midnight (0:00), which is 3:05am, and stop (put into pv only mode) at 7:55am by calling the [wbec](https://github.com/steff393/wbec) (for [Heidelberg](https://www.amperfied.de/2022/11/21/wbec-fuer-heidelberg-wallbox-energy-control-blog/) chargers) API:

```
MEM1=185   # trigger for rule1 in minutes after midnight
MEM2=475   # trigger for rule2 in minutes after midnight
MEM3=60    # charging limit for grid mode [A * 10 ]
MEM4=160   # charging limit for PV mode [A * 10]
RULE1
  ON Time#Minute=%MEM1% DO
    WebQuery http://<ip>/json?id=0&pvMode=1&currLim=%MEM3%
  ENDON
RULE2
  ON Time#Minute=%MEM2% DO
    WebQuery http://<ip>/json?id=0&pvMode=2&currLim=%MEM4%
  ENDON
```
```
RULE1 1
RULE2 2
```
Several commands can be run using ```BACKLOG```.

One more example to toggle a switch every 10min between 12:00 and 18:00:
```
# 10min = 100+600
MEM1=100+60
PulseTime %MEM1%
# 12:00=12*60[min]=720
MEM2=12*60
# 18:00=18*60[min]=1080
MEM3=18*60
MEM4 20
RULE1
  ON time#minute<%MEM2% DO break
  ON time#minute>=%MEM3% DO break
  ON time#minute|%MEM4% DO power1 on
ENDON
```
```
RULE1 1
```

Complex cronjobs with timer-triggered rule
==========================================

1. Create timers with action "Rule" (Configuration -> Configure Timer)
2. Create a rule with ```Clock#Timer=<timer>``` indicating the associated timer:

```
RULE1
  ON Clock#Timer=1 DO
    BACKLOG WebQuery http://<ip>/cm?cmnd=Power%20ON ; WebQuery http://ntfy.sh/<topic> POST [Title: <title for power on>] <message for power on>
  ENDON
  ON Clock#Timer=2 DO
    BACKLOG WebQuery http://<ip>/cm?cmnd=Power%20OFF ; WebQuery http://ntfy.sh/<topic> POST [Title: <title for power off>] <message for power off>
  ENDON
```
```
RULE1 1
```

Charging a SonnenBatterie every night when Tibber price is low
==============================================================
=== Work in progress - not working atm ===
```
# SonnenBatterie API token:
MEM1=Auth-Token: <YourSonnenBattAPIToken>
# SonnenBatterie API URL:
MEM2=http://<YourSonnenBatteryIP>:80/api/v2
# charging limit in W
MEM3=4600

RULE1
  ON Clock#Timer=1 DO
    BACKLOG WebQuery %MEM2%/configurations PUT [%MEM1%] EM_OperatingMode=1 ; WebQuery %MEM2%/setpoint/charge/%MEM3% POST [%MEM1%] ; WebQuery http://ntfy.sh/wasserstr56info POST [Title: SonnenBattery state changed] SonnenBattery now charging with %MEM3%W
  ENDON
  ON Clock#Timer=2 DO
    BACKLOG WebQuery %MEM2%/configurations PUT [%MEM1%] EM_OperatingMode=2; WebQuery http://ntfy.sh/wasserstr56info POST [Title: SonnenBattery state changed] SonnenBattery now in auto mode
  ENDON
```
```
RULE1 1
```

Watchdog - turn device off and on if not reachable
==================================================
see https://tasmota.github.io/docs/Rules/#watchdog-for-wi-fi-router-or-modem

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
