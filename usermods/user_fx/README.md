# Usermod user FX

This Usermod is a common place to put various user's private LED Effects.  It gives you a way to load in your own custom WLED effects, or to load in depracated WLED effects that you want to bring back--all without having to mess with the core WLED source code.

Multiple Effects can be specified inside this single usermod, as we will illustrate below.  You will be able to define them with custom names, sliders, etc. as with any other Effect.


## How The Usermod Works

The `user_fx.cpp` file can be broken down into four main parts:
* **static effect definition** - This is a static LED setting that is displayed if an effect fails to initialize.
* User FX function definition(s) - This area is where you place the FX code for all of the custom effects you want to use.  This mainly includes the FX code and the static variable containing the [metadata string](https://kno.wled.ge/interfaces/json-api/#effect-metadata). 
* Usermod Class definition(s) - The class definition defines the blueprint from which all your custom Effects (or any usermod, for that matter) are created.
* Usermod registration - All usermods have to be registered so that they are able to be compiled into your binary.

We will go into greater detail on how custom effects work in the usermod and how to go abour creating your own in the section below.


## Understanding WLED Effects
 
Currently set in the code have the pixels be all black, but can be modified to take on a different behavior.  For example, say you wanted to...
TODO

### 

Pre-loaded in this template is an example 2D Effect called "Diffusion Fire", which is the name that would be shown in the UI once the binary is compiled and run on your device.  We can explore the anatomy of this effect below.



## Basic Understanding of C++ and WLED Effects



## Installation

## Compiling



## Change Log


(what could one do with mode_static?)



 
## Installation

Add `Temperature` to `custom_usermods` in your platformio_override.ini.

Example **platformio_override.ini**:

```ini
[env:usermod_temperature_esp32dev]
extends = env:esp32dev
custom_usermods = ${env:esp32dev.custom_usermods} 
  Temperature
```

### Define Your Options

* `USERMOD_DALLASTEMPERATURE_MEASUREMENT_INTERVAL` - number of milliseconds between measurements, defaults to 60000 ms (60s)

All parameters can be configured at runtime via the Usermods settings page, including pin, temperature in degrees Celsius or Fahrenheit and measurement interval.

## Project link

* [QuinLED-Dig-Uno](https://quinled.info/2018/09/15/quinled-dig-uno/) - Project link
* [Srg74-WLED-Wemos-shield](https://github.com/srg74/WLED-wemos-shield) - another great DIY WLED board

## Change Log

2020-09-12

* Changed to use async non-blocking implementation
* Do not report erroneous low temperatures to MQTT
* Disable plugin if temperature sensor not detected
* Report the number of seconds until the first read in the info screen instead of sensor error

2021-04

* Adaptation for runtime configuration.

2023-05

* Rewrite to conform to newer recommendations.
* Recommended @blazoncek fork of OneWire for ESP32 to avoid Sensor error

2024-09

* Update OneWire to version 2.3.8, which includes stickbreaker's and garyd9's ESP32 fixes:
  blazoncek's fork is no longer needed
