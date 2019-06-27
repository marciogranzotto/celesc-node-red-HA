# celesc-node-red-HA
Get the current energy price for CELESC

## How it works

The formula to get the current price for a kWh is:
```
(baseRate+flagRate)/((100-(pis+confins+icms))/100)
```
let's break that up:
- `baseRate` is a fixed value for your home. Mine is `R$0.52049`. You can check yours [HERE](http://www.celesc.com.br/portal/index.php/duvidas-mais-frequentes/1140-tarifa)
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
[{"id":"863c84ec.22c148","type":"inject","z":"6c3468f3.75c788","name":"","topic":"","payload":"","payloadType":"date","repeat":"7200","crontab":"","once":false,"onceDelay":0.1,"x":130,"y":400,"wires":[["90bef970.b544f8"]]},{"id":"90bef970.b544f8","type":"http request","z":"6c3468f3.75c788","name":"celesc","method":"GET","ret":"txt","paytoqs":false,"url":"http://www.celesc.com.br/portal/","tls":"","proxy":"","authType":"basic","x":270,"y":400,"wires":[["43a567f.fafd998"]]},{"id":"abfb297b.46c3d8","type":"function","z":"6c3468f3.75c788","name":"valor","func":"var altText = msg.payload.alt\nvar value = 0\n\n// Possiveis Valores:\n// verde\n// amarela\n// bandeira-vermelha\nswitch(altText) {\n    case \"verde\":\n        valued = 0\n        break;\n    case \"amarela\":\n        value = 0.015\n        break;\n    case \"bandeira-vermelha\":\n        value = 0.060\n        break;\n}\nflow.set(\"flag\", value);\n\nmsg.topic = \"flag\"\nmsg.payload = value\nreturn msg;","outputs":1,"noerr":0,"x":550,"y":400,"wires":[["34251f38.944a8"]]},{"id":"43a567f.fafd998","type":"html","z":"6c3468f3.75c788","name":"bandeira","property":"payload","outproperty":"payload","tag":"#groupDestaque > div.boxDestaque.tipoAgweb > table > tbody > tr:nth-child(1) > td:nth-child(4) > p:nth-child(3) > strong > span > strong > a > img","ret":"attr","as":"multi","x":420,"y":400,"wires":[["abfb297b.46c3d8"]]},{"id":"34251f38.944a8","type":"http request","z":"6c3468f3.75c788","name":"tarifa celesc","method":"GET","ret":"txt","paytoqs":false,"url":"http://www.celesc.com.br/portal/index.php/duvidas-mais-frequentes/1140-tarifa","tls":"","proxy":"","authType":"basic","x":290,"y":460,"wires":[["3798c67.ca40c3a"]]},{"id":"3798c67.ca40c3a","type":"html","z":"6c3468f3.75c788","name":"taxas","property":"payload","outproperty":"payload","tag":"#contentMain > div > table:nth-child(25)","ret":"html","as":"multi","x":430,"y":460,"wires":[["284ac6c.adc473a"]]},{"id":"284ac6c.adc473a","type":"function","z":"6c3468f3.75c788","name":"valor","func":"var require = global.get('require');\nvar DOMParser = require('xmldom').DOMParser;\nvar moment = require('moment');\nlet parser = new DOMParser();\n\nlet table = parser.parseFromString(\"<table>\" + msg.payload + \"</table>\", \"text/xml\").getElementsByTagName(\"table\")[0];\n\nvar elements = table.lastChild.childNodes;\nlet currentDate = moment().format('MM/YYYY')\n\nvar msgOne = {topic: \"pis\", payload: {}}\nvar msgTwo = {topic: \"confins\", payload: {}}\n\nfor (let i = 0; i < elements.length; i++) {\n    let elementA = elements[i];\n    if (elementA.hasChildNodes()) {\n        let element = elementA.childNodes\n        \n        if (element[1].textContent == currentDate) { // get current date\n            msgOne.payload = parseFloat(element[3].textContent.replace(\",\", \".\"))\n            msgTwo.payload = parseFloat(element[5].textContent.replace(\",\", \".\"))\n        }\n    }\n}\n\n\nflow.set(\"pis\", msgOne.payload);\nflow.set(\"confins\", msgTwo.payload);\n\nreturn [msgOne, msgTwo];","outputs":2,"noerr":0,"x":550,"y":460,"wires":[[],["7e56e24a.a86fcc"]]},{"id":"7e56e24a.a86fcc","type":"function","z":"6c3468f3.75c788","name":"tarifa atual","func":"let pis = flow.get(\"pis\");\nlet confins = flow.get(\"confins\");\nlet flag = flow.get(\"flag\");\nlet base = 0.52049;\nlet icmsFirst = 12;\nlet icmsSecond = 25;\n\n// (base+flag)/((100-(pis+confins+icms))/100)\n\nmsgOne = {\n    topic: \"home/power/cost1\",\n    payload: (base+flag)/((100-(pis+confins+icmsFirst))/100)\n}\n\nmsgTwo = {\n    topic: \"home/power/cost2\",\n    payload: (base+flag)/((100-(pis+confins+icmsSecond))/100)\n}\n\nreturn [msgOne, msgTwo];","outputs":2,"noerr":0,"x":290,"y":520,"wires":[["46ee30d1.1fc48"],["46ee30d1.1fc48"]]},{"id":"46ee30d1.1fc48","type":"mqtt out","z":"6c3468f3.75c788","name":"","topic":"","qos":"1","retain":"true","broker":"e1a868b9.a931c8","x":430,"y":520,"wires":[]},{"id":"e1a868b9.a931c8","type":"mqtt-broker","z":"","name":"","broker":"localhost","port":"1883","clientid":"","usetls":false,"compatmode":true,"keepalive":"60","cleansession":true,"birthTopic":"","birthQos":"0","birthPayload":"","closeTopic":"","closeQos":"0","closePayload":"","willTopic":"","willQos":"0","willPayload":""}]
```

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
        value_template: "{{states('sensor.power_cost_less_150kwh')|float if states('sensor.power_meter_energy_total')|float < 150 else states('sensor.power_cost_more_150kwh')|float}}"
      total_power_cost:
        friendly_name: "Power Cost"
        unit_of_measurement: 'R$'
        value_template: "{% if states('sensor.power_meter_energy_total')|float < 150 %} {{'%.2f'|format(states('sensor.power_meter_energy_total')|float * states('sensor.power_cost_less_150kwh')|float)}} {%- else -%} {{'%.2f'|format((150 * states('sensor.power_cost_less_150kwh')|float) + ((states('sensor.power_meter_energy_total')|float) - 150 * states('sensor.power_cost_more_150kwh')|float))}}{%- endif %}"
```
