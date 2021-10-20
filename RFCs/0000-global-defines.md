# Restore the possibility of using global defines across a project

## Overview

This RFC proposes a simple and generic change in the arduino-cli tool (used by arduino-ide) to give to a sketch the ability to properly configure its environment (the libraries): the ability to define global macros (which is the `-DMACRO=value` compiler option seen in almost every makefile).

## Problem

A long-term issue with the Arduino environment has been the inability for the user to declare macros for them to be available across all source files (sketch, libraries, not the core).

Both users and library maintainers are restrained in their action because of the lack of globally configurable macro definitions, on which C/C++ historically rely on, but which is not possible from the user point of view in this environment from the beginning.

As a consequence,

- A user needs to manually edit a library and change the default definitions.
- A library maintainer who is willing to provide the ability to change the defaults must provide a runtime API (vs compile-time defines) for configuration. Such API can be space consuming and can be a real issue with small targets. Also noting that nowadays modern c++ code tend to use more and more of template and constexpr which are also a compiler way of implementing preprocessor defines (the reason in this case is performance).
- Many attempts were proposed during the past 9 years (see appendix)

Also note that Arduino's `build*extra_flags=` is an installation-wide configuration [barely accessible](https://arduino.github.io/arduino-cli/0.19/platform-specification/#platformlocaltxt) to the average user.

## Constraints

It is not advised to expect already existing libraries to adapt themselves to a new feature. However, some of them are already providing support for overrides:

```cpp
#ifndef BUFFER_SIZE
#define BUFFER_SIZE 128
#endif
```

Such overrides are usable with other integration environments, or when using `arduino-cli` from command-line and scripts. Arduino-IDE currently has no mean to achieve such definition globally and this RFC aims at helping with this situation.

## Recommended Solution

- `sketch_globals.h` next to `sketch.ino`

## Solutions

### 1) "arduifine"

A [closed pull request](https://github.com/arduino/arduino-cli/pull/1117) providing a POC has received no answer for almost a year.

#### Strengths

- Providing an answer to the described issue
- Requiring no further parsing of source code
- Requiring no change from libraries supporting overrides
- Answering to requests such as the one listed in https://github.com/arduino/Arduino/issues/421#issuecomment-327003003

#### Weaknesses

- Proof of concept
- `const char* ARDUIFINE_blah = "-DMACRO=value"` is not really the way. The goal was to have a discussion.

### 2) "libdefs": a way for user to configure libraries

It is another [closed pull request](https://github.com/arduino/arduino-cli/pull/1517) inspired by https://github.com/arduino-libraries/ArduinoBearSSL/pull/45.

#### Strengths

- Using standard C/C++ way of defining macro

#### Weaknesses

- Requiring changes in libraries as pointed by @ubidefeo in https://github.com/arduino/arduino-cli/pull/1517#issuecomment-946634476

### 3) `sketch_globals.h` next to `sketch.ino`

An [on-going pull-request](https://github.com/arduino/arduino-cli/pull/1524) implements this proposal. It allows the automatic inclusion of "<sketch-name>`_global.h`" file _from sketch and libraries_ but not core files.

#### Strengths

- IDE friendly: user needs to open a new file with a clearly defined and hopefully unique name in the sketch directory
- Self contained: A source file which can be released as part of a user project
- Compatible with libraries supporting overrides

#### Weaknesses

- Can be seen as competing with [RFC-0003-build-profiles](https://github.com/arduino/tooling-rfcs/blob/main/RFCs/0003-build-profiles.md).
  But it is not, it rather should be seen as complementary. Average arduino users may be familiar more with `#define BUFFER_SIZE 128` than with the proposed yaml file which is, beside allowing global defines, and if I understand, mainly aimed at project releasers providing nightly builds across multiple architectures.

### 4) Other proposals

See appendix for a list of still ongoing pull request.

## Appendix

- List of other attempts
  - many requests and proposals in 9yo https://github.com/arduino/Arduino/issues/421
  - https://github.com/arduino/Arduino/pull/1808#issuecomment-103507284 closed claiming for richer APIs in libraries instead of global defines
  - Still opened PR from https://github.com/arduino/arduino-cli/issues/846 \'s OP compiled by @per1234
    (there are quite a number of closed attempts clearly showing the lack of something)
    - https://github.com/arduino/arduino-builder/pull/29
    - https://github.com/arduino/arduino-builder/pull/282
