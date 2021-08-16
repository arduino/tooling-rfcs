# Pluggable Monitor (Version 1)

## Overview

This document describes how the Pluggable Monitor works and how it should integrate with discoveries and uploaders.

## Problem

With the introduction of Pluggable Discovery the Arduino platforms are now allowed to seamlessly add support for new types of communication "ports" that can be used to upload new sketches or communicate with the board. In particular the communication with the board until now has been done using the Serial Monitor of the Arduino IDE but, with the new kind of communication protocols enabled by the Pluggable Discovery, this is no more sufficient.

The Pluggable Monitor aims to allow platforms to provide the missing piece to allow the user to communicate with any kind of port through, virtually, any protocol, not only serial.

## Constraints

- Software support for a new **protocol monitor** must be added to the system **using an external command line tool**

- A communication port is identified by a **protocol** and an **address**

- A communication port may require **configuration options** (baudrate, parity, etc.)

- A pluggable monitor must be able to handle a huge amount of data with very high transfer rates

## Proposed Solution

Each platform should provide a tool that can open a connection to a port through a **single specific port protocol**. There will be a tool for each supported protocol so, in the end, we will have a "Serial port monitor", a "Network port monitor", and so on.

These tools must be in the form of command line executables that can be launched as a subprocess. They will communicate to the parent process via stdin/stdout, in particular a monitor will accept commands as plain text strings from stdin and will send answers back in JSON format on stdout. Each tool will implement the commands to open and control communication ports for a specific protocol as specified in this document. The actual I/O data stream from the communication port will be transferred to the parent process through a separate channel via TCP/IP.

### Pluggable monitor API via stdin/stdout

All the commands listed in this specification must be implemented in the monitor tool.

After startup, the tool will just stay idle waiting for commands. The available commands are: `HELLO`, `DESCRIBE`, `CONFIGURE`, `OPEN`, `CLOSE` and `QUIT`.

After each command the client always expects a response from the monitor. The monitor must not introduce any delay and must respond to all commands as fast as possible.

#### HELLO command

`HELLO` **must be the first command sent** to the monitor to tell the name of the client/IDE and the version of the pluggable monitor protocol that the client/IDE supports.
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

- if the client/IDE supports the same or a more recent version of the protocol than the monitor tool, then the IDE should go into a compatibility mode and use the protocol level supported by the monitor tool.
- if the monitor tool supports a more recent version of the protocol than the client/IDE then the monitor tool should downgrade itself into compatibility mode and report a `protocolVersion` that is less than or equal to the one supported by the client/IDE.
- if the monitor tool cannot go into compatibility mode, it will report the protocol version supported (even if greater than the version supported by the client/IDE) and the client/IDE may decide to terminate the monitor tool or print an error/warning.

#### DESCRIBE command

The `DESCRIBE` command returns a description of the communication port.
The description will have metadata about the port configuration, and which parameters are available to the user.

```JSON
{
  "event": "describe",
  "message": "ok",
  "port_description": {
    "protocol": "serial",
    "configuration_parameters": {
      "baudrate": {
        "label": "Baudrate",
        "type": "enum",
        "values": [
          "300", "600", "750", "1200", "2400", "4800", "9600",
          "19200", "38400", "57600", "115200", "230400", "460800",
          "500000", "921600", "1000000", "2000000"
        ],
        "selected": "9600"
      },
      "parity": {
        "label": "Parity",
        "type": "enum",
        "values": [ "N", "E", "O", "M", "S" ],
        "selected": "N"
      },
      "bits": {
        "label": "Data bits",
        "type": "enum",
        "values": [ "5", "6", "7", "8", "9" ],
        "selected": "8"
      },
      "stop_bits": {
        "label": "Stop bits",
        "type": "enum",
        "values": [ "1", "1.5", "2" ],
        "selected": "1"
      }
    }
  }
}
```

Each parameter has a unique name (`baudrate`, `parity`, etc...), a `type` (in this case only `enum` but more types will be added in the future), and the `selected` value for each parameter.

The parameter name can not contain spaces, and the allowed characters in the name are alphanumerics, underscore `_`, dot `.`, and dash `-`.

The `enum` types must have a list of possible `values`.

The client/IDE may expose these configuration values to the user via a config file or a GUI, in this case the `label` field may be used for a user readable description of the parameter.

#### CONFIGURE command

The `CONFIGURE` command sets configuration parameters for the communication port. The parameters can be changed one at a time and the syntax is:

`CONFIGURE <PARAMETER_NAME> <VALUE>`

The response to the command is:

```JSON
{
  "event": "configure",
  "message": "ok",
}
```

or if there is an error:

```JSON
{
  "event": "configure",
  "error": true,
  "message": "invalid value for parameter baudrate: 123456"
}
```

The currently selected parameters may be obtained using the `DESCRIBE` command.

#### OPEN command

The `OPEN` command opens a communication with the board, the data exchanged with the board will be transferred to the Client/IDE via TCP/IP.

The Client/IDE must first TCP-Listen to a randomly selected port and send it to the monitor tool as part of the `OPEN` command. The syntax of the `OPEN` command is:

`OPEN <CLIENT_IP_ADDRESS> <BOARD_PORT>`

For example, let's suppose that the Client/IDE wants to communicate with the serial port `/dev/ttyACM0` then the sequence of actions to perform will be the following:

1. the Client/IDE must first listen to a random TCP port (let's suppose it chose `32123`)
1. the Client/IDE sends the command `OPEN 127.0.0.1:32123 /dev/ttyACM0` to the monitor tool
1. the monitor tool opens `/dev/ttyACM0`
1. the monitor tool connects via TCP/IP to `127.0.0.1:32123` and start streaming data back and forth

The answer to the `OPEN` command is:

```JSON
{
  "event": "open",
  "message": "ok"
}
```

If the monitor tool cannot communicate with the board, or if the tool can not connect back to the TCP port, or if any other error condition happens:

```JSON
{
  "event": "open",
  "error": true,
  "message": "unknown port /dev/ttyACM23"
}
```

The board port will be opened using the parameters previously set through the `CONFIGURE` command.

Once the port is opened, it may be unexpectedly closed at any time due to hardware failure, or because the Client/IDE closes the TCP/IP connection. In this case an asynchronous `port_closed` message must be generated from the monitor tool:

```JSON
{
  "event": "port_closed",
  "message": "serial port disappeared!"
}
```

or

```JSON
{
  "event": "port_closed",
  "message": "lost TCP/IP connection with the client!"
}
```

#### CLOSE command

The `CLOSE` command will close the currently opened port and close the TCP/IP connection used to communicate with the Client/IDE. The answer to the command is:

```JSON
{
  "event": "close",
  "message": "ok"
}
```

or in case of error

```JSON
{
  "event": "close",
  "error": true,
  "message": "port already closed"
}
```

#### QUIT command

The `QUIT` command terminates the monitor. The response to `QUIT` is:

```JSON
{
  "eventType": "quit",
  "message": "OK"
}
```

after this output the monitor exits. This command is supposed to always succeed.

#### Invalid commands

If the client sends an invalid or malformed command, the monitor should answer with:

```JSON
{
  "eventType": "command_error",
  "error": true,
  "message": "Unknown command XXXX"
}
```

#### Reference implementations

TODO...

### Integration with Arduino CLI and core platforms

In this section we will see how monitors are distributed and integrated with Arduino platforms.

#### Monitor tool distribution

The monitor tools must be built natively for each OS and the CLI should run the correct tool for the running OS.

The distribution infrastructure is already available for platform tools, like compilers and uploaders, through [the Arduino package index](https://arduino.github.io/arduino-cli/latest/package_index_json-specification) so, the most natural way forward is to distribute also the monitor tools in the same way.
3rd party developers should provide their monitor tools by adding them as resources in the `tools` section of their package index (at the `packages` level).

Let's see an example of adding a monitor tool to a package index:

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
        {
          "name": "ble-discovery",
          "version": "1.0.0",
          "systems": [ ... ]
        },
+       {
+         "name": "ble-monitor",  <--- Monitor is distributed as a TOOL
+         "version": "1.0.0",
+         "systems": [
+           {
+             "host": "x86_64-pc-linux-gnu",
+             "url": "http://example.com/ble-mon-1.0.0-linux64.tar.gz",
+             "archiveFileName": "ble-mon-1.0.0-linux64.tar.gz",
+             "checksum": "SHA-256:0123456789abcdef0123456789abcdef0123456789abcdef",
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

In this case we are adding an hypothetical `ble-monitor` version `1.0.0` to the toolset of the vendor `arduino`. From now on, we can uniquely refer to this monitor with the pair `PACKAGER` and `MONITOR_NAME`, in this case `arduino` and `ble-monitor` respectively.

The compressed archive of the monitor must contain only a single executable file (the monitor itself) inside a single root folder. This is mandatory since the CLI will run this file automatically when a monitor instance is requested.

#### Monitor tools integration

Each core platform must refer to the specific monitor tools they need by adding them (together with the pluggable discoveries...) inside the `discoveryDependencies` field of the Arduino package index:

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
+         "discoveryDependencies": [  <--- Discoveries AND monitors used in the platform
            {
              "packager": "arduino",
              "name": "ble-discovery"
            },
+           {
+             "packager": "arduino",
+             "name": "ble-monitor"
+                                     <--- Version is not required!
+           }
          ]
        },

        {
          "name": "Arduino SAMD Boards",
          "architecture": "samd",
          "version": "1.6.18",
          ...
          "toolsDependencies": [ ... ],
          "discoveryDependencies": [ ... ]
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
        },
        {
          "name": "ble-monitor",
          "version": "1.0.0",
          "systems": [ ... ]
        }
      ],
    }
  }
}
```

Adding the needed monitor tools in the `discoveryDependencies` allows the CLI to install them together with the platform. Also, differently from the other `toolsDependencies`, the version is not required since the latest version available will always be used.

Finally, to bind a monitor to a protocol, we must also declare in the `platform.txt` that we want to use that specific monitor tool for that specific protocol with the direcive:

```
pluggable_monitor.required.PROTOCOL=PLATFORM:MONITOR_NAME
```

the platform can support as many protocols as needed:

```
pluggable_monitor.required.PROTOCOL1=PLATFORM:MONITOR_NAME1
pluggable_monitor.required.PROTOCOL2=PLATFORM:MONITOR_NAME2
...
```

in our specific example the directive should be:

```
pluggable_monitor.required.ble=arduino:ble-monitor
```

where `ble` is the port protocol identification returned by the matching pluggable discovery.

#### Using a monitor tool made by a 3rd party

A platform developer may opt to depend on a monitor tool developed by a 3rd party instead of writing and maintaining their own.

Since writing a good-quality cross-platform monitor tool is very hard and time consuming, we expect this option to be the one used by the majority of the developers.

#### Direct monitor tool integration (not recommended)

A monitor tool may be directly added to a platform, without passing through the `discoveryDependencies` in the Arduino package index, using the following directive in the `platform.txt`:

```
pluggable_monitor.pattern.PROTOCOL=MONITOR_RECIPE
```

where `MONITOR_RECIPE` must be replaced by the command line to launch the monitor tool for the specific `PROTOCOL`. An example could be:

```
pluggable_monitor.pattern.custom-ble="{runtime.tools.my-ble-monitor.path}/my-ble-monitor" -H
```

in this case the platform provides a new `custom-ble` protocol monitor tool and the command line tool named `my-ble-monitor` is launched with the `-H` parameter to start the monitor tool. In this case the command line pattern may contain any extra parameter in the formula: this is different from the monitor tools installed through the `discoveryDependencies` field that must run without any command line parameter.

This kind of integration may turn out useful:

- during the development of a platform (because providing a full package index may be cumbersome)
- if the monitor tool is specific to a platform and can not be used by 3rd party

Anyway, since this kind of integration does not allow reusing a monitor tool between different platforms, we do not recommend its use.

#### built-in monitor tools and backward compatibliity considerations

Some monitor tools like the Arduino `serial-monitor` or the Arduino `network-monitor` must be always available, so they will be part of the `builtin` package and installed without the need to be part of a real package (`builtin` is a dummy package that we use to install tools that are not part of any platforms like `ctags` for example).

If a platform requires the builtin monitor tools it must declare it with:

```
pluggable_monitor.required.serial=builtin:serial-monitor
pluggable_monitor.required.network=builtin:network-monitor
```

For backward compatibility, if a platform does not declare any discovery or monitor tool (using the `pluggable_discovery.*` or `pluggable_monitor.*` properties in `platform.txt` respectively) it will automatically inherit `builtin:serial-monitor` and `builtin:network-monitor` (but not other `builtin` monitor tools that may be possibly added in the future). This will allow all legacy non-pluggable platforms to migrate to pluggable monitor without disruption.
