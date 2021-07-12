# Sketch build profiles and reproducible builds

## Overview

The Arduino build system and Sketch specifications lack a way to share and archive sketches and guarantee reproducibility of the built firmware at any time in the future as it was last successfully built.

These incompatibilities might arise due to newer versions of a Platform, Tool, or Library.

## Problem

An Arduino project is known as Sketch. It comprises its main `.ino` file plus optional local source code (in the same folder) and an optional extra `src` folder containing additional sources if required.

The sketch compiles against a set of globally installed libraries and platforms. When said library and/or part of the hardware platform is updated, sometimes introducing some breaking changes, that sketch might not compile anymore or behave differently.

Currently, the only way to guarantee portability and reproducibility of a sketch build at a later time is for the developer to provide instructions to the final user for the installation of the sketch dependencies, including the exact versions of the boards platform and libraries.

## Goals

This RFC aims to propose new augmentations to the current library management and a build system that can be entirely transparent to the user and retains backward compatibility yet allowing more flexibility for the future.

The following should be true:

- Implement ways for any Arduino Sketch to behave as a reproducible build with an option for versioning

- Allow sharing of such sketch and “environment settings” from

  - IDE 1.8 and 2.0 using Archive Sketch
  - Arduino Cloud using Share/Download Sketch

## Constraints

- Must be compatible across build systems

  - Arduino CLI
  - Arduino IDE 2.0
  - Arduino IDE 1.8
  - Arduino Cloud

Must not break pre-existing setups or be completely transparent to / ignored by previous versions of the tools

## Proposed Solution

The proposal is to add a project file called `sketch.yaml`. This file will be in YAML format and it will contain a description of all the resources required to compile a sketch, in particular:

- the board FQBN
- the target core platform version (and the 3rd party platform index URL if needed)
- a possible core platform that is a dependency of the target core platform (again with version and index URL)
- the libraries used (including their version)

We will call the set of the information above a **profile**. Each profile will be saved in the file `sketch.yaml` and this file may have one or more profiles.

The format of the file is the following:

```
profiles:
  <PROFILE_NAME>:
    notes: <USER_NOTES>
    fqbn: <FQBN>
    platforms:
      - platform: <PLATFORM> (<PLATFORM_VERSION>)
        platform_index_url: <3RD_PARTY_PLATFORM_URL>
      - platform: <PLATFORM_DEPENDENCY> (<PLATFORM_DEPENDENCY_VERSION>)
        platform_index_url: <3RD_PARTY_PLATFORM_DEPENDENCY_URL>
    libraries:
      - <LIB_NAME> (<LIB_VERSION>)
      - <LIB_NAME> (<LIB_VERSION>)
      - <LIB_NAME> (<LIB_VERSION>)

...more profiles here...

default_profile: <DEFAULT_PROFILE_NAME>
```

We have a `profiles:` section containing all the profiles. Each field is self-explanatory, in particular:

- `<PROFILE_NAME>` is the profile identifier, it’s a user-defined field, and the allowed characters are alphanumerics, underscore `_`, dot `.`, and dash `-`
- `<PLATFORM>` is the target core platform identifier, for example, `arduino:avr` or `adafruit:samd`
- `<PLATFORM_VERSION>` is the target core platform version required
- `<3RD_PARTY_PLATFORM_URL>` is the index URL to download the target core platform (also known as “Additional Boards Manager URLs” in the Arduino IDE). This field can be omitted for the official `arduino:*` platforms.
- `<PLATFORM_DEPENDENCY>`, `<PLATFORM_DEPENDENCY_VERSION>`, and `<3RD_PARTY_PLATFORM_DEPENDENCY_URL>` contains the same information as `<PLATFORM>`, `<PLATFORM_VERSION>`, and `<3RD_PARTY_PLATFORM_URL>` but for the core platform dependency of the main core platform. These fields are optional.
- `<LIB_VERSION>` is the version required for the library, for example, `1.0.0`
- `<USER_NOTES>` is a free text string available to the developer to add comments
- `<DEFAULT_PROFILE_NAME>` is the profile used by default (more on that later)

A complete example of a `sketch.yaml` may be the following:

```
profiles:
  nanorp:
    fqbn: arduino:mbed_nano:nanorp2040connect
    platforms:
      - platform: arduino:mbed_nano (2.1.0)
    libraries:
      - ArduinoIoTCloud (1.0.2)
      - Arduino_ConnectionHandler (0.6.4)
      - TinyDHT sensor library (1.1.0)

  another_profile_name:
    notes: testing the limit of the AVR platform, may be unstable
    fqbn: arduino:avr:uno
    platforms:
      - platform: arduino:avr (1.8.4)
    libraries:
      - VitconMQTT (1.0.1)
      - Arduino_ConnectionHandler (0.6.4)
      - TinyDHT sensor library (1.1.0)

  tiny:
    notes: testing the very limit of the AVR platform, it will be very unstable
    fqbn: attiny:avr:ATtinyX5:cpu=attiny85,clock=internal16
    platforms:
      - platform: attiny:avr@1.0.2
        platform_index_url: https://raw.githubusercontent.com/damellis/attiny/ide-1.6.x-boards-manager/package_damellis_attiny_index.json
      - platform: arduino:avr@1.8.3
    libraries:
      - ArduinoIoTCloud (1.0.2)
      - Arduino_ConnectionHandler (0.6.4)
      - TinyDHT sensor library (1.1.0)

  feather:
    fqbn: adafruit:samd:adafruit_feather_m0
    platforms:
      - platform: adafruit:samd (1.6.0)
        platform_index_url: https://adafruit.github.io/arduino-board-index/package_adafruit_index.json
    libraries:
      - ArduinoIoTCloud (1.0.2)
      - Arduino_ConnectionHandler (0.6.4)
      - TinyDHT sensor library (1.1.0)

default_profile: nanorp
```

## Building a sketch

When a `sketch.yaml` file exists in the sketch, it can be leveraged to compile the sketch with the `--profile` flag in the compile command:

```
arduino-cli compile [sketch] --profile nanorp
```

In this case, the sketch will be compiled using the core platform and libraries specified in the `nanorp` profile. If a core platform or a library is missing it will be automatically downloaded and installed on the fly in a dedicated directory inside the data folder. The dedicated storage is not accessible to the user and is meant as a “cache” of the resources used to build the sketch.

When using the profile-based build, the globally installed platforms and libraries are excluded from the compile and **can not be used in any way**, in other words, the build is isolated from the system and will rely only on the resources specified in the profile: this will ensure that the build is portable and reproducible independently from the platforms and libraries installed in the system.

## Using a default profile

If a `default_profile` is specified in the `sketch.yaml` then the “classic” compile command:

```
arduino-cli compile [sketch]
```

will, instead, trigger a profile-based build using the default profile indicated in the `sketch.yaml`.

## Vendoring libraries with custom modifications

We will add the possibility to add custom libraries directly in the sketch using the `libraries` subdirectory in the sketch root folder.

A typical usage scenario is when the sketch needs a library that is not offered for installation from Library Manager, or if the sketch needs a library with some customizations that are not available in the upstream release.

To accomplish this the idea is to put the custom libraries' source code directly in the `libraries` subdirectory of the sketch (exactly as we do for the globally installed libraries in the `libraries` subdirectory of the sketchbook root folder). For example:

```
~/Arduino/HumiditySensor$ tree
HumiditySensor
├── HumiditySensor.ino
├── sketch.yaml
└── libraries
    ├── DHTSensorCustom
    │   ├── library.properties
    │   ├── README
    │   └── src
    │       ├── DHT.cpp
    │       ├── DHT.h
    │       └── DHTSensorCustom.h
    └── NTPClientEth
        ├── NTPClient.h
        └── NTPClient.cpp
```

If we run a “classic” build, without profiles, then the libraries `DHTSensorCustom` and `NTPClientEth` will be included at compile time as they were installed globally.

If we want to run a reproducible build with profiles, we need to explicitly specify the bundled libraries we intend to use in the `sketch.yaml` in this way:

```
profiles:
  nanorp:
    fqbn: arduino:mbed_nano:nanorp2040connect
    platforms:
      - platform: arduino:mbed_nano (2.1.0)
    libraries:
      - ArduinoIoTCloud (1.0.2)
      - Arduino_ConnectionHandler (0.6.4)
      - TinyDHT sensor library (1.1.0)
    vendored_libraries:
      - NTPClientEth

  uno:
    notes: testing the limit of the AVR platform, may be unstable
    fqbn: arduino:avr:uno
    platforms:
      - platform: arduino:avr (1.8.4)
    libraries:
      - VitconMQTT (1.0.1)
      - Arduino_ConnectionHandler (0.6.4)
    vendored_libraries:
      - DHTSensorCustom
      - NTPClientEth
```

In this case:

- running a build using the profile `nanorp` will use the bundled `NTPClientEth` library, but not `DHTSensorCustom`
- running a build with the profile `uno` will use both `NTPClientEth` and `DHTSensorCustom`

This behavior is required to allow different profiles to use a different set of vendored libraries.
