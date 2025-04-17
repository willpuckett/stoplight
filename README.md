# Stoplight

![top_view](.images/stoplight_top.jpg)

Stoplight is a Xiao shield for operating small stepper motors. It is designed
specifically for 28-BYJ48, although it _may_ work with other small steppers as
well. It aims to provide utility all around the home, and is especially useful
when working with blinds and knobs.

![rear pcb view](.images/stoplight_bottom.jpg)

Commercial stepsticks have sense resistors that are not properly sized for
'smaller' motors to take advantage of many of TMC2209's features, including
stallguard.

> StopLight brings stall sensing to commodity/low cost, high torque motors.

Stoplight seeks to remedy these issues, while also providing software selectable
voltage control and temperature measurement in a package that works well with
HomeAssistant/ESPHome.

## Homing

![homing demo](.images/homing.gif)

## Fabrication

Use the [production files](jlcpcb/production_files/) to upload to jlc.

You'll need to swap the motor wires to a JST XH 4 pin housing. In addition to
removing the red cable, you also **must** perform the
[bipolar mod](https://ardufocus.com/howto/28byj-48-bipolar-hw-mod/) on your
steppers.

## Example HomeAssistant Config

The build sometimes fails, but can usually be completed by repeatedly pressing
'Retry', or switching to 'arduino' framework.

This demo exposes number to make tuning StallGuard easier.

```yaml
substitutions:
  name: "stoplight"
  friendly_name: "StopLight"
  board: "seeed_xiao_esp32c3"

# board/wifi/ota stuff
packages:
  device_base: !include .esp32.yaml

esp32:
  framework:
    type: esp-idf

logger:
  level: DEBUG

esphome:
  on_boot:
    - tmc2209.configure:
        direction: cw # or ccw
        microsteps: 1
        interpolation: True
        enable_spreadcycle: False
        tcool_threshold: 0
        tpwm_threshold: 0
    - tmc2209.stallguard:
        threshold: 23
    - tmc2209.currents:
        standstill_mode: freewheeling
        run_current: 0.150 # was 0.240
        hold_current: 0
    - button.press: home # home right after boot

external_components:
  - source: github://slimcdk/esphome-custom-components
    components: [tmc2209_hub, tmc2209, stepper]
  - source: github://pr#6693
    components: [husb238]

globals:
  - id: has_homed
    type: bool
    initial_value: "true"
    restore_value: no
  - id: homing_speed
    type: int
    initial_value: "480"
    restore_value: True

web_server:
  port: 80

i2c:
  sda: GPIO6
  scl: GPIO7
  frequency: 800kHz

uart:
  tx_pin: GPIO21
  rx_pin: GPIO20
  baud_rate: 500000 # 9600 -> 500k

one_wire:
  - platform: gpio
    pin: GPIO10

stepper:
  - platform: tmc2209
    id: driver
    # https://www.klipper3d.org/TMC_Drivers.html#limitations
    max_speed: 2000 steps/s # was 4000
    acceleration: 1000 steps/s^2 # was 2500
    deceleration: 1000 steps/s^2 # was 2500
    index_pin: GPIO3
    diag_pin: GPIO4
    rsense: 1000 mOhm
    config_dump_include_registers: False
    vsense: False
    analog_current_scale: False
    on_stall:
      - logger.log: "Motor stalled!"
      - if:
          condition:
            lambda: return !id(has_homed);
          then:
            - stepper.stop: driver
            - tmc2209.configure:
                direction: cw # or ccw
                microsteps: 1
                interpolation: True
                enable_spreadcycle: False
                tcool_threshold: 0
            - stepper.report_position:
                id: driver
                position: 0
            - globals.set:
                id: has_homed
                value: "true"
            - logger.log: "Home position set"
            - tmc2209.disable
    on_status:
      - logger.log:
          format: "Driver is reporting an update! (code %d)"
          args: ["code"]
      - if:
          condition:
            lambda: return code == tmc2209::DIAG_TRIGGERED;
          then:
            - logger.log: DIAG_TRIGGERED
      - if:
          condition:
            lambda: return code == tmc2209::DIAG_TRIGGER_CLEARED;
          then:
            - logger.log: DIAG_TRIGGER_CLEARED
      - if:
          condition:
            lambda: return code == tmc2209::RESET;
          then:
            - logger.log: RESET
      - if:
          condition:
            lambda: return code == tmc2209::RESET_CLEARED;
          then:
            - logger.log: RESET_CLEARED
      - if:
          condition:
            lambda: return code == tmc2209::DRIVER_ERROR;
          then:
            - logger.log: DRIVER_ERROR
      - if:
          condition:
            lambda: return code == tmc2209::DRIVER_ERROR_CLEARED;
          then:
            - logger.log: DRIVER_ERROR_CLEARED
      - if:
          condition:
            lambda: return code == tmc2209::CP_UNDERVOLTAGE;
          then:
            - logger.log: CP_UNDERVOLTAGE
      - if:
          condition:
            lambda: return code == tmc2209::CP_UNDERVOLTAGE_CLEARED;
          then:
            - logger.log: CP_UNDERVOLTAGE_CLEARED
      - if:
          condition:
            lambda: return code == tmc2209::OVERTEMPERATURE_PREWARNING;
          then:
            - logger.log: OVERTEMPERATURE_PREWARNING
      - if:
          condition:
            lambda: return code == tmc2209::OVERTEMPERATURE_PREWARNING_CLEARED;
          then:
            - logger.log: OVERTEMPERATURE_PREWARNING_CLEARED
      - if:
          condition:
            lambda: return code == tmc2209::OVERTEMPERATURE;
          then:
            - logger.log: OVERTEMPERATURE
      - if:
          condition:
            lambda: return code == tmc2209::OVERTEMPERATURE_CLEARED;
          then:
            - logger.log: OVERTEMPERATURE_CLEARED
      - if:
          condition:
            lambda: return code == tmc2209::TEMPERATURE_ABOVE_120C;
          then:
            - logger.log: TEMPERATURE_ABOVE_120C
      - if:
          condition:
            lambda: return code == tmc2209::TEMPERATURE_BELOW_120C;
          then:
            - logger.log: TEMPERATURE_BELOW_120C
      - if:
          condition:
            lambda: return code == tmc2209::TEMPERATURE_ABOVE_143C;
          then:
            - logger.log: TEMPERATURE_ABOVE_143C
      - if:
          condition:
            lambda: return code == tmc2209::TEMPERATURE_BELOW_143C;
          then:
            - logger.log: TEMPERATURE_BELOW_143C
      - if:
          condition:
            lambda: return code == tmc2209::TEMPERATURE_ABOVE_150C;
          then:
            - logger.log: TEMPERATURE_ABOVE_150C
      - if:
          condition:
            lambda: return code == tmc2209::TEMPERATURE_BELOW_150C;
          then:
            - logger.log: TEMPERATURE_BELOW_150C
      - if:
          condition:
            lambda: return code == tmc2209::TEMPERATURE_ABOVE_157C;
          then:
            - logger.log: TEMPERATURE_ABOVE_157C
      - if:
          condition:
            lambda: return code == tmc2209::TEMPERATURE_BELOW_157C;
          then:
            - logger.log: TEMPERATURE_BELOW_157C
      - if:
          condition:
            lambda: return code == tmc2209::OPEN_LOAD;
          then:
            - logger.log: OPEN_LOAD
      - if:
          condition:
            lambda: return code == tmc2209::OPEN_LOAD_CLEARED;
          then:
            - logger.log: OPEN_LOAD_CLEARED
      - if:
          condition:
            lambda: return code == tmc2209::OPEN_LOAD_A;
          then:
            - logger.log: OPEN_LOAD_A
      - if:
          condition:
            lambda: return code == tmc2209::OPEN_LOAD_A_CLEARED;
          then:
            - logger.log: OPEN_LOAD_A_CLEARED
      - if:
          condition:
            lambda: return code == tmc2209::OPEN_LOAD_B;
          then:
            - logger.log: OPEN_LOAD_B
      - if:
          condition:
            lambda: return code == tmc2209::OPEN_LOAD_B_CLEARED;
          then:
            - logger.log: OPEN_LOAD_B_CLEARED
      - if:
          condition:
            lambda: return code == tmc2209::LOW_SIDE_SHORT;
          then:
            - logger.log: LOW_SIDE_SHORT
      - if:
          condition:
            lambda: return code == tmc2209::LOW_SIDE_SHORT_CLEARED;
          then:
            - logger.log: LOW_SIDE_SHORT_CLEARED
      - if:
          condition:
            lambda: return code == tmc2209::LOW_SIDE_SHORT_A;
          then:
            - logger.log: LOW_SIDE_SHORT_A
      - if:
          condition:
            lambda: return code == tmc2209::LOW_SIDE_SHORT_A_CLEARED;
          then:
            - logger.log: LOW_SIDE_SHORT_A_CLEARED
      - if:
          condition:
            lambda: return code == tmc2209::LOW_SIDE_SHORT_B;
          then:
            - logger.log: LOW_SIDE_SHORT_B
      - if:
          condition:
            lambda: return code == tmc2209::LOW_SIDE_SHORT_B_CLEARED;
          then:
            - logger.log: LOW_SIDE_SHORT_B_CLEARED
      - if:
          condition:
            lambda: return code == tmc2209::GROUND_SHORT;
          then:
            - logger.log: GROUND_SHORT
      - if:
          condition:
            lambda: return code == tmc2209::GROUND_SHORT_CLEARED;
          then:
            - logger.log: GROUND_SHORT_CLEARED
      - if:
          condition:
            lambda: return code == tmc2209::GROUND_SHORT_A;
          then:
            - logger.log: GROUND_SHORT_A
      - if:
          condition:
            lambda: return code == tmc2209::GROUND_SHORT_A_CLEARED;
          then:
            - logger.log: GROUND_SHORT_A_CLEARED
      - if:
          condition:
            lambda: return code == tmc2209::GROUND_SHORT_B;
          then:
            - logger.log: GROUND_SHORT_B
      - if:
          condition:
            lambda: return code == tmc2209::GROUND_SHORT_B_CLEARED;
          then:
            - logger.log: GROUND_SHORT_B_CLEARED

button:
  - platform: template
    name: Home
    id: home
    on_press:
      - logger.log: "Going home!"
      - tmc2209.configure:
          direction: cw # or ccw
          microsteps: 1
          interpolation: True
          enable_spreadcycle: False
          tcool_threshold: 104
          tpwm_threshold: 0
      # - lambda: id(driver)->write_register(TPWMTHRS, 0);
      # - lambda: id(driver)->write_register(TCOOLTHRS, 1048575);
      - globals.set:
          id: has_homed
          value: "false"
      - stepper.set_speed:
          id: driver
          # 60 to 300 rpm
          speed: !lambda return id(homing_speed);
      - stepper.set_target:
          id: driver
          target: -9999999
  - platform: template
    name: Stop
    id: stop
    on_press:
      - logger.log: "Stopping!"
      - stepper.stop: driver
      - tmc2209.disable
  - platform: template
    name: 1000 Steps forward
    on_press:
      - logger.log: "Going forward!"
      - stepper.set_speed:
          id: driver
          speed: 400 steps/s
      - stepper.set_target:
          id: driver
          target: !lambda return id(driver)->current_position +1000;
      # - tmc2209.disable

husb238:
  id: husb_01

binary_sensor:
  - platform: husb238
    attached: "PD Attached"

  - platform: template
    name: "Has Homed"
    lambda: return id(has_homed);

text_sensor:
  - platform: husb238
    status: "Last request status"
    capabilities: "Capabilities"

select:
  - platform: husb238
    voltage: "Voltage selector"

sensor:
  - platform: husb238
    voltage: "Contracted Voltage"
    current: "Contracted Current"
    selected_voltage: "Selected Voltage"

  - platform: dallas_temp
    name: temperature

  - platform: tmc2209
    type: stallguard_result
    name: Driver stallguard
    update_interval: 10s

  - platform: tmc2209
    type: actual_current
    name: Actual current
    update_interval: 10s

  - platform: tmc2209
    type: motor_load
    name: Motor load
    update_interval: 10s

  # - platform: tmc2209
  #   type: pwm_scale_sum
  #   name: PWM Scale Sum
  #   update_interval: 250ms

  # - platform: tmc2209
  #   type: pwm_scale_auto
  #   name: PWM Scale Auto
  #   update_interval: 250ms

  # - platform: tmc2209
  #   type: pwm_ofs_auto
  #   name: PWM OFS Auto
  #   update_interval: 250ms

  # - platform: tmc2209
  #   type: pwm_grad_auto
  #   name: PWM Grad Auto
  #   update_interval: 250ms

number:
  - platform: template
    name: Target position
    min_value: -999999
    max_value: 999999
    step: 100
    lambda: return id(driver)->current_position;
    update_interval: 10s
    set_action:
      - stepper.set_target:
          id: driver
          target: !lambda "return x;"

  - platform: template
    name: Homing RPM
    min_value: 0
    max_value: 300
    step: 1
    lambda: return id(homing_speed) * 60 / 32;
    update_interval: 10s
    set_action:
      - globals.set:
          id: homing_speed
          value: !lambda "return x * 32 / 60;"

  - platform: template
    name: SGTHRS
    min_value: 0
    max_value: 255
    step: 1
    lambda: return id(driver)->read_register(SGTHRS);
    update_interval: 10s
    set_action:
      - lambda: id(driver)->write_register(SGTHRS, x);

  - platform: template
    name: TCOOLTHRS
    min_value: 0
    max_value: 1048575
    step: 1
    lambda: return id(driver)->read_register(TCOOLTHRS);
    update_interval: 10s
    set_action:
      - lambda: id(driver)->write_register(TCOOLTHRS, x);

  - platform: template
    name: "Run Current"
    min_value: 0
    max_value: 31
    step: 1
    lambda: return id(driver)->read_field(IRUN_FIELD);
    update_interval: 10s
    set_action:
      - tmc2209.currents:
          irun: !lambda "return x;"
```

## Schematic

![schematic](.images/stoplight.svg)

## Reference

[TMC2209 Calaulations Worksheet](https://www.analog.com/media/en/engineering-tools/design-tools/TMC2209_Calculations.xlsx)

[BTT-TMC2209-v1.2](https://pax.deno.dev/bigtreetech/BIGTREETECH-TMC2209-V1.2@master/Schematic/TMC2209-V1.2.pdf?b)

[Cookie Robotics](https://cookierobotics.com/042/)

[jangeox](http://www.jangeox.be/2013/10/change-unipolar-28byj-48-to-bipolar.html)

