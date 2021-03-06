# Pluggable Discovery protocol (Version 1)

## Overview

This document describes how the Pluggable Discovery works and how it should integrate with monitors and uploaders.

## Problem

When a sketch is uploaded to an Arduino board the only way for transferring the binary executable to the microcontroller is through the serial port. Currently:

- the subroutines to enumerate the serial ports and to possibly identify a board are “hardcoded” in the IDE/CLI
- the only way to identify a board is via USB VID/PID identifiers
- the "serial monitor", to communicate with the board after the upload, is “hardcoded” in the IDE

The current structure does not allow to use different kind of “ports” to communicate with the microcontroller. For example it would be interesting to have the possibility to:

- discover a board (MDNS) and upload via network (OTA)

- discover a board and upload via a communication port that is not the serial port (for example via USB-HID as implemented by Teensy or via any other kind of port)

- allow a third party to implement his own upload protocol / port

The pluggable discovery aims to provide a solution to these problems.

## Constraints

- A communication port may use any kind of **protocol**

- Software support for a new **protocol** must be added to the system **using an external command line tool**

- Each port must be **enumerated/discovered**

- Each port may provide **properties relative to the port** (USB serial number / MAC address)

- Each port must have an **unique address and protocol pair**

- A single device may expose multiple ports

## Proposed Solution

The proposed solution is to provide a tool that can enumerate and discover the ports for a **single specific protocol**. There will be a tool for each supported protocol so, in the end, we will have a “Serial ports discovery”, a “Network port discovery” and so on.

These tools must be in the form of executables that can be launched as a subprocess using a `platform.txt` command line recipe. They will communicate to the parent process via stdin/stdout, in particular a discovery will accept commands as plain text strings from stdin and will send answers back in JSON format on stdout. Each tool will implement the commands to list and enumerate ports for a specific protocol as specified in this document.

### Pluggable discovery API via stdin/stdout

All the commands listed in this specification must be implemented in the discovery.

After startup, the tool will just stay idle waiting for commands. The available commands are: `HELLO`, `START`, `STOP`, `QUIT`, `LIST` and `START_SYNC`.

After each command the client always expect a response from the discovery. The discovery must not introduce any delay and must respond to all commands as fast as possible.

#### HELLO command

`HELLO` **must be the first command sent** to the discovery to tell the name of the client/IDE and the version of the pluggable discovery protocol that the client/IDE supports.
The syntax of the command is:

`HELLO <PROTOCOL_VERSION> "<USER_AGENT>"`

- `<PROTOCOL_VERSION>` is the maximum protocol version supported by the client/IDE (currently `1`)

- `<USER_AGENT>` is the name and version of the client (double-quotes `"` are not allowed)

some examples:

- `HELLO 1 "Arduino IDE 1.8.13"`

- `HELLO 1 "arduino-cli 1.2.3"`

the response to the command is:

```JSON
{
  "eventType": "hello",
  "protocolVersion": 1,
  "message": "OK"
}
```

The `protocolVersion` field represents the protocol version that will be used in the rest of the communication. There are three possible cases:

- if the client/IDE supports the same or a more recent version of the protocol than the discovery, then the IDE should go into a compatibility mode and use the protocol level supported by the discovery.
- if the discovery supports a more recent version of the protocol than the client/IDE: the discovery should downgrade itself into compatibility mode and report a `protocolVersion` that is less than or equal to the one supported by the client/IDE.
- if the discovery cannot go into compatibility mode, it will report the protocol version supported (even if greater than the version supported by the client/IDE) and the client/IDE may decide to terminate the discovery or produce an error/warning.

#### START command

The `START` command initializes and start the discovery internal subroutines. This command must be called before `LIST` or `START_SYNC`. The response to the start command is:

```JSON
{
  "eventType": "start",
  "message": "OK"
}
```

If the discovery could not start, for any reason, it must report the error with:

```JSON
{
  "eventType": "start",
  "error": true,
  "message": "Permission error"
}
```

The `error` field must be set to `true` and the `message` field should contain a description of the error.

#### STOP command

The `STOP` command stops the discovery internal subroutines and possibly free the internally used resources. This command should be called if the client wants to pause the discovery for a while. The response to the command is:

```JSON
{
  "eventType": "stop",
  "message": "OK"
}
```

If an error occurs:

```JSON
{
  "eventType": "stop",
  "error": true,
  "message": "Resource busy"
}
```

The `error` field must be set to `true` and the `message` field should contain a description of the error.

#### QUIT command

The `QUIT` command terminates the discovery. The response to `QUIT` is:

```JSON
{
  "eventType": "quit",
  "message": "OK"
}
```

after this output the discovery exits. This command is supposed to always succeed.

#### LIST command

The `LIST` command executes an enumeration of the ports and returns a list of the available ports at the moment of the call. The format of the response is the following:

```
{
  "eventType": "list",
  "ports": [
    {
      "address":       <-- THE ADDRESS OF THE PORT
      "label":         <-- HOW THE PORT IS DISPLAYED ON THE GUI
      "protocol":      <-- THE PROTOCOL USED BY THE BOARD
      "protocolLabel": <-- HOW THE PROTOCOL IS DISPLAYED ON THE GUI
      "properties": {
                       <-- A LIST OF PROPERTIES OF THE PORT
      }
    }, {
      ...              <-- OTHER PORTS...
    }
  ]
}
```

The `ports` field contains a list of the available ports.

Each port has:

- an `address` (for example `/dev/ttyACM0` for serial ports or `192.168.10.100` for network ports)

- a `label` that is the human readable form of the `address` (it may be for example `ttyACM0` or `SSH on 192.168.10.100`)

- `protocol` is the protocol identifier (such as `serial` or `dfu` or `ssh`)

- `protocolLabel` is the `protocol` in human readable form (for example `Serial port` or `DFU USB` or `Network (ssh)`)

- `properties` is a list of key/value pairs that represent information relative to the specific port

To make the above more clear let’s show an example with the `serial_discovery`:

```JSON
{
  "eventType": "list",
  "ports": [
    {
      "address": "/dev/ttyACM0",
      "label": "ttyACM0",
      "protocol": "serial",
      "protocolLabel": "Serial Port (USB)",
      "properties": {
        "pid": "0x804e",
        "vid": "0x2341",
        "serialNumber": "EBEABFD6514D32364E202020FF10181E",
        "name": "ttyACM0"
      }
    }
  ]
}
```

In this case the serial port metadata comes from an USB serial converter. Inside the `properties` we have all the properties of the port, and some of them may be useful for product identification (in this case only USB VID/PID is useful to identify the board model).

The `LIST` command performs a one-shot polling of the ports. The discovery should answer as soon as reasonably possible, without any additional delay.

Some discoveries may require some time to discover a new port (for example network protocols like MDNS, Bluetooth, etc. requires some seconds to receive the broadcasts from all available clients) in that case is fine to answer with an empty or incomplete list.

If an error occurs and the discovery can't complete the enumeration is must report the error with:

```JSON
{
  "eventType": "list",
  "error": true,
  "message": "Resource busy"
}
```

The `error` field must be set to `true` and the `message` field should contain a description of the error.

#### START_SYNC command

The `START_SYNC` command puts the tool in "events" mode: the discovery will send `add` and `remove` events each time a new port is detected or removed respectively. If the discovery goes into "events" mode successfully the response to this command is:

```JSON
{
  "eventType": "start_sync",
  "message": "OK"
}
```

After this message the discoery will send `add` and `remove` event asyncronoushly (more on that later). If an error occurs and the discovery can't go in "events" mode the error must be reported as:

```JSON
{
  "eventType": "start_sync",
  "error": true,
  "message": "Resource busy"
}
```

The `error` field must be set to `true` and the `message` field should contain a description of the error.

Once in "event" mode, the discovery is allowed to send `add` and `remove` messages asynchronously in realtime, this means that the client must be able to handle these incoming messages at any moment.

The `add` event looks like the following:

```JSON
{
  "eventType": "add",
  "port": {
    "address": "/dev/ttyACM0",
    "label": "ttyACM0",
    "properties": {
      "pid": "0x804e",
      "vid": "0x2341",
      "serialNumber": "EBEABFD6514D32364E202020FF10181E",
      "name": "ttyACM0"
    },
    "protocol": "serial",
    "protocolLabel": "Serial Port (USB)"
  }
}
```

It basically provides the same information as the `list` event but for a single port. After calling `START_SYNC` an initial burst of add events must be generated in sequence to report all the ports available at the moment of the start.

The `remove` event looks like the following:

```JSON
{
  "eventType": "remove",
  "port": {
    "address": "/dev/ttyACM0",
    "protocol": "serial"
  }
}
```

The content is straightforward, in this case only the `address` and `protocol` fields are reported.

If the information about a port needs to be updated the discovery may send a new `add` message for the same port address and protocol without sending a `remove` first: this means that all the previous information about the port must be discarded and replaced with the new one.

#### Invalid commands

If the client sends an invalid or malformed command, the discovery should answer with:

```JSON
{
  "eventType": "command_error",
  "error": true,
  "message": "Unknown command XXXX"
}
```

#### arduino-cli usage scenario

If `arduino-cli` is used in command-line mode (for example the `arduino-cli board list` command) then we need a one-shot output and we can run the discoveries with the `LIST` command instead of using `START_SYNC`.

Note that some discoveries may not be able to `LIST` ports immediately after the launch (in particular network protocols like MDNS, Bluetooth, etc. requires some seconds, even minutes, to receive the broadcasts from all available clients). This is intrinsic in how the `LIST` command is defined and it's a duty of the client (CLI/IDE) to take it into consideration.

If `arduino-cli` is running in daemon mode, ideally, it should start all the installed discoveries simultaneously in “event mode” (with `START_SYNC`) and the list of available ports should be cached and updated in realtime inside the `arduino-cli` daemon.

#### Reference implementations

A demo tool is available here:
https://github.com/arduino/serial-discovery

A typical usage scenario is here:
https://github.com/arduino/serial-discovery#example-of-usage

### Integration with `arduino-cli` and core platforms

In this section we will see how discvoeries are distributed and integrated with Arduino platforms.

#### Discovery tool distribution

The discovery tools must be built natively for each OS and the CLI should run the correct tool for the running OS.

The distribution infrastracture is already available for platform tools, like compilers and uploaders, through `package_index.json` so, the most natural way forward is to distribute also the discoveries in the same way.
3rd party developers should provide their discovery tools by adding them as resources in the `tools` section of `package_index.json` (at the `packages` level).

Let's see how this looks into a `package_index.json` example:

```diff
{
  "packages": [
    {
      "name": "arduino",
      "maintainer": "Arduino",
      "websiteURL": "http://www.arduino.cc/",

      "platforms": [
        ...
      ],
      "tools": [
        {
          "name": "arm-none-eabi-gcc",
          "version": "4.8.3-2014q1",
          "systems": [ ... ]
        },
+       {
+         "name": "ble-discovery", <--- Discovery is distributed as a TOOL
+         "version": "1.0.0",
+         "systems": [
+           {
+             "host": "x86_64-pc-linux-gnu",
+             "url": "http://example.com/ble-disc-1.0.0-linux64.tar.gz",
+             "archiveFileName": "ble-disc-1.0.0-linux64.tar.gz",
+             "checksum": +SHA-256:0123456789abcdef0123456789abcdef0123456789abcdef",
+             "size": "12345678"
+           },
+           ...
+         ]
+       }
      ],
    }
  }
}
```

In this case we are adding an hypotetical `ble-discovery` version `1.0.0` to the toolset of the vendor `arduino`. From now on, we can uniquely refer to this discovery with the pair `PACKAGER` and `DISCOVERY_NAME`, in this case `arduino` and `ble-discovery` respectively.

The compressed archive of the discovery must contain only a single executable file (the discovery itself) inside a single root folder. This is mandatory since the CLI will run this file automatically when the discovery is started.

#### Discovery tools integration

Each core platform must refer to the specific discovery tools they need by adding them a new `discoveryDependencies` field of the `package_index.json`. Let's look again at the previous example:

```diff
{
  "packages": [
    {
      "name": "arduino",
      "maintainer": "Arduino",
      "websiteURL": "http://www.arduino.cc/",

      "platforms": [
        {
          "name": "Arduino AVR Boards",
          "architecture": "avr",
          "version": "1.6.2",
          ...
          "toolsDependencies": [
            {
              "packager": "arduino",
              "name": "arm-none-eabi-gcc",
              "version": "4.8.3-2014q1"
            },
            {
              "packager": "arduino",
              "name": "CMSIS",
              "version": "4.5.0"
            },
            ...
          ],
+         "discoveryDependencies": [  <--- Discoveries used in the platform
+           {
+             "packager": "arduino",
+             "name": "ble-discovery"
+                                     <--- Version is not required!
+           }
+         ]
        },

        {
          "name": "Arduino SAMD Boards",
          "architecture": "samd",
          "version": "1.6.18",
          ...
          "toolsDependencies": [ ... ],
+         "discoveryDependencies": [ ... ]
        }

      ],
      "tools": [
        {
          "name": "arm-none-eabi-gcc",
          "version": "4.8.3-2014q1",
          "systems": [ ... ]
        },
        {
          "name": "ble-discovery",
          "version": "1.0.0",
          "systems": [ ... ]
        }
      ],
    }
  }
}
```

Adding the needed discoveries in the `discoveryDependencies` allows the CLI to install them together with the platform. Also, differently from the other `toolsDependencies`, the version is not required since it will always be used the latest version available.

Finally, to actually bind a discovery to a platform, we must also declare in the `platform.txt` that we want to use that specific discovery with the direcive:

```
discovery.required=PLATFORM:DISCOVERY_NAME
```

or if the platform needs more discoveries we can use the indexed version:

```
discovery.required.0=PLATFORM:DISCOVERY_ID_1
discovery.required.1=PLATFORM:DISCOVERY_ID_2
...
```

in our specific example the directive should be:

```
discovery.required=arduino:ble-discovery
```

#### Using a discovery made by a 3rd party

A platform developer may opt to depend on a discovery developed by a 3rd party instead of writing and maintaining his own.

Since writing a good-quality cross-platform discovery is very hard and time consuming, we expect this option to be the one used by the majority of the developers.

#### Direct discovery integration (not recommended)

A discovery may be directly added to a platform, without passing through the `discoveryDependencies` in `package_index.json`, using the following directive in the `platform.txt`:

```
discovery.DISCOVERY_ID.pattern=DISCOVERY_RECIPE
```

`DISCOVERY_ID` must be replaced by a unique identifier for the particular discovery and `DISCOVERY_RECIPE` must be replaced by the command line to launch the discovery. An example could be:

```
## Teensy Ports Discovery
discovery.teensy.pattern="{runtime.tools.teensy_ports.path}/hardware/tools/teensy_ports" -J2
```

in this case the platform provides a new `teensy` discovery and the command line tool named `teensy_ports` is launched with the `-J2` parameter to start the discovery tool. In this case the command line pattern may contain any extra parameter in the formula: this is different from the discoveries installed through the `discoveryDependencies` field that are run automatically without any command line parameter.

This kind of integration may turn out useful:

- during the development of a platform (because providing a full `package_index.json` may be cumbersome)
- if the discovery is specific for a platform and can not be used by 3rd party

Anyway, since this kind of integration does not allow reusing a discovery between different platforms, we do not recommend its use.

#### built-in discoveries and backward compatibliity considerations

Some discoveries like the Arduino `serial-discovery` or the Arduino `network-discovery` must be always available, so they will be part of the `builtin` package and installed without the need to be part of a real package (`builtin` is a dummy package that we use to install tools that are not part of any platforms like `ctags` for example).

If a platform requires the builtin discoveries it must declare it with:

```
discovery.required.0=builtin:serial-discovery
discovery.required.1=builtin:network-discovery
```

For backward compatibility, if a platform does not declare any discovery (using the `discovery.*` properties in `platform.txt`) it will automatically inherits `builtin:serial-discovery` and `builtin:network-discovery` (but not other `builtin` discoveries that may be possibly added in the future). This will allow all legacy non-pluggable platforms to migrate to pluggable discovery without disruption.

#### Conflicting discoveries

In case different discoveries provide conflicting information (for example if two discoveries provide different information for the same port) we could partially mitigate the issue by giving priority to the discovery that is used by the package of the selected board.

### Board identification

The `properties` associated to a port can be used to identify the board attached to that port. The algorithm is simple:

- each board listed in the platform file `boards.txt` may declare a set of `upload_port.*` properties
- if each `upload_port.*` property has a match in the `properties` set coming from the discovery then the board is a “candidate” board attached to that port.

Some port `properties` may not be precise enough to uniquely identify a board, in that case more boards may match the same set of `properties`, that’s why we called it “candidate”.

Let’s see an example to clarify things a bit, let's suppose that we have the following `properties` coming from the serial discovery:

```JSON
  "port": {
    "address": "/dev/ttyACM0",
    "properties": {
      "pid": "0x804e",
      "vid": "0x2341",
      "serialNumber": "EBEABFD6514D32364E202020FF10181E",
      "name": "ttyACM0"
    }
```

in this case we can use `vid` and `pid` to identify the board. The `serialNumber`, instead, is unique for that specific instance of the board so it can't be used to identify the board model. Let’s suppose we have the following `boards.txt`:

```
# Arduino Zero (Prorgamming Port)
# ---------------------------------------
arduino_zero_edbg.name=Arduino Zero (Programming Port)
arduino_zero_edbg.upload_port.vid=0x03eb
arduino_zero_edbg.upload_port.pid=0x2157
[...CUT...]
# Arduino Zero (Native USB Port)
# --------------------------------------
arduino_zero_native.name=Arduino Zero (Native USB Port)
arduino_zero_native.upload_port.0.vid=0x2341
arduino_zero_native.upload_port.0.pid=0x804d
arduino_zero_native.upload_port.1.vid=0x2341
arduino_zero_native.upload_port.1.pid=0x004d
arduino_zero_native.upload_port.2.vid=0x2341
arduino_zero_native.upload_port.2.pid=0x824d
arduino_zero_native.upload_port.3.vid=0x2341
arduino_zero_native.upload_port.3.pid=0x024d
[...CUT...]
# Arduino MKR1000
# -----------------------
mkr1000.name=Arduino MKR1000
mkr1000.upload_port.0.vid=0x2341       <------- MATCHING IDs
mkr1000.upload_port.0.pid=0x804e       <------- MATCHING IDs
mkr1000.upload_port.1.vid=0x2341
mkr1000.upload_port.1.pid=0x004e
mkr1000.upload_port.2.vid=0x2341
mkr1000.upload_port.2.pid=0x824e
mkr1000.upload_port.3.vid=0x2341
mkr1000.upload_port.3.pid=0x024e
[...CUT...]
```

As we can see the only board that has the two properties matching is the `mkr1000`, in this case the CLI knows that the board is surely an MKR1000.

Note that `vid` and `pid` properties are just free text key/value pairs: the discovery may return basically anything, the board just needs to have the same properties defined in `boards.txt` as `upload_port.*` to be identified.

We can also specify multiple identification properties for the same board using the `.N` suffix, for example:

```
myboard.name=My Wonderful Arduino Compatible Board
myboard.upload_port.pears=20
myboard.upload_port.apples=30
```

will match on `pears=20, apples=30` but:

```
myboard.name=My Wonderful Arduino Compatible Board
myboard.upload_port.0.pears=20
myboard.upload_port.0.apples=30
myboard.upload_port.1.pears=30
myboard.upload_port.1.apples=40
```

will match on both `pears=20, apples=30` and `pears=30, apples=40` but not `pears=20, apples=40`, in that sense each "set" of identification properties is indepentent from each other and cannot be mixed for port matching.

#### Backward compatibility considerations

Many platforms predating the pluggable discovery have the following definitions for board identification using USB Serial VID/PID:

```
myboard.vid=0x1234
myboard.pid=0x4567
```

or:

```
myboard.vid.0=0x1234
myboard.pid.0=0x4567
```

to ensure backward compatibility we will transparently and automatically convert these definitions into the new format

```
myboard.upload_port.0.vid=0x1234
myboard.upload_port.0.pid=0x4567
```

### Upload (state of the art)

In this section we will discuss the current status of the upload business logic in the IDE/CLI.

#### Serial upload (CLI and Java IDE)

Currently, to upload we use the `tools.UPLOAD_RECIPE_ID.upload.pattern` recipe in `platform.txt`, where `UPLOAD_RECIPE_ID` is defined as the value of `upload.tool` property in `boards.txt` for the currently user-selected board.

The recipe contains the variables `{serial.port}` and`{serial.port.file}` that are replaced with the user selected serial port (in particular on unix systems `{serial.port.file}` contains only the file name `ttyACM0` while `{serial.port}` contains the full path to the serial port `/dev/ttyACM0`). A typical serial upload recipe is something like:

`tools.bossac.upload.pattern="{runtime.tools.bossac-1.7.0-arduino3.path}/bossac" --port={serial.port.file} -U true -i -e -w -v "{build.path}/{build.project_name}.bin" -R`

#### Network upload (Java IDE only)

The Java IDE has an integrated MDNS client that listen for network boards. If a network board is selected the`tools.UPLOAD_RECIPE_ID.upload.network_pattern` is used for upload. `UPLOAD_RECIPE_ID` is obtained in the same way as for the serial upload. A typical recipe is:

`tools.bossac.upload.network_pattern="{runtime.tools.arduinoOTA.path}/bin/arduinoOTA" -address {serial.port} -port 65280 -username arduino -password "{network.password}" -sketch "{build.path}/{build.project_name}.bin" -upload /sketch -b`

please note that:

- the property is called `network_pattern` instead of `pattern`, this is hardcoded

- the port address is still called `{serial.port}` even if it is a network address

- there is a `{network.password}` that is a user entered field, this is hardcoded

### Upload with Pluggable discovery

Given the above, it is clear that the pluggable discovery should provide features to allow:

1. selection of different upload recipes based on the port `protocol`
1. provide extra port metadata to use on the command line recipe
1. provide eventual extra info that should be user-supplied (by entering them in a dialog or provided in other ways)
1. should be backward compatible with the current legacy

#### Recipe selection based on protocol

To allow the above the `UPLOAD_RECIPE_ID` selection is now **dependent on protocol**. The `boards.txt` now must specify a recipe id for each type of upload protocol supported by the board. Previously we used the property `upload.tool=UPLOAD_RECIPE_ID` now this property should be changed to `upload.tool.UPLOAD_PROTOCOL=UPLOAD_RECIPE_ID` so we can choose the correct tool for each protocol. We should also keep the old property for backward compatibility

**Before:**

```
myboard.upload.tool=bossac
```

**After:**

```
# keep for backward compatibility
myboard.upload.tool=bossac
# Upload recipes
myboard.upload.tool.serial=bossac
myboard.upload.tool.network=arduino_ota
```

The selected port address will be provided in the variable `{upload.port.address}`. In general, all the metadata provided by the discovery in the `port` section will be provided under the `{upload.port.*}` variables.

For backward compatibility we will keep a copy of the address also in `{serial.port}` and in the specific case of a `protocol=serial` we will populate also `{serial.port.file}`.

For example, the following port metadata coming from a pluggable discovery:

```JSON
{
  "eventType": "add",
  "port": {
    "address": "/dev/ttyACM0",
    "label": "ttyACM0",
    "protocol": "serial",
    "protocolLabel": "Serial Port (USB)",
    "properties": {
      "pid": "0x804e",
      "vid": "0x2341",
      "serialNumber": "EBEABFD6514D32364E202020FF10181E",
      "name": "ttyACM0"
    }
  }
}
```

will be available on the recipe as the variables:

```
{upload.port.address} = /dev/ttyACM0
{upload.port.label} = ttyACM0
{upload.port.protocol} = serial
{upload.port.protocolLabel} = Serial Port (USB)
{upload.port.properties.pid} = 0x8043
{upload.port.properties.vid} = 0x2341
{upload.port.properties.serialNumber} = EBEABFD6514D32364E202020FF10181E
{upload.port.properties.name} = ttyACM0
{serial.port} = /dev/ttyACM0                # for backward compatibility
{serial.port.file} = ttyACM0                # only because protocol=serial
```

Here another example:

```JSON
{
  "eventType": "add",
  "port": {
    "address": "192.168.1.232",
    "label": "SSH on my-board (192.168.1.232)",
    "protocol": "ssh",
    "protocolLabel": "SSH Network port",
    "properties": {
      "macprefix": "AA:BB:CC",
      "macaddress": "AA:BB:CC:DD:EE:FF"
    }
  }
}
```

that is translated to:

```
{upload.port.address} = 192.168.1.232
{upload.port.label} = SSH on my-board (192.168.1.232)
{upload.port.protocol} = ssh
{upload.port.protocolLabel} = SSH Network port
{upload.port.properties.macprefix} = AA:BB:CC
{upload.port.properties.macaddress} = AA:BB:CC:DD:EE:FF
{serial.port} = 192.168.1.232                  # for backward compatibility
```

This configuration, together with protocol selection, allows to remove the hardcoded `network_pattern`, now we can replace the legacy recipe (split into multiple lines for clarity):

```
tools.bossac.upload.network_pattern="{runtime.tools.arduinoOTA.path}/bin/arduinoOTA"
                                    -address {serial.port} -port 65280
                                    -sketch "{build.path}/{build.project_name}.bin"
```

with:

```
tools.arduino_ota.upload.pattern="{runtime.tools.arduinoOTA.path}/bin/arduinoOTA"
                                 -address {upload.port.address} -port 65280
                                 -sketch "{build.path}/{build.project_name}.bin"
```

#### User provided fields

The recipe may require some user-entered fields (like username or password). In this case the recipe must use the special placeholder `{upload.field.FIELD_NAME}` where `FIELD_NAME` must be declared separately in the recipe using the following format:

```
tools.UPLOAD_RECIPE_ID.upload.field.FIELD_NAME=FIELD_LABEL
tools.UPLOAD_RECIPE_ID.upload.field.FIELD_NAME.secret=true
```

`FIELD_LABEL` is the label shown in the dialog to ask the user to enter the value of the field. The `secret` property is optional and it should be set to `true` if the field is a secret (like passwords or token).

Let’s see how a complete example will look like:

```
tools.arduino_ota.upload.field.username=Username
tools.arduino_ota.upload.field.password=Password
tools.arduino_ota.upload.field.password.secret=true
tools.arduino_ota.upload.pattern="{runtime.tools.arduinoOTA.path}/bin/arduinoOTA"
                                 -address {upload.port.address} -port 65280
                                 -username "{upload.field.username}
                                 -password "{upload.field.password}"
                                 -sketch "{build.path}/{build.project_name}.bin"
```

#### Support for `default` fallback (allows upload "without a port")

Some upload tools already have the port detection builtin so there is no need to specify an upload port (for example the `openocd` tool is often able to autodetect the devices to upload by itself).

To support this particular use case a dummy `default` protocol has been reserved:

```
myboard.upload.tool.default=openocd_without_port
```

The `default` upload protocol is a kind of wildcard-protocol, it will be selected when:

- The upload port is not specified

or

- The upload port is specified but the protocol of the selected port doesn't match any of the available upload protocols for a board

Let's see some examples to clarify:

```
board1.upload.tool.default=openocd_without_port

board2.upload.tool.serial=bossac
board2.upload.tool.default=openocd_without_port

board3.upload.tool.serial=bossac
```

In the `board1` case: the `openocd_without_port` recipe will be always used, whatever the user's port selection.

In the `board2` case: the `bossac` recipe will be used if the port selected is a `serial` port, otherwise the `openocd_without_port` will be used in all other cases (whatever the user's port selection).

In the `board3` case: the `bossac` recipe will be used if the port selected is a `serial` port, otherwise the upload will fail.

A lot of legacy platforms already have recipes without an explicit port address, for example let's consider the Arduino Zero board:

```
# Arduino Zero (Prorgamming Port)
# ---------------------------------------
arduino_zero_edbg.name=Arduino Zero (Programming Port)
arduino_zero_edbg.vid.0=0x03eb
arduino_zero_edbg.pid.0=0x2157

arduino_zero_edbg.upload.tool=openocd             <---
arduino_zero_edbg.upload.protocol=sam-ba
arduino_zero_edbg.upload.maximum_size=262144
arduino_zero_edbg.upload.maximum_data_size=32768
arduino_zero_edbg.upload.use_1200bps_touch=false
arduino_zero_edbg.upload.wait_for_upload_port=false
arduino_zero_edbg.upload.native_usb=false
```

in this case, to ensure backward compatibility, the upload tool specified in the old (non-pluggable) way will be considered as a `default` protocol upload, and it will be automatically converted into:

```
arduino_zero_edbg.upload.default.tool=openocd
```

Please note that the transformation above is intended only as a backward compatibility helper and it will be applied only on platforms that does not support Pluggable Discovery at all: if any other `*.upload.*.tool` or `discovery.*` definition is found in the platform, the transformation above will **not** be automatically applied.

## Open Questions

### CLI command line UX considerations

Currently only serial ports are allowed on the CLI, so we don’t need to specify a protocol when we select a port, if we write `arduino-cli upload --port /dev/ttyACM0` or `arduino-cli upload --port COM1` we are sure that this is a serial port.

Things get a bit more complex with Pluggable Discovery: `--port COM1` could be a serial port or the hostname of a network board that is called exactly `COM1`. To solve this amiguity we may add a flag to the `upload` command to specify the protocol so it becomes `arduino-cli upload --protocol serial --port COM1`.

Obviously using `--protocol` is ugly and makes the command line overcrowded. To avoid the `--protocol` flag we may adopt some strategies:

- we could check the current overall port list and see if the specified port address match **only one** of the available ports

- we may check if the address is a valid address for **only one** of the supported protocol. For example if we have support for `serial` and `network` only and if the entered address is `1.2.3.4` we know that this is a `network` address. This strategy requires that the pluggable discoveries have a command to check if the address is a valid address for their specific protocol.

- we may save the protocol as part of the `board attach` command (as we do already for the port address)

- we may use a URL format to specify protocol and address at the same time: `--port serial:///dev/ttyACM0` or `--port ssh://1.2.3.4` or `--port xyz://12-34/32:23/442/212`. BTW It may be problematic to express an address in form of an URL.

## Appendix

We have a POC implementation of a serial discovery:
https://github.com/arduino/serial-discovery

and a network discovery:
https://github.com/arduino/mdns-discovery
