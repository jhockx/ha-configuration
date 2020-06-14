# Gas Water Light (GWL) monitoring
My Home Assistant is configured to get information about electricity usage, gas usage and solar panels. I will explain my setup to read out my Dutch Smart Meter (DSMR5.0), solar panels (SMA) and how to combine these two to give proper insights in your electricity consumption an delivery. With this setup, I'm also sending my gas usage to [mindergas.nl](https://mindergas.nl/) via a REST API call.

## Solar panels
My solar panels use a SMA inverter. You can easily readout the SMA inverter by using the [SMA Solar integration](https://www.home-assistant.io/integrations/sma/). Add this to `configuration.yaml`:

```
sensor:
  - platform: sma
    host: IP_ADDRESS_OF_DEVICE
    ssl: true
    verify_ssl: false
    password: YOUR_SMA_PASSWORD
    sensors:
      - pv_power
      - total_yield
```

## Electricity/gas usage from the DSMR5.0
Home Assistant is setup as a MQTT broker by just installing the [Mosquitto MQTT broker addon](https://github.com/home-assistant/hassio-addons/blob/master/mosquitto/README.md). Hooked up to the DSMR5.0 is an esp8266 as a MQTT sensor. The esp8266 reads out the P1 port and sends this to the MQTT broker. To setup the esp8266, checkout my [other repository](https://github.com/jhockx/esp8266_p1meter). To subscripe to the MQTT topics from the esp8266 and add them as sensors in your Home Assistant, add the following to `configuration.yaml`:

```
sensor:
  - platform: mqtt
    name: P1 Consumption Low Tarif
    unit_of_measurement: 'kWh'
    state_topic: "sensors/power/p1meter/consumption_low_tarif"
    value_template: "{{ value|float / 1000 }}"
    icon: mdi:flash

  - platform: mqtt
    name: P1 Consumption High Tarif
    unit_of_measurement: 'kWh'
    state_topic: "sensors/power/p1meter/consumption_high_tarif"
    value_template: "{{ value|float / 1000 }}"
    icon: mdi:flash

  - platform: mqtt
    name: P1 Delivered Low Tarif
    unit_of_measurement: 'kWh'
    state_topic: "sensors/power/p1meter/delivered_low_tarif"
    value_template: "{{ value|float / 1000 }}"
    icon: mdi:flash

  - platform: mqtt
    name: P1 Delivered High Tarif
    unit_of_measurement: 'kWh'
    state_topic: "sensors/power/p1meter/delivered_high_tarif"
    value_template: "{{ value|float / 1000 }}"
    icon: mdi:flash

  - platform: mqtt
    name: P1 Actual Power Consumption
    unit_of_measurement: 'W'
    state_topic: "sensors/power/p1meter/actual_consumption"
    value_template: "{{ value|int }}"
    icon: mdi:flash
    expire_after: 30
    
  - platform: mqtt
    name: P1 Actual Power Delivered
    unit_of_measurement: 'W'
    state_topic: "sensors/power/p1meter/actual_received"
    value_template: "{{ value|int }}"
    icon: mdi:flash
    expire_after: 30

  - platform: mqtt
    name: P1 Instant Power Usage
    unit_of_measurement: 'W'
    state_topic: "sensors/power/p1meter/instant_power_usage_l1"
    value_template: "{{ value|int }}"
    icon: mdi:flash
    expire_after: 30

  - platform: mqtt
    name: P1 Instant Power Current
    unit_of_measurement: 'A'
    state_topic: "sensors/power/p1meter/instant_power_current_l1"
    value_template: "{{ value|float / 1000 }}"
    icon: mdi:counter
    
  - platform: mqtt
    name: P1 Instant Power Voltage
    unit_of_measurement: 'V'
    state_topic: "sensors/power/p1meter/instant_voltage_l1"
    value_template: "{{ value|float / 1000 }}"
    icon: mdi:counter

  - platform: mqtt
    name: P1 Gas Usage
    unit_of_measurement: 'm3'
    state_topic: "sensors/power/p1meter/gas_meter_m3"
    value_template: "{{ value|float / 1000 }}"
    icon: mdi:fire

  - platform: mqtt
    name: P1 Actual Tarif Group
    state_topic: "sensors/power/p1meter/actual_tarif_group"
    icon: mdi:counter

  - platform: mqtt
    name: P1 Short Power Outages
    state_topic: "sensors/power/p1meter/short_power_outages"
    icon: mdi:counter

  - platform: mqtt
    name: P1 Long Power Outages
    state_topic: "sensors/power/p1meter/long_power_outages"
    icon: mdi:counter

  - platform: mqtt
    name: P1 Short Power Drops
    state_topic: "sensors/power/p1meter/short_power_drops"
    icon: mdi:counter

  - platform: mqtt
    name: P1 Short Power Peaks
    state_topic: "sensors/power/p1meter/short_power_peaks"
    icon: mdi:counter
```

## Using the utility meter
Now the sensors are all up and running! The only problem is that I didn't necessarily want to monitor the cumulative history of my electricity usage, gas usage and solar panels. I actually wanted to see this on a daily, monthly and yearly basis. This is where the [Utility Meter](https://www.home-assistant.io/integrations/utility_meter/) integration comes in. Add this to `configuration.yaml`:

```
utility_meter:
  # Solar energy
  daily_yield:
    source: sensor.total_yield
    cycle: daily
  monthly_yield:
    source: sensor.total_yield
    cycle: monthly
  yearly_yield:
    source: sensor.total_yield
    cycle: yearly
  
  # P1 meter
  daily_electricity_consumption_high_tarif:
    source: sensor.p1_consumption_high_tarif
    cycle: daily
    net_consumption: true
  monthly_electricity_consumption_high_tarif:
    source: sensor.p1_consumption_high_tarif
    cycle: monthly
    net_consumption: true
  yearly_electricity_consumption_high_tarif:
    source: sensor.p1_consumption_high_tarif
    cycle: yearly
    net_consumption: true
  
  daily_electricity_consumption_low_tarif:
    source: sensor.p1_consumption_low_tarif
    cycle: daily
    net_consumption: true
  monthly_electricity_consumption_low_tarif:
    source: sensor.p1_consumption_low_tarif
    cycle: monthly
    net_consumption: true
  yearly_electricity_consumption_low_tarif:
    source: sensor.p1_consumption_low_tarif
    cycle: yearly
    net_consumption: true
  
  daily_electricity_delivered_high_tarif:
    source: sensor.p1_delivered_high_tarif
    cycle: daily
    net_consumption: true
  monthly_electricity_delivered_high_tarif:
    source: sensor.p1_delivered_high_tarif
    cycle: monthly
    net_consumption: true
  yearly_electricity_delivered_high_tarif:
    source: sensor.p1_delivered_high_tarif
    cycle: yearly
    net_consumption: true
  
  daily_electricity_delivered_low_tarif:
    source: sensor.p1_delivered_low_tarif
    cycle: daily
    net_consumption: true
  monthly_electricity_delivered_low_tarif:
    source: sensor.p1_delivered_low_tarif
    cycle: monthly
    net_consumption: true
  yearly_electricity_delivered_low_tarif:
    source: sensor.p1_delivered_low_tarif
    cycle: yearly
    net_consumption: true
  
  daily_gas_usage:
    source: sensor.p1_gas_usage
    cycle: daily
    net_consumption: true
  monthly_gas_usage:
    source: sensor.p1_gas_usage
    cycle: monthly
    net_consumption: true
  yearly_gas_usage:
    source: sensor.p1_gas_usage
    cycle: yearly
    net_consumption: true
```

## Combining sensors
Now we can start combining and fixing some sensors. I wanted to:
- Fix the solar panel realtime readout: The SMA device stops sending out values during the night, which means `sensor.pv_power` will be "unknown" during this time, while it obviously is just zero.
- Fix some MQTT sensors that don't send zero values. By setting the `expire_after`, the sensor becomes "unknown" after some amount of time. For some sensors we know that "unknown" is actually zero.
- Display the current tarif from the P1 meter with words, not integer coded.
- See the total direct power usage
- See the total solar power delivered to the P1 meter (low + high tarif)
- See the total electricity consumption from the P1 meter (low + high tarif)
- See the total electricity consumption (low + high tarif + direct solar energy usage)

I have built these custom sensors using the [Template](https://www.home-assistant.io/integrations/template/) integration. Add this to `configuration.yaml`:

```
sensor:
  - platform: template
    sensors:
        # Solar sensors fixes
        pv_power_incl_0:
            friendly_name: 'Vermogen zonnepanelen'
            value_template: "{{ states.sensor.pv_power.state | replace('unknown','0') }}"
            icon_template: mdi:solar-power
            unit_of_measurement: W
        
        # P1 sensors fixes
        p1_instant_power_usage_incl_0:
            friendly_name: 'Vermogen levering'
            value_template: "{{ states.sensor.p1_instant_power_usage.state | replace('unknown','0') }}"
            icon_template: mdi:flash
            unit_of_measurement: W
        
        p1_actual_power_delivered_incl_0:
            friendly_name: 'Vermogen teruglevering'
            value_template: "{{ states.sensor.p1_actual_power_delivered.state | replace('unknown','0') }}"
            icon_template: mdi:flash
            unit_of_measurement: W
        
        # Total direct power usage P1 + direct power usage from solar
        power_usage_total:
            friendly_name: 'Vermogen verbruik'
            value_template: "{{ [states.sensor.pv_power_incl_0.state|int - states.sensor.p1_actual_power_delivered_incl_0.state|int + states.sensor.p1_instant_power_usage_incl_0.state|int, 0]|max }}"
            icon_template: mdi:flash
            unit_of_measurement: W
        
        # P1 sensors
        actual_tarif_group_text:
            friendly_name: 'Huidige tarief'
            value_template: "{{ states.sensor.p1_actual_tarif_group.state | replace('1', 'Laag') | replace('2', 'Hoog') }}"
            icon_template: mdi:counter
        
        # Total solar delivered P1 meter
        daily_electricity_delivered_total:
          friendly_name: Dag teruglevering totaal
          unit_of_measurement: kWh
          value_template: "{{ (states('sensor.daily_electricity_delivered_high_tarif')|float + states('sensor.daily_electricity_delivered_low_tarif')|float)|round(3) }}"
          icon_template: mdi:flash
        monthly_electricity_delivered_total:
          friendly_name: Maand teruglevering totaal
          unit_of_measurement: kWh
          value_template: "{{ (states('sensor.monthly_electricity_delivered_high_tarif')|float + states('sensor.monthly_electricity_delivered_low_tarif')|float)|round(3) }}"
          icon_template: mdi:flash
        yearly_electricity_delivered_total:
          friendly_name: Jaar teruglevering totaal
          unit_of_measurement: kWh
          value_template: "{{ (states('sensor.yearly_electricity_delivered_high_tarif')|float + states('sensor.yearly_electricity_delivered_low_tarif')|float)|round(3) }}"
          icon_template: mdi:flash
          
        # Total consumption P1 meter
        daily_electricity_consumption_total:
          friendly_name: Dag levering totaal
          unit_of_measurement: kWh
          value_template: "{{ (states('sensor.daily_electricity_consumption_high_tarif')|float + states('sensor.daily_electricity_consumption_low_tarif')|float)|round(3) }}"
          icon_template: mdi:flash
        monthly_electricity_consumption_total:
          friendly_name: Maand levering totaal
          unit_of_measurement: kWh
          value_template: "{{ (states('sensor.monthly_electricity_consumption_high_tarif')|float + states('sensor.monthly_electricity_consumption_low_tarif')|float)|round(3) }}"
          icon_template: mdi:flash
        yearly_electricity_consumption_total:
          friendly_name: Jaar levering totaal
          unit_of_measurement: kWh
          value_template: "{{ (states('sensor.yearly_electricity_consumption_high_tarif')|float + states('sensor.yearly_electricity_consumption_low_tarif')|float)|round(3) }}"
          icon_template: mdi:flash
        
        # Total usage P1 meter + direct use from solar
        daily_electricity_usage_total:
          friendly_name: Dag verbruik totaal
          unit_of_measurement: kWh
          value_template: "{{ [(states('sensor.daily_electricity_consumption_total')|float + states('sensor.daily_yield')|float - states('sensor.daily_electricity_delivered_total')|float)|round(3), 0]|max }}"
          icon_template: mdi:flash
        monthly_electricity_usage_total:
          friendly_name: Maand verbruik totaal
          unit_of_measurement: kWh
          value_template: "{{ [(states('sensor.monthly_electricity_consumption_total')|float + states('sensor.monthly_yield')|float - states('sensor.monthly_electricity_delivered_total')|float)|round(3), 0]|max }}"
          icon_template: mdi:flash
        yearly_electricity_usage_total:
          friendly_name: Jaar verbruik totaal
          unit_of_measurement: kWh
          value_template: "{{ [(states('sensor.yearly_electricity_consumption_total')|float + states('sensor.yearly_yield')|float - states('sensor.yearly_electricity_delivered_total')|float)|round(3), 0]|max }}"
          icon_template: mdi:flash
```

## Making sensors readable
To make your sensors more readable in Lovelace, add the following to `configuration.yaml`:

```
homeassistant:
  # Display names in a nice way
  customize:
    # Solar energy
    sensor.daily_yield:
      friendly_name: Dag opbrengst
      icon: mdi:solar-power
    sensor.monthly_yield:
        friendly_name: Maand opbrengst
        icon: mdi:solar-power
    sensor.yearly_yield:
        friendly_name: Jaar opbrengst
        icon: mdi:solar-power
    sensor.total_yield:
        friendly_name: Totale opbrengst
        icon: mdi:solar-power
    
    #P1 meter
    sensor.daily_electricity_consumption_high_tarif:
      friendly_name: Dag levering hoog tarief
      icon: mdi:flash
    sensor.monthly_electricity_consumption_high_tarif:
      friendly_name: Maand levering hoog tarief
      icon: mdi:flash
    sensor.yearly_electricity_consumption_high_tarif:
      friendly_name: Jaar levering hoog tarief
      icon: mdi:flash
      
    sensor.daily_electricity_consumption_low_tarif:
      friendly_name: Dag levering laag tarief
      icon: mdi:flash
    sensor.monthly_electricity_consumption_low_tarif:
      friendly_name: Maand levering laag tarief
      icon: mdi:flash
    sensor.yearly_electricity_consumption_low_tarif:
      friendly_name: Jaar levering laag tarief
      icon: mdi:flash
      
    sensor.daily_electricity_delivered_high_tarif:
      friendly_name: Dag teruglevering hoog tarief
      icon: mdi:flash
    sensor.monthly_electricity_delivered_high_tarif:
      friendly_name: Maand teruglevering hoog tarief
      icon: mdi:flash
    sensor.yearly_electricity_delivered_high_tarif:
      friendly_name: Jaar teruglevering hoog tarief
      icon: mdi:flash
    
    sensor.daily_electricity_delivered_low_tarif:
      friendly_name: Dag teruglevering laag tarief
    sensor.monthly_electricity_delivered_low_tarif:
      friendly_name: Maand teruglevering laag tarief
    sensor.yearly_electricity_delivered_low_tarif:
      friendly_name: Jaar teruglevering laag tarief
      
    sensor.daily_gas_usage:
      friendly_name: Dag verbruik gas
      icon: mdi:fire
    sensor.monthly_gas_usage:
      friendly_name: Maand verbruik gas
      icon: mdi:fire
    sensor.yearly_gas_usage:
      friendly_name: Jaar verbruik gas
      icon: mdi:fire
```

## Mindergas.nl
If you want to make use of [mindergas.nl](https://mindergas.nl/), add a REST API call with the [RESTful Command](https://www.home-assistant.io/integrations/rest_command/) integration. Add the following to `configuration.yaml`.

```
rest_command:
  mindergas:
    url: https://www.mindergas.nl/api/gas_meter_readings
    method: POST
    headers:
      AUTH-TOKEN: 'GET_YOUR_API_TOKEN_FROM_MINDERGAS.NL'
      Content-Type: 'application/json'
    payload: '{"date": "{{ now().date() }}", "reading": "{{ states.sensor.p1_gas_usage.state }}"}'
    verify_ssl: true
```

Setup an automation to call this REST command at the end of the day around midnight. To reduce the load on the website, I have set the time to make the API call to 23.52. Just in case mindergas.nl gets a lot of API calls exactly at midnight (00.00), you can also pick some time around midnight.
