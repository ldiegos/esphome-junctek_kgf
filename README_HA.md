Hi,
This is the configuration that I came to with home assistant, is not neat, is not the most clean and efficient way, but, is which works with me.

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

      daly150a_powerwall_currentpower:
        value_template: >
          {{ '%0.1f' | format
                        (
                          states('sensor.02_dalybmss_esp32_battery_voltage') | float * 
                          states('sensor.02_dalybmss_esp32_battery_current') | float
                        )
          }}
        unit_of_measurement: 'W'
        friendly_name: Daly150a-Powerwall-CurrentPower

```


The only thing that is not working for me, is the tasmota rule, despite whatever i put in the rule, the transmission to the HA is every second... I bought some D1 mini, just for check if the problem is the ESP32 or not.

```
Rule1 on System#Boot do RuleTimer1 10 endon on Rules#Timer=1 do backlog SerialSend :R50=1,2,1,; RuleTimer1 10 endon

Rule1 1
```


So the dashboard is:
When the shunt detect the negative current:
![image|690x166](upload://dM73EeXEu9Ss5MgNioPPWZxA0zu.png)

When the shunt detect the positive current(charging):
![image|690x180](upload://QfwWV7bBl9qrOmW6758Ui9sML9.png)


Thanks a lot to all of you for your knowledge
Luis.
