# Frisquet Boiler for ESPHome

This ESPHome component allows communication between an ESPHome device
(ESP8266 or ESP32) and a [Frisquet](<https://www.frisquet.com/en>) heating boiler (equipped with Eco Radio System remote thermostat).

The solution developed is applicable to all Frisquet boilers marketed until 2012 and fitted with the Eco Radio System module. More recent boilers equipped with the Visio module are not compatible because Frisquet has since implemented encryption in its communication protocol.

The **Frisquet Boiler** component appears as a [Float Output](<https://esphome.io/components/output/>) device.

It is recommended to combine it with the **Heating Curve Climate** component also provided in this project. This [Climate](<https://esphome.io/components/climate/index.html>) component will offer temperature control using an outdoor temperature sensor. If needed, it is also possible to use an other type of Climate component, such as the [PID Climate](https://esphome.io/components/climate/pid.html?highlight=pid).

## Breaking change in version 1.5

Some parameters names and behaviour have changed in version 1.5. The parameters `heat_factor` and `offset` of the `heat_curve_climate`component have been replaced by `slope` and `shift`. Those terms are more commonly used by boiler manufacturers.

Whilst `slope` provides the same functionnality as `heat_factor`, `shift` is slightly different. One way to define `shift` is to take the `offset` value you were previously using and substract your usual setpoint temperature (`shift` = `offset` - `setpoint`). Negative values are accepted.

The same changes are applicable to the component [actions](<https://esphome.io/guides/automations.html?highlight=automation#actions>) and component [sensors](<#heat_curve_climate-sensor>).

## References

This work is strongly inspired from:

- [Décodage du signal Frisquet Eco Radio System](<https://antoinegrall.wordpress.com/decodage-frisquet-ers/>) (French)
- [Decoding the wireless heating control Vaillant CalorMatic 340f](<http://wiki.kainhofer.com/hardware/vaillantvrt340f>)
- [frisquet-arduino](<https://github.com/etimou/frisquet-arduino>)

and from the discussions held in this thread:

- [Régulation d'une chaudière Frisquet ECO radio System](<https://easydomoticz.com/forum/viewtopic.php?f=17&t=1486sid=d2f41ac68e5bab18fd412a192a21c2c4>) (French)

## Wiring

The ESPHome replaces the original Eco Radio System HF receiver and is conneted to the boiler main board through a micro-fit 4 socket.

| ESP32                 | Boiler Side         | Pin number |
| --------------------- | ------------------- |:----------:|
| GND                   | black wire          | 1          |
| Pin 21 (configurable) | yellow wire         | 2          |
| 5V                    | red wire (optional) | 3          |

**Micro-fit 4 pin out:**

![Micro-fit 4 pinout drawing](images/connector_4pin1_80px.png)

Defined viewing direction for the connector pin out:

- Receptable - *rear view*
- Header - *front view*

*Note*: It has been observed that the current supplied by the boiler main board is not sufficent to power the ESP32.

## Installation

The Frisquet ESPHome component concists in two components:

- `heat_curve_climate` a custom [Climate](<https://esphome.io/components/climate/index.html>) component that will control the boiler water setpoint based on external temperature measurement and ambiant temperature setpoint.
- `friquet_boiler` a custom [Float Output](<https://esphome.io/components/output/>) component that will actually communicate with the Frisquet boiler.

They can be installed using the [External Components](https://esphome.io/components/external_components) feature of ESPHome.

### Local

The complete `components` folder must be copied into your `esphome` configuration folder and the YAML configuration file must show the following lines:

```yaml
external_components:
  - source: components
```

### Git

With this method, you don't have to copy the files onto your system. Instead, the configuration file will show the following lines:

```yaml
external_components:
  - source: github://philippemezzadri/frisquet-esphome
```

## Frisquet Boiler Output

The core component allowing communication with the boiler control board is a [Float Output](<https://esphome.io/components/output/>) component:

```yaml
output:
  - platform: frisquet_boiler
    id: boiler_cmd
    pin: 21
    boiler_id: 03B9
```

Configuration variables:

- **id** (**Required**, [ID](<https://esphome.io/guides/configuration-types.html#config-id>)): The id to use for this output component.
- **pin** (**Required**, [Pin](<https://esphome.io/guides/configuration-types.html#config-pin>)): The pin number connected to the boiler.
- **boiler_id** (**Required**, string): The identifier of your boiler (see below).
- **calibration_factor** (*Optional*, float): Calibration factor of the boiler. Defaults to `1.9`.
- **calibration_offset** (*Optional*, float): Calibration offset of the boiler. Defaults to `-41`.
- All other options from [Float Output](<https://esphome.io/components/output/>).

If `min_power`is set to a value that is not zero, it is important to set `zero_means_zero` to `true`. This can be safely ignored if `min_power` and `max_power` are kept at their default values.

`calibration_factor` and  `calibration_offset` are used by the internal sensor to calculate the water flow temperature. The default values have been defined on a *Frisquet Hydroconfort Evolution* boiler.

The output value received by the component is any rational value between `0` and `1`. Internaly, the output value is multiplied by 100 and rounded to an integer value because the Frisquet Boiler only accepts orders as integers between 0 and 100:

- 0 : boiler is stopped
- 10 : water pump starts, no heating
- 11 - 100 : water heating
- 15 : for some reason, the value is not accepted by the boiler. Internally, 15 is converted to 16 to avoid this case.

**Important:** the boiler ID that must be indicated in the YAML configuration file is required to allow
your boiler to receive the messages from the ESP. This ID can be retrieved by connecting the radio receiver signal wire to an Arduino.
See [here](https://github.com/etimou/frisquet-arduino) for more details.

*Note:* The ``frisquet_boiler`` component will send commands to the boiler right after an update of the ``output``value and then every 4 minutes. The component must receive regularly updates from the Climate component. To prevent overheating of the boiler, it will stop sending commands to the boiler if the ``output`` value is not updated during 15 minutes. In such case, the boiler will put itself in safe mode.

## Heating Curve Climate

In addition, a [Climate](<https://esphome.io/components/climate/index.html>) component is necessary to control the output. The [PID Climate](https://esphome.io/components/climate/pid.html?highlight=pid) could be used but it does not provide
smooth control and does not anticipate weather evolution.

It is otherwise recommended to use the **Heating Curve Climate** which adjusts the heating power according to the outdoor temperature.

```yaml
climate:
  - platform: heat_curve_climate
    id: boiler_climate
    name: "Chaudière Frisquet"
    sensor: current_temperature
    outdoor_sensor: outdoor_temperature
    default_target_temperature: 19
    output: boiler_cmd
    control_parameters:
      slope: 1.45
      shift: 3
      kp: 5
    output_parameters:
      minimum_output: 0.1
      output_factor: 1.9
      output_offset: -41
```

Configuration variables:

- **sensor** (**Required**, [ID](<https://esphome.io/guides/configuration-types.html#config-id>)): The sensor that is used to measure the current temperature.
- **outdoor_sensor** (**Required**, [ID](<https://esphome.io/guides/configuration-types.html#config-id>)): The sensor that is used to measure the outside temperature.
- **default_target_temperature** (**Required**, float): The default target temperature (setpoint) for the control algorithm. This can be dynamically set in the frontend later.
- **output** (**Required**, [ID](<https://esphome.io/guides/configuration-types.html#config-id>)): The ID of a float output that increases the current temperature.
- **control_parameters** (*Optional*): Control parameters of the controller (see [below](<#heating-curve-definition>)).
  - **alt_curve** (*Optional*, boolean): Set to `true` to use an alternate heating curve. Defaults to `false`.
  - **slope** (*Optional*, float): The proportional term (slope) of the heating curve. Defaults to `1.5`.
  - **shift** (*Optional*, float): The parallel shift term of the heating curve. Defaults to `0`.
  - **max_error** (*Optional*, float): The regulation error above which the boiler stops (e.g. when the ambiant temperature is too high because of external heat inputs). Defaults to `1`.
  - **min_delta** (*Optional*, float): The target/outdoor temperature difference below which the boiler stops. Defaults to `2`.
  - **kp** (*Optional*, float): The factor for the proportional term of the heating curve. May be useful for accelerating convergence to target temperature. Defaults to `0`.
  - **ki** (*Optional*, float): The factor for the integral term of the heating curve. May be useful if target temperature can't be reached. Use with caution when the house has a lot of thermal inertia. Defaults to `0`.
- **output_parameters** (*Optional*): Output parameters of the controller (see [below](<#setpoint-calibration-factors>)).
  - **rounded** (*Optional*, boolean): Forces rounding of the output value to two digits. Defaults to `false`.
  - **minimum_output** (*Optional*, float): Output value below which output value is set to zero. Defaults to `0.1`.
  - **maximum_output** (*Optional*, float): Output value above which output value won't go (cap). Defaults to `1`.
  - **heat_required_output** (*Optional*, float): Minimum output value to be considered when the [*Heat Required* switch](#heat_curve_climate-switch) is on.  Defaults to `0.1`.
  - **output_factor** (*Optional*, float): Calibration factor of the output. Defaults to `1`.
  - **output_offset** (*Optional*, float): Calibration offset of the output. Defaults to `0`.
- All other options from [Climate](<https://esphome.io/components/climate/index.html#config-climate>)

### Heating curve definition

The boiler flow temperature is calculated from the outdoor temperature:

`WATERTEMP` = `slope` \* `DELTA` + `target temperature` + `shift`

where :

- `WATERTEMP` is the temperature setpoint for the water circulating in the heating circuit.
- `DELTA` is the temperature difference between the target and the outdoor,
- `slope` and `shift` are defined in the Climate `control_parameters`.

![heat curve example graph](images/heat_curve_graph.png)

In this example, heating curves are given for an ambiant temperature (target) of 20°C with no shift. The `shift`parameter allows you to move up and down the curves by a few degrees.

`slope`and `shift`strongly depend on the heat insulation of the house. Therefore slight adjustments may be necessary to find the best settings. Guidelines to do so can be found [here](https://blog.elyotherm.fr/2013/08/reglage-optimisation-courbe-de-chauffe.html) (French).
In order to ease the fine tuning of those parameters, a service can be set in Home Assistant to change the parameters without restarting ESPHome ([see below](<#integration-with-home-assistant>)).

The following standard values for the `slope` may be used as a guide:

- 0.3 to 0.5 in a well insulated house with underfloor heating
- 1.0 to 1.2 for a well insulated house with radiators
- 1.4 to 1.6 for an older, detached building with radiators

If you don't know how to start, you can use the following values:

```yaml
control_parameters:
  slope: 1.5
  shift: 0
  kp: 2
```

**Alternate heating curve:**

If you struggle in finding the good `slope`and `shift`, you can try to set `alt_curve` to `true`. You can do it especially if you can't find settings that work for both cold winter and spring. The alternate heating curve is not linear like the standard curve but is polynomial and is designed to show a reduced slope for high delta between the outdoor and target temperatures.

![Graph of alternate heating curve](images/alternate_heating_curve.png)

In the above example, both curves have the same `slope` parameter.

### Proportionnal and integral terms

If needed, proportionnal and integral terms can be added to the heating curve:

`WATERTEMP` =  `HEATING_CURVE_TEMP` + `ERROR`* `kp` + `INTEGRAL_TERM`

where :

- `WATERTEMP` is the temperature setpoint for the water circulating in the heating circuit.
- `HEATING_CURVE_TEMP`is the heating curve temperature calculated above.
- `ERROR` is the calculated error (target - current)
- `INTEGRAL_TERM` is the cumulative sum of `ki` \* `ERROR` \* `dt`
- `dt` is the time difference in seconds between two calculations.
- `kp` and `ki` are defined in the Climate `control_parameters`.

**Warning:**

Setting a proportionnal factor `kp` can be useful to accelerate the convergence when the target temperature is changed. The value of `kp` should remain low to maintain the stability of the system and avoid overshoots.

However, setting an integral factor `ki` can be tricky to use and depends on many factors such as the house thermal inertia. We do not recommend to use it unless you know what you are doing.

### Hysteresis

In some instances, the boiler may go on idle mode because the ambiant temperature exceeds the maximum limit or if the outdoor temperature is too high. This is controlled by the `max_error`, `min_delta` and `minimum_output` settings.

If the above conditions disappear, the boiler will be allowed to restart only if the ambiant temperature goes below the target.

### Setpoint calibration factors

The boiler `SETPOINT` (integer in the `[0 - 100]` range) and the water flow temperature (`WATERTEMP`) are linked by the following formula:

`SETPOINT` = `WATERTEMP` * `output_factor` + `output_offset`

The actual value sent to the Output component is: `RESULT`= `SETPOINT` / 100

`output_factor` and `output_offset` are defined in the Climate `output_parameters`.
The following values seem to work well on **Frisquet Hydromotrix** and **Hydroconfort** boilers:

```yaml
output_parameters:
  output_factor: 1.9
  output_offset: -41
```

## Temperature Sensors

To get the Climate component working, two temperature sensors are required. They can be retrieved using [`homeassistant`](<https://esphome.io/components/sensor/homeassistant.html>) sensors:

```yaml
sensor:
  - platform: homeassistant
    id: current_temperature
    entity_id: sensor.living_room_temperature
    unit_of_measurement: "°C"
    filters:
      - filter_out: nan
      - heartbeat: 60s
        
  - platform: homeassistant
    id: outdoor_temperature
    entity_id: sensor.outdoor_temperature
    unit_of_measurement: "°C"
    filters:
      - filter_out: nan
      - exponential_moving_average:
          alpha: 0.3
          send_every: 1
      - heartbeat: 60s
```

If you are not using Home Assistant, you can use any local temperature sensor connected to the ESP or retrieve other sensor data using [`mqtt_subscribe`](<https://esphome.io/components/sensor/mqtt_subscribe.html>) sensors.

*Note:* Sensors should have a regular update interval as the heat curve update frequency is tied to the update interval of the sensors. We recommend putting a filter on the sensors to filter out the noise to ensure better stability of the output.

## `heat_curve_climate` Switch

On some occasions, external temperature conditions or high values of the Proportional and Integral factors may cause the boiler to enter idle mode (in accordance with `max_error`, `min_delta` and `minimum_output` settings). This can be undesirable as heat may be required by radiators in other rooms of the house.

To address this issue, the Heating Curve Climate platform provides a switch that will force the boiler to run at a minimum power level instead of shutting off completely.

This ensures that heat is still being supplied to the radiators and helps maintain a comfortable temperature throughout the house.

```yaml
switch:
  - platform: heat_curve_climate
    name: "Heat Required"
```

Configuration variables:

- **name** (**Required**, string): The name of the switch.

When the switch is on, the boiler will never go below the minimum power defined by the `heat_required_output` parameter.

## `frisquet_boiler` Sensor

Additionally, the **Frisquet Boiler** platform provides an optional sensor platform to monitor and give feedback from the Output component.

```yaml
sensor:
  - platform: frisquet_boiler
    name: "Boiler Flow Temperature"
    type: FLOWTEMP
```

Configuration variables:

- **name** (**Required**, string): The name of the sensor.
- **type** (**Required**, string): The value to monitor. One of
  - `SETPOINT` - The setpoint given the boiler (%).
  - `FLOWTEMP` - The water temperature resulting from the `SETPOINT`.

If the boiler is off, the flow temperature is unavailable.

## `heat_curve_climate` Sensor

The **Heating Curve Climate** platform also provides an optional sensor platform to monitor and give feedback from the Climate component.

```yaml
sensor:
  - platform: heat_curve_climate
    name: "Heating Curve Temperature"
    type: WATERTEMP
```

Configuration variables:

- **name** (**Required**, string): The name of the sensor.
- **type** (**Required**, string): The value to monitor. One of
  - `RESULT` - The resulting value sent to the output component (float between 0 and 1).
  - `SETPOINT` - The setpoint sent to the boiler (%, actually 100 * `RESULT`).
  - `WATERTEMP` - The calculated heating water temperature.
  - `DELTA` - The temperature difference between the target and the outdoor.
  - `ERROR` - The calculated error (target - process_variable)
  - `PROPORTIONAL` - The proportional term of the controller (if `kp` is not 0).
  - `INTEGRAL` - The integral term of the controller (if `ki` is not 0).
  - `SLOPE`- The current value of `slope`
  - `SHIFT`- The current value of `shift`
  - `KP`- The current value of `kp`
  - `KI`- The current value of `ki`

Those sensors may be useful to set up your heating curve `control_parameters`.

## `climate.heat_curve.set_control_parameters` Action

This [action](<https://esphome.io/guides/automations.html?highlight=automation#actions>) sets new values for the control parameters. This can be used to manually tune the controller. Make sure to update the values you want on the YAML file! They will reset on the next reboot.

```yaml
on_...:
  then:
    - climate.heat_curve.set_control_parameters:
        id: boiler_climate
        slope: 1.2
        shift: 1
        kp: 0
```

Configuration variables:

- **id** (**Required**, [ID](<https://esphome.io/guides/configuration-types.html#config-id>)): ID of the Heating Curve Climate.
- **heat_factor** (**Required**, float): The proportional term (slope) of the heating curve.
- **offset** (**Required**, float): The offset term of the heating curve.
- **kp** (*Optional*, float): The factor for the proportional term of the controller. Defaults to 0.
- **ki** (*Optional*, float): The factor for the integral term of the controller. Defaults to 0.

## `climate.pid.reset_integral_term` Action

This [action](<https://esphome.io/guides/automations.html?highlight=automation#actions>) resets the integral term of the PID controller to 0. This might be necessary under certain conditions to avoid the control loop to overshoot (or undershoot) a target.

```yaml
on_...:
  # Basic
  - climate.heat_curve.reset_integral_term: boiler_climate
```

Configuration variables:

- **id** (**Required**, [ID](<https://esphome.io/guides/configuration-types.html#config-id>)): ID of the Heating Curve Climate being reset.

## `boiler.set_mode` Action

This action sets the boiler operating mode.
This parameter is actually included in the frames sent to the boiler but I haven't seen any significant effect of the setting.

```yaml
on_...:
  then:
    - output.set_mode:
        id: boiler_cmd
        mode: 3
```

Configuration variables:

- **id** (**Required**, [ID](<https://esphome.io/guides/configuration-types.html#config-id>)): ID of the Frisquet Boiler Output.
- **mode** (**Required**, int): operating mode (0 = eco / 3 = confort / 4 = away)

## `output.set_level` Action

The `frisquet_boiler` Output component also inherits actions from [Float Output](<https://esphome.io/components/output/>) and in particular [`output.set_level`](<https://esphome.io/components/output/#output-set-level-action>).

This action sets the float output to the given level when executed. This can be usefull to set the boiler output if it is not connected to a Climate component.

```yaml
on_...:
  then:
    - boiler.set_level:
        id: boiler_cmd
        level: 50%
```

Configuration variables:

- **id** (**Required**, [ID](<https://esphome.io/guides/configuration-types.html#config-id>)): ID of the Frisquet Boiler Output.
- **level** (**Required**, percentage): output level

## Integration with Home Assistant

The Heating Curve Climate component automatically appears in Home Assistant as a [Climate](<https://www.home-assistant.io/integrations/climate/>) integration.

Also, when using the [native API](<https://esphome.io/components/api.html>) with Home Assistant, it is also possible to get data from Home Assistant to ESPHome with [user-defined services](<https://esphome.io/components/api.html#api-services>). When you declare services in your ESPHome YAML file, they will automatically show up in Home Assistant and you can call them directly.

This way it is possible to call the [Actions](<https://esphome.io/guides/automations.html?highlight=automation#actions>) provided by the Boiler Output and Heating Curve Climate components:

```yaml
# Example configuration entry
api:
  services:
    - service: set_boiler_setpoint
      variables:
        setpoint: int
      then:
        - output.set_level:
            id: boiler_cmd
            level: !lambda 'return setpoint / 100.0;'

    - service: set_boiler_mode
      variables:
        mode: int
      then:
        - boiler.set_mode:
            id: boiler_cmd
            mode: !lambda 'return mode;'

    - service: set_control_parameters
      variables:
        slope: float
        shift: float
        kp: float
      then:
        - climate.heat_curve.set_control_parameters:
            id: boiler_climate
            slope: !lambda 'return slope;'
            shift: !lambda 'return shift;'
            kp: !lambda 'return kp;'
            ki: !lambda 'return ki;'
        - climate.heat_curve.reset_integral_term: boiler_climate
```

Those lines in the YAML file will expose three [services](https://www.home-assistant.io/docs/scripts/service-calls/) in Home Assistant that can be called with the following lines (provided that the ESP device name is `myFrisquetBoiler`):

### Set climate control parameters

```yaml
service: esphome.myFrisquetBoiler_set_control_parameters
data:
  slope: 1.2
  shift: 3
  kp: 0
```

### Set boiler setpoint Service

```yaml
service: esphome.myFrisquetBoiler_set_boiler_setpoint
data:
  level: 50
```

### Set boiler mode Service

```yaml
service: esphome.myFrisquetBoiler_set_boiler_mode
data:
  mode: 3
```

Those are only examples. Any kind of service can be defined to suit your needs.

## Configuration files

The [boiler.yaml](boiler.yaml) file includes all options described above. To use it, you need to customize all sensors IDs and names. If you are using Dallas temperature sensor, you need to enter their proper addresses. If not, you have to delete the corresponding lines.

The [automations/boiler.yaml](automations/boiler.yaml) file is to be used in Home Assistant. It includes `input_number` and `automation` definitions that allow you to easily manage the `control_parameters` of the ESP. IDs and entity names should be changed before use to suit your own configuration.

The proposed automations allows you to modifiy the `output_parameters` from the Home Assistant UI and to restore them anytime the ESP reboots.

One way of using this file is to copy it in a folder named `packages`of your Home Assistant `config` folder and then add the following in your `configuration.yaml` file:

```yaml
homeassistant:
  packages: !include_dir_named packages
```
