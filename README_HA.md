Hi,
This is the configuration that I came to with home assistant, is not neat, is not the most clean and efficient way, but, is which works with me.

The KG140F DC 0-120V 100A is connected with a 4 wire hose, the one that is for telephone, with a RJ10 connector attached to the KG in the RS485 port and the other end is crimped with JST and connected to the serial TX, RX and Ground pins from a D1-mini-ESP8266 with tasmota firmware.
The initial configuration on tasmota to send MQTT comms with:
```
Setoption19 0
Baudrate 115200
SerialSend 0
```

In HomeAssistant community forums they wrot about configuring a rule in tasmota like:
```
Rule1 on System#Boot do RuleTimer1 10 endon on Rules#Timer=1 do backlog SerialSend :R50=1,2,1,; RuleTimer1 10 endon
Rule1 1
```
But this kind of rule do not works for me, the serial channel is always sending each second the information. So many times something happend and Home assistant received differente values and the meassures are not correctly shown in the dashboards and in the historical data.

So this are my resutls....

After reading the documentation about the KG140F DC 0-120V 100A, The MQTT serial communication channel give me the following each second:

{"SerialReceived":":R50=1,\n:r50=1,119,1653,93,124123,109565,408091,869319,124,0,0,0,8008,29032,\r\n:R51=1,\n:r51=1,147,0,0,0,0,0,100,0,0,1275,100,100,100,0,0,1,\r\n"}

The information that is needed for the KG140F to works on Home Assistant is from the ":r50=" to the ":R51" so is this:

1,119,1653,93,124123,109565,408091,869319,124,0,0,0,8008,29032,\r\n

Each of the numbers is a position and define one of the parameters:

Explanation:
```
position    Description
1           1 represents the communication address;
2           119 represents the checksum;
3           1653 represents the voltage of 16.53V(1653/100);
4           93 represents instant current 0.93A(93/100;
5           124123 represents the remaining battery capacity is 124.123 Ah (124123/1000);
6           109565 means the Elapsed Ah 109.565 Ah (109565/1000);
7           408091 represents the cumulative charging capacity 4.08091kw.h (408091/100000);
8           869319 represents the running time of 869319 segs;
9           124 represents the ambient temperature is 24℃ (124-100); 
10          0 means the function is pending;
11          0 means the output status is ON; (0-ON, 1-OVP, 2-OCP, 3-LVP, 4-NCP, 5-OPP, 6-OTP, 255-OFF)
12          0 represents the direction of current, and the current is forward current; (0-forward, 1-reverse)
13          8008 means battery life is 8008 minutes;
14          29032 represents the internal resistance of the battery is 290.32mΩ. (29032/100)
```

To transform this to HomeAssistant the values start at 0 from the split position
Decoding:
```
                                HA position split
Address(1,2,3,4,5,6,....)               0
Checksum:153,		                        1
Voltage(V)(2056->20.56)                 2   
CurrentAmpere(A)(200->2.0)              3
RemainingAh(Ah)(5408->5.408)            4
ElapsedAH(Ah)			                      5   
AccumulatedChargingCapacity(Kwh)	      6
RunningTimeSeconds,				              7    
Temperature(ºC)(134->34ºc->134-100)     8
pending 		                            9
OutputStatus(alarms)                    10
CurrentDirection(0/1)                   11
BatteryLifeMinutes      	              12
ohm(30682->306.82->30682/100)           13
```

After struggling my head with the sensors, templates and so on, i finished with the following package(yaml file): pck_enrgyconsum_battery_control.yaml
At this package i'm able to control the update information from MQTT and try to overide the problems with odd information and values not in range.


```
###################################################################################
###################################################################################
input_number:
  number_ampere_preset:
    name: Number-Ampere-preset
    icon: mdi:battery-high
    min: 0
    max: 360
    initial: 127.5
    step: 0.1
    mode: box
    unit_of_measurement: Ah


###################################################################################
###################################################################################
mqtt:
  sensor:
    - name: kg140f_01_mqtt_filter
      # friendly_name: "KG140F-01-MQTT-FILTER"
      # icon: mdi:window-open-variant
      state_topic: "tele/tasmota_D1-Mini-01-ESP8266-Shunt-PowerBank/RESULT"
      value_template: >
        {% if 'SerialReceived' in value_json %}
          {{  value_json | regex_findall_index("r50=([^:]+):R51") }}
        {% endif %}
      # availability_topic: "tele/tasmota_03-ESP32-Shunt-PowerBank/LWT"
      # payload_available: "Online"
      # payload_not_available: "Offline"
      # qos: 0

###################################################################################
###################################################################################
automation:
###################################################################################
  - alias: Battery state - KG140F-01- Publish MQTT cleaned and checked values 
    description: ""
    trigger:
      - platform: state
        entity_id:
          - sensor.kg140f_01_mqtt_filter
        enabled: false
      - platform: time_pattern
        seconds: /5
        enabled: true
    condition: []
    action:
      - service: mqtt.publish
        data:
          qos: 0
          retain: true
          topic: tele/KG140F-01/RESULT
          payload: >-
            {% set t = states('sensor.kg140f_01_mqtt_filter') %} {% set maxAmp =
            (states('input_number.number_ampere_preset')|float * 1000|float) | float
            %} {% if (t.split(',') | count) == 15 %}
              {% if (t.split(',')[4] | float) <= maxAmp %}
                {{ states('sensor.kg140f_01_mqtt_filter') }}
              {% endif %}
            {% endif %}
      - service: mqtt.publish
        data:
          qos: 0
          retain: true
          topic: tele/KG140F-01/BAD
          payload: >-
            {% set t = states('sensor.kg140f_01_mqtt_filter') %} {% if (t.split(',')
            | count) != 15 %}
              {{ states('sensor.kg140f_01_mqtt_filter') }}
            {% endif %}
    mode: single

###################################################################################
###################################################################################
template:
# "SerialReceived": ":R50=1,\n:r50=1,210,1643,165,120191,109358,401245,857439,125,0,0,1,262,0,\r\n:R51=1,\n:r51=1,147,0,0,0,0,0,100,0,0,1275,100,100,100,0,0,1,\r\n"
  - trigger:
      - platform: mqtt
        # topic: tele/tasmota_03-ESP32-Shunt-PowerBank/RESULT
        topic: tele/KG140F-01/RESULT
    sensor:
      - name: KG140F-01-Address
        # unit_of_measurement: ""
        state: "{{ (trigger.payload.split(',')[0] | float) }}"

      - name: KG140F-01-Checksum
        # unit_of_measurement: "V"
        state: "{{ (trigger.payload.split(',')[1] | float) }}"

      - name: KG140F-01-Voltage
        unit_of_measurement: "V"
        state: "{{ (trigger.payload.split(',')[2] | float/100) }}"  

      - name: KG140F-01-CurrentAmpere
        unit_of_measurement: "Ah"
        state: "{{ (trigger.payload.split(',')[3] | float/100) }}"

      - name: KG140F-01-RemainingAh
        unit_of_measurement: "Ah"
        state: "{{ (trigger.payload.split(',')[4] | float/1000) }}"

      - name: KG140F-01-ElapsedAH
        unit_of_measurement: "Ah"
        state: "{{ (trigger.payload.split(',')[5] | float/1000) }}"

      - name: KG140F-01-AccumChrgCapacity
        unit_of_measurement: "Kwh"
        state: "{{ (trigger.payload.split(',')[6] | float/100000) }}"        

      - name: KG140F-01-RunningTimeSeconds
        unit_of_measurement: "s"
        state: "{{ (trigger.payload.split(',')[7] | float) }}"

      - name: KG140F-01-RunningTimeHours
        unit_of_measurement: "h"
        state: "{{ (trigger.payload.split(',')[7] | float/60/60) }}"        

      - name: KG140F-01-Temperature
        unit_of_measurement: "ºC"
        state: "{{ (trigger.payload.split(',')[8] | float-100) }}"        

      - name: KG140F-01-Pending
        # unit_of_measurement: ""
        state: "{{ (trigger.payload.split(',')[9] | float) }}"

      - name: KG140F-01-OutputStatus
        # unit_of_measurement: ""
        state: "{{ (trigger.payload.split(',')[10] | float) }}"

      - name: KG140F-01-CurrentDirection
        # unit_of_measurement: ""
        state: "{{ (trigger.payload.split(',')[11] | float) }}"        

      - name: KG140F-01-BatteryLifeMinutes
        unit_of_measurement: "min"
        state: "{{ (trigger.payload.split(',')[12] | float) }}"

      - name: KG140F-01-BatteryLifeHours
        unit_of_measurement: "h"
        state: "{{ (trigger.payload.split(',')[12] | float/60) }}"

      - name: KG140F-01-Ohm      
        unit_of_measurement: "mΩ"
        state: "{{ (trigger.payload.split(',')[13] | float/100) }}"
###################################################################################
###################################################################################
sensor:
# Calculate current power
  - platform: template
    sensors:
      kg140f_01_currentpower:
        value_template: >
            {% set t = states('sensor.kg140f_01_currentdirection') | float(0) %}
            {% if t == 0 %}
              {{ '%0.1f' | format ( -1 |float * 
                                    ( 
                                      states('sensor.kg140f_01_voltage') | float * 
                                      states('sensor.kg140f_01_currentampere') | float 
                                    )
                                  )
              }}
            {% elif t == 1 %}
              {{ '%0.1f' | format 
                              ( 
                                states('sensor.kg140f_01_voltage') | float * 
                                states('sensor.kg140f_01_currentampere') | float 
                              )
              }}
            {% else %}
              {{ '%0.1f' | format 
                              ( 
                                states('sensor.kg140f_01_voltage') | float * 
                                states('sensor.kg140f_01_currentampere') | float 
                              )
              }}              
            {% endif %}           
        unit_of_measurement: 'W'
        friendly_name: KG140F-01 Current Power
      
      kg140f_01_ampereconsumption:
        value_template: >
            {% set t = states('sensor.kg140f_01_currentdirection') | float(0) %}
            {% if t == 0 %}
              {{ '%0.1f' | format ( -1 |float * states('sensor.kg140f_01_currentampere') | float ) }}
            {% elif t == 1 %}
              {{states('sensor.kg140f_01_currentampere')|float }}
            {% else %}
              {{states('sensor.kg140f_01_currentampere')|float }}
            {% endif %}          
        unit_of_measurement: 'Ah'
        friendly_name: KG140F-01 Ampere Consumption

      kg140f_01_battery_level:
        value_template: >
          {{ '%0.1f' | format
                        (
                          (states('sensor.kg140f_01_remainingah') | float * 100 ) /
                          states('input_number.number_ampere_preset') | float
                        )
          }}       
        unit_of_measurement: '%'
        friendly_name: KG140F-01 Battery Level

```


So the dashboard is:

When the shunt detect the negative current:
![image](https://github.com/ldiegos/esphome-junctek_kgf/assets/29803600/bcdc8f09-a21b-48bc-9dbd-35b2331814c6)


When the shunt detect the positive current(charging):
![image](https://github.com/ldiegos/esphome-junctek_kgf/assets/29803600/01580810-45d6-47f5-9178-a2c59879604c)


Explanation:
- input_number.number_ampere_preset: This number represent the current capacity of the battery monitored with the KG, this is not received by the KG so i created a variable as input_number, in my situation, the 127.5 Ah is de total of the battery.
- mqtt.sensor.kg140f_01_mqtt_filter: This mqtt sensor, filter and ignore all the other stuff that the KG sents to the MQTT server and only get the "1,119,1653,93,124123,109565,408091,869319,124,0,0,0,8008,29032,\r\n"
- automation.Battery state - KG140F-01- Publish MQTT cleaned and checked values: The automation check two things, the information when its splitted has 15 positions in the array and also the remainingAh position(4) is below the ampere preset configure in the "input_number.number_ampere_preset". If everything is correct it publish to the MQTT server a new topic with the name of the KG, in my case KG140F-01
  I realise that time to time the remaingAh spikes over the ampere_preset and that give a incorrect meassure.
- template.trigger: This is in charge of connect to the new topic, tele/KG140F-01/RESULT and split the information into the sensors.
-sensor.template.sensors.kg140f_01_currentpower: This calculated sensor check the direction of the current and represents the positive or negative ampere consumption. Because the KG only gives absolute numbers and you should check the "CurrentDirection" position.
-sensor.template.sensors.kg140f_01_ampereconsumption: same as previous but with the ampere consumption.
-sensor.template.sensors.kg140f_01_battery_level: The SOC from the battery, because the KG do not give the percentaje only the remain capacity. This calculated take the input_number.number_ampere_preset and the remain Ah and give the percentaje of SOC.


Thanks a lot
Luis.
