# Solar Forecast Electricity Price for Home Assistant

The Solar Forecast Electricity Price Integration will calculate the electricity
price for a fixed load (for example, EV Charging) for homes with
a solar panel installation, taking into account the export tariff
as well as the cost of importing electricity, in combination
with a solar energy forecast. It will emit both a sensor value
for the price "now", and attributes with price calculations
for today/tomorrow in the [Generic Price Sensor](https://github.com/jonasbkarlsson/ev_smart_charging/wiki/Price-sensor)
format.

The question it's trying to answer is "is it cheaper to use my own
solar power now (maybe in combination with grid import) to run a certain load, 
or better to export the power and get paid for it and run the load later (i.e during the night)".

A common scenario is charging an electric vehicle when coming home
in the afternoon, where, depending on the prices now and the coming
hours it may make most economic sense to charge "now" with solar power
from your own panels, or sell the power to the grid and charge during
the night.

The primary intended use case for this integration is to function
as a price sensor for the [EV Smart Charging](https://github.com/jonasbkarlsson/ev_smart_charging)
integration. This will allow EV Smart Charging to make better decisions
on when to schedule the charging using not only the energy price now
and the coming hours, but also the forecasted energy production from the
solar panels.

## Installation

Ensure the [Prerequisites](#prerequisites) (Solar Forecast and Cost Sensors) are fulfilled as detailed below.

<!-- ### Option 1: HACS

- Follow [![Open your Home Assistant instance and open a repository inside the Home Assistant Community Store.](https://my.home-assistant.io/badges/hacs_repository.svg)](https://my.home-assistant.io/redirect/hacs_repository/?owner=forsberg&repository=solar_forecast_electricity_price&category=integration) and install it
- Restart Home Assistant

  *or*
- Go to `HACS` -> `Integrations`,
- In the top right corner, click on the three dots and choose Custom Repository
- Add https://github.com/forsberg/solar_forecast_electricity_price as an Integration Repository
- Select `+`,
- Search for `solar forecast electricity price` and install it,
- Restart Home Assistant -->

### Option 2: Manual

Download the [latest release](https://github.com/forsberg/solar_forecast_electricity_price/releases)

```bash
cd YOUR_HASS_CONFIG_DIRECTORY    # same place as configuration.yaml
mkdir -p custom_components/solar_forecast_electricity_prices
cd custom_components/solar_forecast_electricity_price
unzip solar_forecast_electricity_price-X.Y.Z.zip
mv solar_forecast_electricity_prices-X.Y.Z/custom_components/nordpool/* .
rm -r solar_forecast_electricity_prices-X.Y.Z
```

## Configuration - this integration

- Ensure [Prerequisites](#prerequisites) are in place when it comes to sensors and Solar Forecasts, see below.
- Go into Settings -> Devices & Services
- Click the "Add Integration" button and search for `Solar Forecast Electricity Price`
- Fill in the form
  - Name is the name of your load, and will form the basis of the name of the electricity cost sensor
  - Grid Import Cost Sensor is the sensor that provides the cost of importing electricity from the grid. 
  - Grid Export Cost Sensor is the sensor that provides the income from exporting electricity.
  - Power Draw should be set to the amount of power in Watts that your load (EV Charger) will use when
    turned on.
  - Solar Forecast Sensors should be set to all the solar forecast sensors you have added to your
    home assistant. There can be multiple if you have multiple orientations on your solar panels.

## Configuration - EV Smart Charging

- Configure/Reconfigure your EV Smart Charging integration to use the newly created sensor
  as electricity cost sensor.

## Prerequisites

### Solar Forecast

In order to calculate the cost of running the load the this integration needs a Solar Forecast. It has been tested
with [forecast.solar](https://www.home-assistant.io/integrations/forecast_solar/)
but should work with any solar forecast that shows up correctly in the energy 
integration dashboard.

Obviously, the calculations made by this integration will only ever be as good as the quality of the forecast.
Some days, the forecast will be wrong, but as long as it's at least roughly correct, the total cost
of running the load over time should decrease by using this integration.

### Price Sensors for Cost of Import and Income of Export

In order to calculate the true cost of running the load (i.e charge the car) by combining solar and
grid import, the integration needs to know the cost of importing electricity from the grid now and as
far into the future as possible, and also the income from exporting electricity to the grid.

Ensure there are two sensors in [Generic Price Sensor](https://github.com/jonasbkarlsson/ev_smart_charging/wiki/Price-sensor)
format available for the integration. Since there are many different variants of how your grid charges
you for electricity, and how you get payed for export (if at all), 

#### Grid Import Cost Sensor

This sensor should include all components that contribute to the price of importing
electricity, electricity price, taxes, grid cost. 

#### Grid Export Income Sensor

This sensor should include all components that contribute to the income per kWh 
when exporting to the grid.

#### Price Sensor Examples

Below is an example of a two template sensors

The first is a grid import cost sensor which take
the raw price for every 15m period from the [Nordpool Custom Component](https://github.com/custom-components/nordpool)
and adds 25% VAT, 0.06 SEK of fixed addition from electricity wholesaler, 0.45 SEK tax on electricity and 0.143 SEK
of grid import cost per kWh.

The second is a grid export income sensor that take the raw price from Nordpool, then add 0.042 SEK of "Grid usefulness"
reimbursement. 

Since the template expose attributes, it can't at the time of writing be added via the Helper UI, but must
instead be added to `configuration.yaml`

```yaml
template:
      - name: "Grid Import Cost"
        unique_id: grid_import_cost
        unit_of_measurement: "SEK/kWh"
        state_class: total
        device_class: monetary
        availability: >
          {{ has_value('sensor.nordpool_kwh_se3_sek_3_10_0') }}
        state: >
          {% set price = states('sensor.nordpool_kwh_se3_sek_3_10_0') | float %}
          {{ (price*1.25 + 0.06 + 0.45 + 0.143) | round(3) }}
        attributes:
          raw_today: >
            {% set raw = state_attr('sensor.nordpool_kwh_se3_sek_3_10_0', 'raw_today') %}
            {% if raw is iterable and raw is not string %}
              {% set ns = namespace(items=[]) %}
              {% for item in raw %}
                {% set new_item = {
                  'price': (item.value | float(0) * 1.25 + 0.06 + 0.45 + 0.143 ) | round(2),
                  'time': item.start.isoformat(),
                } %}
                {% set ns.items = ns.items + [new_item] %}
              {% endfor %}
              {{ ns.items | to_json }}
            {% else %}
              []
            {% endif %}
          raw_tomorrow: >
            {% set raw = state_attr('sensor.nordpool_kwh_se3_sek_3_10_0', 'raw_tomorrow') %}
            {% if raw is iterable and raw is not string %}
              {% set ns = namespace(items=[]) %}
              {% for item in raw %}
                {% set new_item = {
                  'price': (item.value | float(0) * 1.25 + 0.06 + 0.45 + 0.143 ) | round(2),
                  'time': item.start.isoformat(),
                } %}
                {% set ns.items = ns.items + [new_item] %}
              {% endfor %}
              {{ ns.items | to_json }}
            {% else %}
              []
            {% endif %}

  - sensor:
      - name: "Grid Export Income per kWh"
        unique_id: grid_export_income
        unit_of_measurement: "SEK/kWh"
        state_class: total
        device_class: monetary
        availability: >
          {{ has_value('sensor.nordpool_kwh_se3_sek_3_10_0') }}
        state: >
          {% set price = states('sensor.nordpool_kwh_se3_sek_3_10_0') | float %}
          {{ (price+ 0.042) | round(3) }}
        attributes:
          raw_today: >
            {% set raw = state_attr('sensor.nordpool_kwh_se3_sek_3_10_0', 'raw_today') %}
            {% if raw is iterable and raw is not string %}
              {% set ns = namespace(items=[]) %}
              {% for item in raw %}
                {% set new_item = {
                  'price': (item.value | float(0) + 0.042) | round(2),
                  'time': item.start.isoformat(),
                } %}
                {% set ns.items = ns.items + [new_item] %}
              {% endfor %}
              {{ ns.items | to_json }}
            {% else %}
              []
            {% endif %}
          raw_tomorrow: >
            {% set raw = state_attr('sensor.nordpool_kwh_se3_sek_3_10_0', 'raw_tomorrow') %}
            {% if raw is iterable and raw is not string %}
              {% set ns = namespace(items=[]) %}
              {% for item in raw %}
                {% set new_item = {
                  'price': (item.value | float(0) + 0.042) | round(2),
                  'time': item.start.isoformat(),
                } %}
                {% set ns.items = ns.items + [new_item] %}
              {% endfor %}
              {{ ns.items | to_json }}
            {% else %}
              []
            {% endif %}


```