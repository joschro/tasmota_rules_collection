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

Additional tools & information
------------------------------
To handle more than one Tasmota device with one central webinterface, you may want to give [TasmoAdmin](https://github.com/TasmoAdmin/TasmoAdmin) a try.

Information on retained message options can be found [here](https://tasmota.github.io/docs/MQTT/#retained-mqtt-messages).

Current sensor readings as JSON from web API: ```http://<ip>/cm?cmnd=Status%2010```

Timer
=====
To toggle a relay or light off after a specific time being switched on, use PulseTime:
```
PulseTime<x> 1..111ms / ((112..64900)-100)s
```
in Tasmota's console in the webinterface with \<x\> being the number of the relay/LED connected.

If the webinterface's timers are used to toggle on, make sure to set the timezone once in the console, e.g. 
```
Backlog0 Timezone 99; TimeStd 0,0,10,1,3,60; TimeDst 0,0,3,1,2,120
```
for Germany (DE).

To display the current time in the web ui, enter
```
WebTime 11,19
```
in the console (https://tasmota.github.io/docs/Commands/#webtime).

For inverted operation, i.e. switched off by any trigger and toggled on after a specific time, specify
```
PowerOnState 5
```
once in the console.


Flip-Flop
=========
Turn on for ```Mem1``` seconds, turn off again for ```Mem2``` seconds and repeat that indefinitively.

Use case examples:

Warm water circulation
----------------------
* Turning on for 4 minutes and turning off for 20 miutes
```
MEM1 240
MEM2 1200
```
* Toggle every half hour
```
MEM1 1800
MEM2 1800
```
Copy and paste into the Tasmota console:
```
BACKLOG MEM1 240; MEM2 1200
```
```
RULE1
  ON Power1#State=1 DO
    BACKLOG Power1 %value%; RuleTimer1 %MEM1%
  ENDON
  ON Rules#Timer=1 DO
    Power1 OFF
  ENDON
  ON Power1#State=0 DO
    BACKLOG Power1 %value%; RuleTimer2 %MEM2%
  ENDON
  ON Rules#Timer=2 DO
    IF (%VAR1%=ON)
      Power1 ON
    ELSE
      RuleTimer2 %MEM1%
    ENDIF
  ENDON
  ON System#Boot DO
    BACKLOG VAR1 ON; Power1 2
  ENDON
```
Activate rule with ```RULE1 1``` in the console. Using the variable of type ```MEM``` ensures they survive a reboot of the device. To change the timing, just re-enter ```MEM1 <x>``` and ```MEM2 <y>``` and restart the device.

To define times when activated/deactivated, you can use timers; under "Configuration" -> "Timer" check "Enable Timers" and define times when to activate and when to deactivate the flip-flop. Timer 1 defines the time of activation, timer 2 the time of deactivation; timer 3 and timer 4, timer 5 and 6 respectively.

```
RULE2
  ON Clock#Timer=1 DO
    VAR1 ON
  ENDON
  ON Clock#Timer=2 DO
    VAR1 OFF
  ENDON
  ON Clock#Timer=3 DO
    VAR1 ON
  ENDON
  ON Clock#Timer=4 DO
    VAR1 OFF
  ENDON
  ON Clock#Timer=5 DO
    VAR1 ON
  ENDON
  ON Clock#Timer=6 DO
    VAR1 OFF
  ENDON
```
Activate with
```
RULE2 1
```

Coffee mill
-----------
- Using an external power plug, e.g. Shelly Plug S

Invert PulseTimer: when device is turned OFF, it will be turned on again after <PulseTime> (10 is 1s):
```
BACKLOG PowerOnState 5; MEM1 14; PulseTime 20; VAR1 0
```

If device detects power of more than 10W, start a timer and turn off after <MEM1> seconds:
```
RULE1
  ON Energy#Power>10 DO
    IF(%VAR1%==0)
      BACKLOG RuleTimer1 %MEM1%; VAR1 1; WebQuery http://ntfy.sh/<topic> POST [Title: Coffee mill] running for %MEM1% sec
    ENDIF
  ENDON
  ON Rules#Timer=1 DO
    BACKLOG Power1 OFF; VAR1 0; WebQuery http://ntfy.sh/<topic> POST [Title: Coffee mill] stopped
  ENDON
  ON System#Boot Do
    VAR1 0
  EndOn
```

Activate with
```
RULE1 1
```

To add buttons for increasing/decreasing the milling time, add "Relay" 2 and "Relay" 3 in the device config to add two dummy web buttons on the unused GPIOs (e.g. GPIO3 and GPIO4); add a "Counter" 1 on e.g. GPIO16 to show the MEM1 value in the web gui:

```
BACKLOG WebButton2 Länger / + ; WebButton3 Kürzer / - ; Counter1 20
```
```
RULE2
  ON Power2#State=0 DO
    Counter1 +1
  ENDON
  ON Power3#State=0 DO
    Counter1 -1
  ENDON
  ON Counter#C1!=%MEM1% DO
    MEM1 %value%
  ENDON
```

Activate with
```
RULE2 1
```

- Using a built-in relay

Turns off after <MEM1> seconds pressing the button (a high voltage detector like [this](https://www.amazon.de/dp/B08HQ7K14H) connected to GPIO0, configured as a switch), then waits 2 seconds before reset. Relay connected to GPIO2 (inverted operation: connects by default, disconnects on activation).

```
BACKLOG MEM1 14; PulseTime 20; VAR1 0
```
```
RULE1
  ON Switch1#state=1 DO
    IF(%VAR1%==0)
      BACKLOG RuleTimer1 %MEM1%; VAR1 1; WebQuery http://ntfy.sh/<topic> POST [Title: Coffee mill] running for %MEM1% sec
    ENDIF
  ENDON
  ON Rules#Timer=1 DO
    BACKLOG Power1 ON; VAR1 0; WebQuery http://ntfy.sh/<topic> POST [Title: Coffee mill] stopped
  ENDON
  ON System#Boot Do
    VAR1 0
  EndOn
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
MEM1 <trigger level>
VAR1 ALARM_OFF
RULE1 ON [Sensor]#[ValueName]=%MEM1% do IF (%VAR1%=ALARM_OFF) VAR1 ALARM_ON ; Power1 1 ENDIF ENDON
      ON [Sensor]#[ValueName]!=%MEM1% DO IF (%VAR1%=ALARM_ON) VAR1 ALARM_OFF ; Power1 0 ENDIF ENDON
RULE2 ON Power2#State=0 Do VAR1 %MEM1% ENDON ON Power2#State=1 Do VAR1 0 ENDON
```
To reset the alarm, use a virtual button by adding a "Relay" config on one of the free GPIOs:
```
WebButton1 Alarm On/Off
```

Example with ```A0``` configured as ```ADC Input```, ```D1/GPIO5``` configured as ```Relay``` ```1``` and trigger set at 500:
```
MEM1 500
```
Ntfy topic:
```
MEM2 <MyTopic>
```
Alarm "ON" title:
```
MEM3 <MyAlarmOnTitle>
```
Alarm "Off" title:
```
MEM4 <MyAlarmOffTitle>
```
Alarm "ON" message:
```
MEM5 <MyAlarmOnMessage>
```
Alarm "Off" message:
```
MEM6 <MyAlarmOffMessage>
```
```
VAR1 ALARM_OFF
```
```
VAR2 unknown
```
```
RULE1
  ON ANALOG#A0>%Mem1% DO
    IF (%VAR1%=ALARM_OFF)
      VAR1 ALARM_ON; Power1 ON
    ENDIF
  ENDON
```
```
RULE2
  ON Power1#State=0 DO
    BACKLOG VAR1 ALARM_OFF; WebQuery http://ntfy.sh/%MEM2% POST [Title: %MEM4%] %MEM6% http://%VAR2%
  EndOn
  ON Power1#State=1 DO
   BACKLOG VAR1 ALARM_ON; WebQuery http://ntfy.sh/%MEM2% POST [Title: %MEM3%|Priority:max|Tags:rotating_light] %MEM5% http://%VAR2%
  ENDON
  ON StatusNET#IPAddress DO
   VAR2 %value%
  ENDON
```
This example includes a notification via the free notification service ntfy.sh.

Activate with 
```
RULE1 1
```
```
RULE2 1
```

To set the state correctly when using MQTT, add the following rule:
```
RULE3 ON Power1#state DO Publish stat/<topic>/POWER %value% ENDON
```
```
RULE3 1
```

Temperature guard / Temperaturwächter
=====================================
Using parasite mode, it is very easy to build a simple DS18x20 (DS1820, DS18B20, DS18S20 etc.) temperature sensor based temperature guard with only 2 wires.

1. connect both GND and 3.3V of the DS18x20 sensor (pins 1 and 3 in https://tasmota.github.io/docs/DS18x20/#wiring) to the GND pin of an ESP8266, e.g. WEMOS D1 Mini
2. connect the data pin to one of the GPIOs, e.g. GPIO14 (D5 on the WEMOS)
3. in Settings, configure this GPIO as "DS18x20" and a free GPIO, e.g. GPIO5 (D1 on the WEMOS) as "Relay"; this is needed to display and reset an alarm on the device
4. rename the relay button in the frontend:
   ```
   WebButton1 Alarm On/Off
   ```

Now the built-in pull-up resistor needs to be activated in the console:
```
SetOption74 1
```
After a reboot, the device should display the temperatur information.

Next step is to create a rule that fires an alarm when the temperature is above or below a certain threshold. We basically use the same code as in the above alarm device example:

Variables used:
* ```VAR1``` indicates current status: ALARM_ON / ALARM_OFF
* ```VAR2``` 1 if connected to network
* ```VAR3``` is the device's IP address
* ```VAR4``` temperature when alarm is activated

Temperature threshold:
```
BACKLOG MEM1 -10 ; MEM2 <ntfyTopic>
```

```
RULE1
  ON DS18S20#Temperature>%MEM1% DO
    IF ((%VAR1%=ALARM_OFF) AND (%VAR2%=1) AND (%value%!=85))
      BACKLOG VAR4 %value%; Power1 ON
    ENDIF
  ENDON
  ON Power1#State=0 DO
    BACKLOG VAR1 ALARM_OFF; WebQuery http://ntfy.sh/%MEM2% POST [Title: <title for alarm off>] <message for alarm off> http://%VAR3%
  ENDON
  ON Power1#State=1 DO
   BACKLOG VAR1 ALARM_ON; WebQuery http://ntfy.sh/%MEM2% POST [Title: <title for alarm on|Priority:max|Tags:rotating_light] %VAR4%°C - <message for alarm on> http://%VAR3%
  ENDON
```
```
RULE2
  ON System#Init DO
   BACKLOG VAR1 ALARM_OFF ; VAR2 0
  ENDON
  ON System#Boot DO
   BACKLOG IpAddress
  ENDON
  ON IpAddress1 DO
    BACKLOG VAR2 1 ; VAR3 %value% ENDON
  ENDON
  ON StatusNET#IPAddress DO
   VAR3 %value%
  ENDON
```

This example includes a notification via the free notification service ntfy.sh.

Activate with 
```
RULE1 1
```
```
RULE2 1
```

To set the state correctly with "retained" flag when using MQTT, add the following rule:
```
RULE3 ON Power1#state DO Publish2 stat/<topic>/POWER %value% ENDON
```
```
RULE3 1
```

Pulse counter
=============
Example use case is an analog water counter that short cuts two wires at every 1/1000 m³ (1l).
Counter2 is the counted amout in m³.
```
RULE1
  ON Counter#C1=1000 DO
    Counter2 +1
  ENDON
  ON Counter#C1>=1000 DO
    BACKLOG Var1 0; Counter1 0
  ENDON
```
```
RULE1 1
```

To make sure one short cut is ony counted once, you need to set the ```CounterDebounce``` variable to a reasonable value (in milliseconds):
```
CounterDebounce 500
```

To publish the full count in l (Counter1 + Counter2), you can add the following rules:
```
BACKLOG MEM1 0; MEM2 360; Var1 0; Var3 0
```
```
RULE2
  ON Counter#C1>%Var1% DO
    IF (%value%<1000) Var1 %value% ; Var2=%value%/1000+%Mem1% ENDIF
  ENDON
  ON Counter#C2>%Mem1% DO
    Mem1 %value%
  ENDON
  ON Rules#Timer=1 DO
    BACKLOG Var4=1000*(%Var2%-%Var3%)/(%MEM2%/3600); RuleTimer1 %MEM2%
  ENDON
  ON Rules#Timer=1 DO
    BACKLOG Var3=%Var2%; Publish tele/<topic>/FlowRate %Var4%
  ENDON
```
```
RULE3
  ON Counter#C1>%Var1% DO
    Publish tele/<topic>/Counter %Var2%
  ENDON
  ON System#Boot do
    BACKLOG RuleTimer1 %MEM2%; Var1 0; Var3 0
  ENDON
```
```
BACKLOG RULE2 1; RULE3 1
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
  ON time#minute|%MEM4% DO Power1 ON
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

Send temperature value to Google Sheet via Google Forms
=======================================================
###WIP###
1. Follow https://stackoverflow.com/questions/71714110/can-you-submit-a-restful-request-to-a-google-forms-api to create a Google form and extract the URL to submit a value.
2. Make sure to "Publish" the Google Form after creation

Adjust the time between updates to your preferences: ```Mem1``` is delay in seconds until the next temperature value is being submitted.

```
Mem1 300
```
```
Rule1
  ON tele-DS18B20#Temperature DO
    Var1 %value%
  ENDON
  ON Rules#Timer=1 do
    BACKLOG WebQuery https://docs.google.com/forms/d/e/<your-forms-id>/formResponse?entry.408302265=%VAR1%; RuleTimer1 %MEM1%
  ENDON
  ON System#Boot do
    RuleTimer1 %Mem1%
  ENDON
```
Activate with
```
Rule1 1
```
in the console.

Charging a SonnenBatterie every night when Tibber price is low
==============================================================
```
# SonnenBatterie API token:
MEM1 Auth-Token: <YourSonnenBattAPIToken>
# SonnenBatterie API URL:
MEM2 http://<YourSonnenBatteryIP>:80/api/v2
# charging limit in W
MEM3 4600
# Ntfy topic
MEM4 <MyTopic>

RULE1
  ON Clock#Timer=1 DO
    BACKLOG WebQuery %MEM2%/configurations PUT [%MEM1%] EM_OperatingMode=1 ; WebQuery %MEM2%/setpoint/charge/%MEM3% POST [%MEM1%] ; WebQuery http://ntfy.sh/%MEM4% POST [Title: SonnenBattery state changed] SonnenBattery now charging with %MEM3%W
  ENDON
  ON Clock#Timer=2 DO
    BACKLOG WebQuery %MEM2%/configurations PUT [%MEM1%] EM_OperatingMode=2 ; WebQuery http://ntfy.sh/%MEM4% POST [Title: SonnenBattery state changed] SonnenBattery now in auto mode
  ENDON
```
To manually turn on charging, configure one of the free GPIOs as "Relay" and rename the button:
```
WebButton1 Charging On/Off
```
```
RULE2
  ON Power1#State=1 DO
    BACKLOG WebQuery %MEM2%/configurations PUT [%MEM1%] EM_OperatingMode=1 ; WebQuery %MEM2%/setpoint/charge/%MEM3% POST [%MEM1%] ; WebQuery http://ntfy.sh/%MEM4% POST [Title: SonnenBattery state changed] SonnenBattery now charging with %MEM3%W
  EndOn
  ON Power1#State=0 DO
   BACKLOG WebQuery %MEM2%/configurations PUT [%MEM1%] EM_OperatingMode=2 ; WebQuery http://ntfy.sh/%MEM4% POST [Title: SonnenBattery state changed] SonnenBattery now in auto mode
  ENDON
  ```
```
RULE1 1
```
```
RULE2 1
```

To set the state correctly when using MQTT, add the following rule:
```
RULE3 ON Power1#state DO Publish stat/topic/POWER %value% ENDON
```
```
RULE3 1
```


Heating temperatures
====================
The command 
```
DS18Alias 0123456789ABCDEF,AliasMax16Chars
```
is available if compiled with option "#define DS18x20_USE_ID_ALIAS" in user_config_override.h; can be used to assign useful names for sensors:
```
RULE3
  ON System#Boot DO
    Backlog
      DS18Alias B40000006B772F28,Vorlauf_Heiz;
      DS18Alias 270000006B026C28,Rücklauf_Heiz;
      DS18Alias C80000006B651D28,Vorlauf_Fussb;
      DS18Alias 3B00000082E0AC28,Rücklauf_Fussb;
      DS18Alias 210000006B6DC328,WarmWasser;
      DS18Alias F30000006ABC4C28,Zirkulation;
      DS18Alias BE0000008321D028,Vorlauf_Solar;
      DS18Alias F900000086B6F928,Rücklauf_Solar;
      DS18Alias A8000801CA1EF010,Heizungsraum;
      DS18Alias 9F00000085D75B28,Rücklauf_Brenn;
      DS18Alias A800000085D8DD28,Laden_WaWa;
      DS18Alias 91000000860BD528,Vorlauf_Brenn;
      DS18Alias 69000000860BDE28,Laden_Heiz;
      DS18Alias AA00000086A35128,Speicher
    ENDON
```
```
RULE3 1
```
To enable more than 8 DS18x20 sensors, add option "DS18X20_MAX_SENSORS" to user_config_override.h:
```
#ifdef FIRMWARE_HEATING
    // This line will issue a warning during the build (yellow in 
    // VSCode) so you see which section is used
    #warning **** Build: HEATING ****
    // -- CODE_IMAGE_STR is the name shown between brackets on the 
    //    Information page or in INFO MQTT messages
    #undef CODE_IMAGE_STR
    #define CODE_IMAGE_STR "heating"

    // Put here your override for firmware tasmota-heating
    #define DS18X20_MAX_SENSORS 18
    #define DS18x20_USE_ID_ALIAS

#endif
```
Make it available in platformio_tasmota_cenv.ini for compiling:
```
[env:tasmota-heating]
extends                 = env:tasmota-4M
board                   = esp8266_4M2M
build_flags             = ${env.build_flags} -DFIRMWARE_HEATING
```
Run with
```
pio run -e tasmota-heating
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
* https://sites.google.com/view/hichi-lesekopf
* https://hessburg.de/tasmota-wifi-smartmeter-konfigurieren/
* https://tasmota.github.io/docs/Smart-Meter-Interface/#emh-ehz-generation-k-sml
* https://ottelo.jimdofree.com/stromz%C3%A4hler-auslesen-tasmota/#1
* https://docs.google.com/document/d/1olSGZcaE_vkdNXzN_p0zGee61xKCcK3HJe3Al00xj2g/edit
* https://wiki.volkszaehler.org/hardware/channels/meters/power/edl-ehz/emh-ehz-h1#:~:text=Der%20EMH%20eHZ%2DH%20kann,die%20IR%2DSchnittstelle%20ausgelesen%20werden.
* https://youtu.be/VuXpzKetOhc
* https://www.smarthomejetzt.de/gaszaehler-mit-wemos-d1-mini-mit-reed-kontakt-pullup-widerstand-und-tasmota-smart-machen/
* https://homeitems.de/gas-und-wasserzaehler-mit-tasmota-digitalisieren/

Shutter automation with Shelly Plus 2PM Gen3
--------------------------------------------
Setup: https://github.com/arendst/Tasmota/discussions/22526
* Configure -> Other -> Template:
  
  ```{"NAME":"Shelly 2PM Gen3","GPIO":[9472,3458,576,225,4736,224,640,608,1,1,193,0,0,0,0,0,0,0,192,32,1,1],"FLAG":0,"BASE":1}```
* Console: ```ADCGPIO4 10000,10000,4000```

From https://tasmota.github.io/docs/Blinds-and-Shutters/#autosetup-only-shelly-plus-2pm-esp32-based

Setup Shelly to act as a shutter control:
```
Backlog SetOption80 1 ; Shuttermode 1
```
If shutter is completely open, you can force it to close:
```
Backlog Shuttersetopen ; Shutterclose
```
Automatic setup of open and close position; shutter needs to be in closed position:
```
Backlog Interlock 1,2 ; Interlock on ; Shutterrelay1 1 ; Shuttersetup
```
Repeat if it fails.
It can be controlled with e.g.
```
ShutterPosition 50
```

Light controlled via DALI protocol
----------------------------------
See https://tasmota.github.io/docs/DALI/#shelly-dali-dimmer-gen3 and https://us.shelly.com/blogs/documentation/shelly-dali-dimmer-gen3 .

* Flash Shelly DALI Dimmer with latest tasmota32c3.bin
* Enable DALI
```
DaliLight 1
```
```
Backlog AdcGpio1 10000,10000,4000; ButtonTopic 0; SetOption1 1; SetOption11 0; SetOption32 20; DimmerStep 5; LedTable 0
```
* Create rules
```
RULE1
  ON Button1#State=2 DO
    dimmer +
  ENDON
  ON Button2#State=2 DO
    dimmer -
  ENDON
  ON Button1#State=3 DO
    power 2
  ENDON
  ON Button2#State=3 DO
    power 2
  ENDON
```
```
RULE2
  ON System#Boot DO
    DaliPower On
  ENDON
```
```
BACKLOG RULE1 1; RULE2 1
```

wbec-control - Add timer functionality to free wbec version
-----------------------------------------------------------

```CalcRes 0```                 # Prevent decimals to be shown in variable calculation

```MEM1 192.168.178.102```    # IP address of wbec

```MEM2 0```                  # ID of first wallbox

```MEM3 1```                  # ID of second wallbox

```MEM4 16```                 # max charging from grid

```MEM5 16```                 # max charging when PV only

```MEM6 Wasserstr56Info```                 # NTFY.sh topic

```
RULE1
  ON Clock#Timer=1 DO
    Power1 1
  ENDON
  ON Clock#Timer=2 DO
    Power1 0
  ENDON
  ON Clock#Timer=3 DO
    Power1 1
  ENDON
  ON Clock#Timer=4 DO
    Power1 0
  ENDON
  ON Clock#Timer=5 DO
    Power2 1
  ENDON
  ON Clock#Timer=6 DO
    Power2 0
  ENDON
  ON Clock#Timer=7 DO
    Power2 1
  ENDON
  ON Clock#Timer=8 DO
    Power2 0
  ENDON
```
```
RULE2
  ON Power1#State=1 DO
    BACKLOG WebQuery http://%MEM1%/json?id=%MEM2%&pvMode=1&currLim=%VAR1% ; WebQuery http://ntfy.sh/%MEM6% POST [Title: Wallbox charging state changed] Wallbox %MEM2% now charging from grid with max %MEM4%A
  ENDON
  ON Power1#State=0 DO
    BACKLOG WebQuery http://%MEM1%/json?id=%MEM2%&pvMode=2&currLim=%VAR2% ; WebQuery http://ntfy.sh/%MEM6% POST [Title: Wallbox charging state changed] Wallbox %MEM2% now charging PV only with max %MEM5%A
  ENDON
  ON Power2#State=1 DO
    BACKLOG WebQuery http://%MEM1%/json?id=%MEM3%&pvMode=1&currLim=%VAR1% ; WebQuery http://ntfy.sh/%MEM6% POST [Title: Wallbox charging state changed] Wallbox %MEM3% now charging from grid with max %MEM4%A
  ENDON
  ON Power2#State=0 DO
    BACKLOG WebQuery http://%MEM1%/json?id=%MEM3%&pvMode=2&currLim=%VAR2% ; WebQuery http://ntfy.sh/%MEM6% POST [Title: Wallbox charging state changed] Wallbox %MEM3% now charging PV only with max %MEM5%A
  ENDON
```
```
RULE3
  ON Power1#State DO
    Publish stat/wbec_ctrl/POWER1 %value%
  ENDON
  ON Power2#State DO
    Publish stat/wbec_ctrl/POWER2 %value%
  ENDON
  ON System#Boot DO
    BACKLOG VAR1=MEM4*10; VAR2=MEM5*10
  ENDON
```
```
BACKLOG RULE1 1; RULE2 1; RULE3 1
```
