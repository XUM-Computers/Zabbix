# Monitoring Home Assistant in Zabbix
Monitoring Home Assistant entities in Zabbix via <a href="https://developers.home-assistant.io/docs/api/rest/" target="_blank">HA REST API</a>.

![image](https://github.com/XUM-Computers/Zabbix/assets/164992171/c2dea0e8-2fd0-430c-a9a8-d8b2a48f5e6f)

## Home Assistant
1) Create an Long life access token _(If you don't know how, Google it)_
## Zabbix
Brief overview:
- Create host
- Create master Item
- Create dependent Items

_Note: This is just one of many possible approaches._
### Configuration
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
  
   Congratulations, the values are now displayed in Zabbix.

   ![image](https://github.com/XUM-Computers/Zabbix/assets/164992171/a9ae4880-bd40-4f8e-9647-5be2026d107b)
   ![image](https://github.com/XUM-Computers/Zabbix/assets/164992171/d6997fc4-6e04-462a-b86e-052f681e177b)

