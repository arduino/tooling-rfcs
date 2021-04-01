# Pluggable Discovery

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

- Software support for a new **protocol** must be added to the system as a **plugin**

- Each port must be **enumerated/discovered**

- Each port may provide metadata to identify the board model (**identification properites**)

- Each port may provide metadata to identify a specific instance of the board (**serial number / MAC address**)

- Each port must have an **unique address**

- A single device may expose multiple ports

## Recommended Solution

The proposed solution is to provide a tool that can enumerate and discover the ports for a **single specific protocol**. There will be a tool for each supported protocol so, in the end, we will have a “Serial ports discovery”, a “Network port discovery” and so on.

These tools must be in the form of executables that can be launched as a subprocess using a `platform.txt` command line recipe. They will communicate to the parent process via stdin/stdout, in particular a discovery will accept commands as plain text strings from stdin and will send answers back in JSON format on stdout. Each tool will implement the commands to list and enumerate ports for a specific protocol as specified in this document.

### Pluggable discovery API via stdin/stdout

All the commands listed in this specification must be implemented in the discovery.

After startup, the tool will just stay idle waiting for commands. The available commands are: `START`, `STOP`, `QUIT`, `LIST` and `START_SYNC`.

#### START command

The `START` command initializes and start the discovery internal subroutines. This command must be called before `LIST` or `START_SYNC`. The response to the start command is:

```JSON
{
  "eventType": "start",
  "message": "OK"
}
```

#### STOP command

The `STOP` command stops the discovery internal subroutines and possibly free the internally used resources. This command should be called if the client wants to pause the discovery for a while. The response to the command is:

```JSON
{
  "eventType": "stop",
  "message": "OK"
}
```

#### QUIT command

The `QUIT` command terminates the discovery. The response to `QUIT` is:

```JSON
{
  "eventType": "quit",
  "message": "OK"
}
```

after this output the discovery exits.

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
      "prefs": {
                       <-- A LIST OF PROPERTIES FOR THAT SPECIFIC BOARD
      },
      "identificationPrefs": {
                       <-- A LIST OF PROPERTIES TO IDENTIFY THE BOARD MODEL
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

- `prefs` is a list of key/value pairs that represent information relative to the specific instance of the board

- `identificationPrefs` is a list of key value pairs that represent information to identify the board **model**

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
      "prefs": {
        "pid": "0x804e",
        "vid": "0x2341",
        "serialNumber": "EBEABFD6514D32364E202020FF10181E"
      },
      "identificationPrefs": {
        "pid": "0x804e",
        "vid": "0x2341"
      }
    }
  ]
}
```

In this case the serial port metadata comes from an USB serial converter. The USB VID/PID and USB SERIAL NUMBER properties are also reported inside `prefs`. Inside the `identificationPrefs` instead we have only the properties useful for product identification (in this case only USB VID/PID is useful to identify the board model).

The `LIST` command performs a one-shot polling of the ports. If you need continuous monitoring of ports you should use the `START_SYNC` command.

#### START_SYNC command

The `START_SYNC` command puts the tool in "events" mode: the discovery will send `add` and `remove` events each time a new port is detected or removed respectively.

If the CLI is used in command-line mode (`arduino-cli board list` command) then we need a one-shot output and we can run the discoveries with the `LIST` command instead. Note that some discoveries may not be able to `LIST` ports immediately after the launch (in particular network protocols like MDNS, Bluetooth, etc. requires some seconds to receive the broadcasts from all available clients).

If the CLI is running in daemon mode, ideally, all the installed discoveries should be started simultaneously in “event mode” (with `START_SYNC`) and the list of available ports should be cached inside the CLI daemon.

The `add` event looks like the following:

```JSON
{
  "eventType": "add",
  "port": {
    "address": "/dev/ttyACM0",
    "label": "ttyACM0",
    "prefs": {
      "pid": "0x804e",
      "vid": "0x2341",
      "serialNumber": "EBEABFD6514D32364E202020FF10181E"
    },
    "identificationPrefs": {
      "pid": "0x804e",
      "vid": "0x2341"
    },
    "protocol": "serial",
    "protocolLabel": "Serial Port (USB)"
  }
}
```

it basically provides the same information as the `list` event but for a single port. After calling `START_SYNC` an initial burst of add events may be generated in sequence to report all the ports available at the moment of the start.

The `remove` event looks like the following:

```JSON
{
  "eventType": "remove",
  "port": {
    "address": "/dev/ttyACM0"
  }
}
```

the content is straightforward, in this case only the `address` field is reported.

If the information about a port needs to be updated the discovery may send a new `add` message for the same port address without sending a `remove` first: this means that all the previous information about the port must be discarded and replaced with the new one.

A demo tool is available here:
[https://github.com/arduino/serial-discovery](https://github.com/arduino/serial-discovery#example-of-usage)

A typical usage scenario is here:
https://github.com/arduino/serial-discovery#example-of-usage

### Integration with CLI and core platforms

#### Discovery tool install and launch

Discovery tools should be build natively for each OS and the CLI should run the correct tool for the running OS. This infrastracture is already available for platforms tools so the most natural way forward is to distribute the discoveries as tools within core platforms (in the same way we do for `gcc` or `avrdude`).

Some discovery like the `serial-discovery` must be always available, so they will be part of the `builtin` platform package and installed without the need to be part of a real platform (`builtin` is a dummy package that we use to install tools that are not part of any platforms like `ctags` for example).

The CLI will run `serial-discovery`, and other “builtin“ discoveries that we may want, by default.

3rd party platforms may add other discoveries by providing them as tools dependencies for their platform and by adding a directive to their `platform.txt` that informs the CLI that a new discovery is available.

```
discovery.DISCOVERY_ID.pattern=DISCOVERY_RECIPE
```

The CLI will look for directives matching the above pattern. `DISCOVERY_ID` must be replaced by a unique identifier for the particular discovery and `DISCOVERY_RECIPE` must be replaced by the command line to run to launch the discovery. An example could be:

```
## Teensy Ports Discovery
discovery.teensy.pattern="{runtime.tools.teensy_ports.path}/hardware/tools/teensy_ports" -J2
```

in this case the platform provides a new `teensy` discovery and the command line tool named `teensy_ports` is launched with the `-J2` parameter to start the discovery tool.

#### Duplicate discoveries

It may happen that different 3rd party platforms provides the same discovery or different versions of the same discovery or, worse, different version of the same discovery launched with different parameters.

We can partially handle this if the `DISCOVERY_ID` field in `platform.txt` is well defined: from the CLI we could group together the platforms that requires the same discovery and launch the latest version available just once. How the different 3rd party will agree on the `DISCOVERY_ID` value population is TBD.

#### Board identification

The metadata `identificationPrefs` associated to a port can be used to identify the board attached to that port. The algorithm is very simple: if a board listed in the platform file `boards.txt` match the same `identificationPrefs` coming from the discovery then the board is a “candidate” board attached to that port. Some identification properties may be imperfect and not uniquely identify a board, in that case more boards can match the same `identificationPrefs`, that’s why we called it “candidate”.

Let’s see an example to clarify things a bit, let suppose that we have the following `identificationPrefs` coming from the serial discovery:

```JSON
[...CUT...]
  "port": {
    "address": "/dev/ttyACM0",
    "identificationPrefs": {
      "pid": "0x804e",
      "vid": "0x2341"
    }
[...CUT...]
```

so we got `pid=0x804e, vid=0x2341`. Let’s suppose we have the following `boards.txt`:

```
# Arduino Zero (Prorgamming Port)
# ---------------------------------------
arduino_zero_edbg.name=Arduino Zero (Programming Port)
arduino_zero_edbg.vid.0=0x03eb
arduino_zero_edbg.pid.0=0x2157
arduino_zero_edbg.debug.tool=gdb
arduino_zero_edbg.upload.tool=openocd
arduino_zero_edbg.upload.protocol=sam-ba
[...CUT...]
# Arduino Zero (Native USB Port)
# --------------------------------------
arduino_zero_native.name=Arduino Zero (Native USB Port)
arduino_zero_native.vid.0=0x2341
arduino_zero_native.pid.0=0x804d
arduino_zero_native.vid.1=0x2341
arduino_zero_native.pid.1=0x004d
arduino_zero_native.vid.2=0x2341
arduino_zero_native.pid.2=0x824d
arduino_zero_native.vid.3=0x2341
arduino_zero_native.pid.3=0x024d
arduino_zero_native.upload.tool=bossac
arduino_zero_native.upload.protocol=sam-ba
[...CUT...]
# Arduino MKR1000
# -----------------------
mkr1000.name=Arduino MKR1000
mkr1000.vid.0=0x2341                   <------- MATCHING IDs
mkr1000.pid.0=0x804e                   <------- MATCHING IDs
mkr1000.vid.1=0x2341
mkr1000.pid.1=0x004e
mkr1000.vid.2=0x2341
mkr1000.pid.2=0x824e
mkr1000.vid.3=0x2341
mkr1000.pid.3=0x024e
mkr1000.debug.tool=gdb
mkr1000.upload.tool=bossac
mkr1000.upload.protocol=sam-ba
[...CUT...]
```

As we can see the only board that has the two properties matching is the `mkr1000`, in this case the CLI knows that the board is surely an MKR1000.

Note that `vid` and `pid` properties are just free text key/value pairs: the discovery may returns basically anything, the board just needs to have the same properties defined in `boards.txt` to be identified.

We can also specify multiple identification properties for the same board using the `.N` suffix, for example:

```
myboard.name=My Wonderful Arduino Compatible Board
myboard.pears=20
myboard.apples=30
```

will match on `pears=20, apples=30` but:

```
myboard.name=My Wonderful Arduino Compatible Board
myboard.pears.0=20
myboard.apples.0=30
myboard.pears.1=30
myboard.apples.1=40
```

will match on both `pears=20, apples=30` and `pears=30, apples=40`.

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

The selected port address will be provided in the variable `{upload.address}`. Other metadata provided by the discover in the `prefs` section will be provided in the `{upload.port.*}` variables.

For backward compatibility we will keep a copy of the address also in `{serial.port}` and in the specific case of a `protocol=serial` we will populate also `{serial.port.file}`.

For example, the following port metadata coming from a pluggable discovery:

```JSON
{
  "eventType": "add",
  "port": {
    "address": "/dev/ttyACM0",
    "label": "ttyACM0",
    "protocol": "serial",
    "protocolLabel": "Serial Port (USB)"
      "prefs": {
        "pid": "0x804e",
        "vid": "0x2341",
        "serialNumber": "EBEABFD6514D32364E202020FF10181E"
      },
      "identificationPrefs": {
        "pid": "0x804e",
        "vid": "0x2341"
      }
   }
}
```

will be available on the recipe as the variables:

```
{upload.address} = /dev/ttyACM0
{upload.protocol} = serial
{upload.port.pid} = 0x8043
{upload.port.vid} = 0x2341
{upload.port.serialNumber} = EBEABFD6514D32364E202020FF10181E
{serial.port} = ttyACM0                     # for backward compatibility
{serial.port.file} = /dev/ttyACM0           # only because protocol=serial
```

Here another example:

```JSON
{
  "eventType": "add",
  "port": {
    "address": "192.168.1.232",
    "label": "SSH on my-board ()192.168.1.232)",
    "protocol": "ssh",
    "protocolLabel": "SSH Network port"
      "prefs": {
        "macprefix": "AA:BB:CC",
        "macaddress": "AA:BB:CC:DD:EE:FF"
      },
      "identificationPrefs": {
        "macprefix": "AA:BB:CC"
      }
   }
}
```

that is translated to:

```
{upload.address} = 192.168.1.232
{upload.protocol} = ssh
{upload.port.macprefix} = AA:BB:CC
{upload.port.macaddress} = AA:BB:CC:DD:EE:FF
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
                                 -address {upload.address} -port 65280
                                 -sketch "{build.path}/{build.project_name}.bin"
```

#### User provided fields

The recipe may require some user-entered fields (like username or password). In this case the recipe must use the special placeholder `{upload.user.FIELD_NAME}` where `FIELD_NAME` must be declared separately in the recipe using the following format:

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
                                 -address {upload.address} -port 65280
                                 -username "{upload.user.username}
                                 -password "{upload.user.password}"
                                 -sketch "{build.path}/{build.project_name}.bin"
```

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
