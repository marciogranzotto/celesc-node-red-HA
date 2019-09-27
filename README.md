# celesc-node-red-HA
Get the current energy price for CELESC

## How it works

The formula to get the current price for a kWh is:
```
(baseRate+flagRate)/((100-(pis+confins+icms))/100)
```
let's break that up:
- `baseRate` is a fixed value for your home that only changes in August. Mine is `R$0.46978`. You can check yours [HERE](https://www.celesc.com.br/tarifas-de-energia#tarifas-vigentes)
- `flagRate` is the rate of the current Flag. That changes every month. You can read more [HERE](http://www.aneel.gov.br/bandeiras-tarifarias)
- `pis` and `confins` are tributes that change every month
- `icms` is 12% for the first 150kWh and 25% for any consumption after that

## Node-RED

For getting the values that change every month, we need to craw CELESC's page.
I'm doing that on a Node-RED flow.

### What you need

Before importing the flow, you need to add this to your Node-RED add-on configuration:
```json
"npm_packages": [
    "xmldom",
    "moment"
  ]
```
If you are using a stand-alone Node-RED installation just use
```shell
npm install xmldom moment
```

After that you need to enable the `require` package in Node-RED. To do it just open your `node-red/settings.js` and add `require:require` to `functionGlobalContext`, like so:
```javascript
functionGlobalContext: {
    require:require,
},
```

That will allow to use `require` inside a Function node to get `xmldom` and `moment`.

After that just paste this flow and change your broker settings:

```json
[{"id":"863c84ec.22c148","type":"inject","z":"6c3468f3.75c788","name":"","topic":"","payload":"","payloadType":"date","repeat":"7200","crontab":"","once":false,"onceDelay":0.1,"x":230,"y":440,"wires":[["3f7b4d1e.413432"]]},{"id":"90bef970.b544f8","type":"http request","z":"6c3468f3.75c788","name":"bandeiras","method":"GET","ret":"obj","paytoqs":false,"url":"","tls":"","proxy":"","authType":"","x":500,"y":440,"wires":[["b130fff1.9e33a"]]},{"id":"34251f38.944a8","type":"http request","z":"6c3468f3.75c788","name":"tarifa celesc","method":"GET","ret":"txt","paytoqs":false,"url":"https://www.celesc.com.br/tarifas-de-energia#tributos","tls":"","proxy":"","authType":"basic","x":370,"y":480,"wires":[["3798c67.ca40c3a"]]},{"id":"3798c67.ca40c3a","type":"html","z":"6c3468f3.75c788","name":"taxas","property":"payload","outproperty":"payload","tag":"#tributos > div > table:nth-child(2)","ret":"html","as":"multi","x":510,"y":480,"wires":[["284ac6c.adc473a"]]},{"id":"284ac6c.adc473a","type":"function","z":"6c3468f3.75c788","name":"valor","func":"var require = global.get('require');\nvar DOMParser = require('xmldom').DOMParser;\nvar moment = require('moment');\nlet parser = new DOMParser();\n\nlet table = parser.parseFromString(\"<table>\" + msg.payload + \"</table>\", \"text/xml\").getElementsByTagName(\"table\")[0];\n\nvar elements = table.lastChild.childNodes;\nlet currentDate = moment().format('MM/YYYY')\n\nvar msgOne = {topic: \"pis\", payload: {}}\nvar msgTwo = {topic: \"confins\", payload: {}}\n\nvar element = elements[3].childNodes\nnode.warn(element[3].textContent.replace(\",\", \".\"))\nnode.warn(element[5].textContent.replace(\",\", \".\"))\nmsgOne.payload = parseFloat(element[3].textContent.replace(\",\", \".\"))\nmsgTwo.payload = parseFloat(element[5].textContent.replace(\",\", \".\"))\n\n\n// for (let i = 0; i < elements.length; i++) {\n//     let elementA = elements[i];\n//     if (elementA.hasChildNodes()) {\n//         let element = elementA.childNodes\n//         if (element[1].textContent == currentDate) { // get current date\n//             msgOne.payload = parseFloat(element[3].textContent.replace(\",\", \".\"))\n//             msgTwo.payload = parseFloat(element[5].textContent.replace(\",\", \".\"))\n//         }\n//     }\n// }\n\n\nflow.set(\"pis\", msgOne.payload);\nflow.set(\"confins\", msgTwo.payload);\n\nreturn [msgOne, msgTwo];","outputs":2,"noerr":0,"x":630,"y":480,"wires":[["7e56e24a.a86fcc","55821d91.b82a24"],["55821d91.b82a24"]]},{"id":"7e56e24a.a86fcc","type":"function","z":"6c3468f3.75c788","name":"tarifa atual","func":"let pis = flow.get(\"pis\");\nlet confins = flow.get(\"confins\");\nlet flag = flow.get(\"flag\");\nlet base = 0.46978;\nlet icmsFirst = 12;\nlet icmsSecond = 25;\n\n// (base+flag)/((100-(pis+confins+icms))/100)\n\nmsgOne = {\n    topic: \"home/power/cost1\",\n    payload: (base+flag)/((100-(pis+confins+icmsFirst))/100)\n}\n\nmsgTwo = {\n    topic: \"home/power/cost2\",\n    payload: (base+flag)/((100-(pis+confins+icmsSecond))/100)\n}\n\nreturn [msgOne, msgTwo];","outputs":2,"noerr":0,"x":610,"y":520,"wires":[["46ee30d1.1fc48"],["46ee30d1.1fc48"]]},{"id":"46ee30d1.1fc48","type":"mqtt out","z":"6c3468f3.75c788","name":"","topic":"","qos":"1","retain":"true","broker":"e1a868b9.a931c8","x":770,"y":520,"wires":[]},{"id":"c0e46958.cfd788","type":"debug","z":"6c3468f3.75c788","name":"","active":true,"tosidebar":true,"console":false,"tostatus":false,"complete":"payload","targetType":"msg","x":770,"y":440,"wires":[]},{"id":"3f7b4d1e.413432","type":"function","z":"6c3468f3.75c788","name":"","func":"var url = \"https://apidosetoreletrico.com.br/api/energy-providers/tariff-flags?monthStart=\"\nvar date = new Date();\nvar firstDay = new Date(date.getFullYear(), date.getMonth(), 1).toISOString().split(\"T\")[0];\nvar lastDay = new Date(date.getFullYear(), date.getMonth() + 1, 0).toISOString().split(\"T\")[0];\nmsg.url = url + firstDay + \"&monthEnd=\" + lastDay\nreturn msg;","outputs":1,"noerr":0,"x":370,"y":440,"wires":[["90bef970.b544f8"]]},{"id":"b130fff1.9e33a","type":"function","z":"6c3468f3.75c788","name":"valor","func":"var value = msg.payload.items[0].value\nflow.set(\"flag\", value);\n\nmsg.topic = \"flag\"\nmsg.payload = value\nreturn msg;","outputs":1,"noerr":0,"x":630,"y":440,"wires":[["c0e46958.cfd788","34251f38.944a8"]]},{"id":"55821d91.b82a24","type":"debug","z":"6c3468f3.75c788","name":"","active":true,"tosidebar":true,"console":false,"tostatus":false,"complete":"payload","targetType":"msg","x":810,"y":480,"wires":[]},{"id":"e1a868b9.a931c8","type":"mqtt-broker","z":"","name":"","broker":"192.168.1.133","port":"1883","clientid":"","usetls":false,"compatmode":true,"keepalive":"60","cleansession":true,"birthTopic":"","birthQos":"0","birthPayload":"","closeTopic":"","closeQos":"0","closePayload":"","willTopic":"","willQos":"0","willPayload":""}]
```
#### This flow uses:
- [API do Setor Elétrico](https://apidosetoreletrico.com.br/api-docs/index.html) to get the current `flag` value.
- Crawling [CELESC's website](https://www.celesc.com.br/tarifas-de-energia#tributos) to get `pis` and `confins`. Keep in mind that if the website changes, this can break
- Hardcoded `base` tariff value on the `tarifa atual` node. Please change it to match your value.

## Home Assistant

You need to create two MQTT sensors, one for the first `<150 kWh` rate, and another for the `>150 kWh` rate

```yaml
sensor:
  - platform: mqtt
    state_topic: "home/power/cost1"
    name: "Power Cost less 150kWh"
    unit_of_measurement: "R$"
  - platform: mqtt
    state_topic: "home/power/cost2"
    name: "Power Cost more 150kWh"
    unit_of_measurement: "R$"
```

Assuming that you have a energy sensor called `sensor.power_meter_energy_total`, this two template sensors track the current power rate and the total cost.

```yaml
sensor:
  - platform: template
    sensors:
      power_rate:
        friendly_name: "Power Rate"
        unit_of_measurement: 'R$'
        value_template: >
          {% set energy = states('sensor.power_meter_energy_total') | float(default=-1) %}
          {% if energy != -1 %}
            {{states('sensor.power_cost_less_150kwh')|float if energy < 150 else states('sensor.power_cost_more_150kwh')|float}}
          {% else %}
            {{ states('sensor.power_rate') }}
          {% endif %}
      total_power_cost:
        friendly_name: "Power Cost"
        unit_of_measurement: 'R$'
        value_template: >
          {% set energy = states('sensor.power_meter_energy_total') | float(default=-1) %}
          {% if energy != -1 %}
            {% if energy < 150 %} 
                {{'%.2f'|format(energy * states('sensor.power_cost_less_150kwh')|float)}}
              {%- else -%} 
                {{'%.2f'|format((150 * states('sensor.power_cost_less_150kwh')|float) + (energy - 150 * states('sensor.power_cost_more_150kwh')|float))}}
              {%- endif %}
          {% else %}
            {{ states('sensor.total_power_cost') }}
          {% endif %}
```
