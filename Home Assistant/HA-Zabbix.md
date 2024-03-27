# Monitoring Home Assistant in Zabbix
Monitoring Home Assistant entities in Zabbix via <a href="https://developers.home-assistant.io/docs/api/rest/" target="_blank">HA REST API</a>.

![image](https://github.com/XUM-Computers/Zabbix/assets/164992171/c2dea0e8-2fd0-430c-a9a8-d8b2a48f5e6f)

## Home Assistant
1) Create an Long life access token _(If you don't know how, Google it)_
## Zabbix

  - Create host
    - Create master Item
    - Create dependent Item
  - Create template
    - Create Discovery rules

### Configuration #1
1) Create Host
  
   Example:
   ```
   Hostname: Home Assistant Stats
   Host group: Home Assistant
   ```
   ![image](https://github.com/XUM-Computers/Zabbix/assets/164992171/362345e1-b56e-4bf3-b923-31c4c626da86)
4) Create an master ITEM that will retrieve the states of all objects in Home Assistant
<br>   Example:
   ```
   - name: 'HA States'
     type: HTTP_AGENT
     key: ha.states
     history: '0'
     trends: '0'
     value_type: TEXT
     description:
     url: 'http://your_HA_IP:8123/api/states'
     headers:
       - name: Authorization
         value: 'Bearer {$HA_API}'
       - name: Content-Type
         value: application/json
     tags:
       - tag: component
         value: ha
   ```
   ![image](https://github.com/XUM-Computers/Zabbix/assets/164992171/67c15ea2-fc27-42a6-adf0-411b05ccbef8)

   - There is no need to store this data in the database, so I recommend set the _History storage time_ to _Do not store history_
     ![image](https://github.com/XUM-Computers/Zabbix/assets/164992171/34e0cc73-efa6-4f30-9629-6c3f18ee03c1)

6) Create a macro <code>{$HA_API}</code> with the API key _(Long life access token obtained in the first step from Home Assistant)_
   - Administration -> Macros
7) Create Dependent Items and use Preprocessing to search for a specific entity using JSONPath.
   _<br>The entity name can be found, for example, in Home Assistant under Developer Tools - States._

   Example _(Bathroom thermometer)_:
   ```
   - name: 'Teplota koupelna'
     type: DEPENDENT
     key: ha.teplota.koupelna
     delay: '0'
     value_type: FLOAT
     units: °C
     valuemap:
       name: Unknown
     preprocessing:
       - type: JSONPATH
         parameters:
           - '$[?(@.entity_id=='sensor.teplota_koupelna_temperature')].state'
       - type: TRIM
         parameters:
           - '[""]'
     master_item:
       key: ha.states
     tags:
       - tag: component
         value: ha
       - tag: component
         value: teplota
       - tag: mistnost
         value: koupelna
   ```
   ![image](https://github.com/XUM-Computers/Zabbix/assets/164992171/5298f2ad-d89d-4a44-847e-470a1bdc87f3)

### Configuration #2

This method involves creating a Template where we create one Master Item and then, using Discovery, create a separate Item for each metric.

1) Create Template
   
   ```
   Template name: Home Assistant
   Template groups: Home Assistant
   ```

2) Create an master ITEM that will retrieve the states of all objects in Home Assistant
  
   Example:
   ```
   - name: 'HA States'
     type: HTTP_AGENT
     key: ha.states
     history: '0'
     trends: '0'
     value_type: TEXT
     description:
     url: 'http://your_HA_IP:8123/api/states'
     headers:
       - name: Authorization
         value: 'Bearer {$HA_API}'
       - name: Content-Type
         value: application/json
     tags:
       - tag: component
         value: ha
   ```
   ![image](https://github.com/XUM-Computers/Zabbix/assets/164992171/67c15ea2-fc27-42a6-adf0-411b05ccbef8)
   
      - There is no need to store this data in the database, so I recommend set the _History storage time_ to _Do not keep history_
     ![image](https://github.com/XUM-Computers/Zabbix/assets/164992171/34e0cc73-efa6-4f30-9629-6c3f18ee03c1)

  3) Create new Discovery rule
  
      ![image](https://github.com/XUM-Computers/Zabbix/assets/164992171/0616c145-a055-43c3-97d2-676947e5da68)

      ![image](https://github.com/XUM-Computers/Zabbix/assets/164992171/82858cf2-1d19-4818-94cb-280ae868e705)

      ![image](https://github.com/XUM-Computers/Zabbix/assets/164992171/b2bca820-d8f1-45fe-93d3-bb78f286705c)

      Macros:
      - {#DEVICE_CLASS} = $..attributes.device_class.first()
        - serves for filtering entities _(in Filters tab)_
      - {#ENTITY_ID} = $..entity_id.first()
        - simply the entity ID
      - {#FRIENDLY_NAME} = $..attributes.friendly_name.first()
        - friendly name _(which (if) is entered in HA)_
      - {#UNIT} = $..attributes.unit_of_measurement.first()
        - the unit of the desired unit
     
      Filters:
      - {#DEVICE_CLASS} exist = checks if a device_class exists for the entity _(without that, Zabbix reported errors for Items that don't contain it)_
      - {#DEVICE_CLASS} matches = specify the device class that you want monitor _(temperature, humidity, power consumption, etc.)_ - you can find it again in HA _(Developer Tools - States)_
    
  4) Create Item prototype

     ![image](https://github.com/XUM-Computers/Zabbix/assets/164992171/aad7e575-7e49-4a74-af75-f365729f154f)

     ![image](https://github.com/XUM-Computers/Zabbix/assets/164992171/d9ffabde-e36a-4b17-993f-7b2cd697e137)

     ```
          item_prototypes:
              name: '{#FRIENDLY_NAME}'
              type: DEPENDENT
              key: 'ha.battery[{#ENTITY_ID}]'
              delay: '0'
              value_type: FLOAT
              units: '{#UNIT}'
              preprocessing:
                - type: JSONPATH
                  parameters:
                    - '$[?(@.entity_id=="{#ENTITY_ID}")].first()'
                - type: JSONPATH
                  parameters:
                    - $.state
              master_item:
                key: ha.states
     ```


  
   Congratulations, the values are now displayed in Zabbix.

   ![image](https://github.com/XUM-Computers/Zabbix/assets/164992171/a9ae4880-bd40-4f8e-9647-5be2026d107b)
   ![image](https://github.com/XUM-Computers/Zabbix/assets/164992171/d6997fc4-6e04-462a-b86e-052f681e177b)

