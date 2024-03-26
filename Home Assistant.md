# Monitoring Home Assistant in Zabbix
Monitoring Home Assistant entities in Zabbix via HA REST API.
## Home Assistant
1) Create an Long life access token _(If you don't know how, Google it.)_
## Zabbix
_Note: This is just one of many possible approaches._
1) Create Host
   Example:
   ```
   Hostname: Home Assistant Stats
   Host group: Home Assistant
   ```
   ![image](https://github.com/XUM-Computers/Zabbix/assets/164992171/362345e1-b56e-4bf3-b923-31c4c626da86)
2) Create an ITEM that will retrieve the states of all objects in Home Assistant
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
3) Create a macro <code>{$HA_API}</code> with the API key (token)
   - Administration -> Macros
4) Create Dependent Items, where we use Preprocessing to search for a specific entity using JSONPath.
   <br>The entity name can be found, for example, in Home Assistant under Developer Tools - States.

   Example:
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
           - '$[?(@.entity_id==''sensor.teplota_koupelna_temperature'')].state'
       - type: TRIM
         parameters:
           - '[""]'
       - type: JAVASCRIPT
         parameters:
           - |
             if (value.includes("unknown")) {
                 return -100
             } else if (value.includes("unavailable")) {
                 return -100
             } else {
                 return parseFloat(value)
             }
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
   ![image](https://github.com/XUM-Computers/Zabbix/assets/164992171/e3d8d531-32ef-4649-ad9a-f0d693e51786)
   - Because some entities may occasionally be unavailable (for example, if the battery in a thermometer dies or in some wireless button, etc.), Zabbix would return the value "unknown" or "unavailable". However, since we need the value as a Float to display numbers in graphs, Zabbix would return an error "Not Supported". This is handled by JavaScript, where if the value is "unknown" or "unavailable", "-100" is displayed (here any value we choose can be used). This value can then be named via Value mapping. If everything is fine and the device is available, we convert String to Float.
      ![image](https://github.com/XUM-Computers/Zabbix/assets/164992171/3914c2df-977b-4321-80b9-03f4d3ac9a6c)


