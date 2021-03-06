+++
title = "periph"
description = "Peripherals I/O in Go"
+++

![boardimage](/img/periph-mascot-280.png)

# Overview

[periph.io/x/periph](https://periph.io/x/periph) is a standalone library with no
external dependency to interface with low-level board facilities exposed by the
OS or leveraged against OS' will. It can be viewed as a lower level layer than
[Gobot](https://gobot.io), and yes we're discussing to collaborate in the
future!


# Features

- No external dependencies
- No C dependency, doesn't use `cgo`
- Explicit initialization: know what [hardware is detected and what is
  not](https://github.com/google/periph/tree/master/cmd/periph-info)
- [Interfaces](https://periph.io/x/periph/conn):
  - [GPIO](https://periph.io/x/periph/conn/gpio): both memory mapped
  registers and edge detection
  - [I²C](https://periph.io/x/periph/conn/i2c)
  - [SPI](https://periph.io/x/periph/conn/spi)
  - [1-wire](https://periph.io/x/periph/conn/onewire)
- [Devices](/device/):
  - Output: [apa102](/device/apa102/), [ssd1306](/device/ssd1306/),
    [tm1637](/device/tm1637/)
  - Sensing: [bmp180, bme280 and bmp280](/device/bmxx80/),
    [ds18b20](/device/ds18b20/), [ir](/device/ir/), [lepton](/device/lepton/)
  - Interfacing: [ds248x](/device/ds248x/)
- Continuously tested via [gohci](https://github.com/periph/gohci) on:
  - [BeagleBone](/platform/#beaglebone)
  - [C.H.I.P.](/platform/#chip)
  - [ODROID-C1+](/platform/#odroid-c1)
  - [Raspberry Pi](/platform/#raspberrypi)
  - [Windows 10 VM](/platform/#windows)
- **[SemVer](http://semver.org) compatibility guarantee**
  - Major version change (`v1.0` to `v2.0`) may introduce breaking changes.
  - Minor version change (`v1.1` to `v1.2`) will be backward compatible.
  - `master` may contain breaking changes, use
    [dep](https://github.com/golang/dep).


# Documentation

- [platform/](/platform/) for an overview of supported boards (Raspberry Pi,
  CHIP, etc) and OSes plus tips and tricks.
- [device/](/device/) for an overview of included device drivers and examples.
  [![GoDoc](/img/godoc.svg)](https://periph.io/x/periph)
- [project/](/project/) to read about design goals, using as a libray, using
  included tools, contribution rules or expand the list of supported hardware.


# Tools

`periph` includes [many ready-to-use
tools](https://github.com/google/periph/tree/master/cmd/)! See [project/tools/](/project/tools/)
for more info about using the included tools.

```bash
# Retrieve and install all the commands at once:
go get periph.io/x/periph/cmd/...
# List the host drivers registered and/or initialized:
periph-info
# List the known headers:
headers-list
# List the known GPIO state:
gpio-list
```


# Library

For [application developers](/project/library/), using `periph` as a library
provides OS-independent bus interfacing. It really tries hard to _get out of the
way_.  Here's the canonical "toggle a LED" example:


~~~go
package main

import (
    "time"
    "periph.io/x/periph/conn/gpio"
    "periph.io/x/periph/conn/gpio/gpioreg"
    "periph.io/x/periph/host"
)

func main() {
    host.Init()
    for l := gpio.Low; ; l = !l {
        gpioreg.ByName("11").Out(l)
        time.Sleep(500 * time.Millisecond)
    }
}
~~~

The following are synonyms, use the form you prefer:

- Runtime discovery:
  - [`gpioreg.ByName("11")`](https://periph.io/x/periph/conn/gpio/gpioreg#ByName): gpio number
  - [`gpioreg.ByName("GPIO11")`](https://periph.io/x/periph/conn/gpio/gpioreg#ByName): gpio name as per the CPU driver
  - [`gpioreg.ByName("P1_23")`](https://periph.io/x/periph/conn/gpio/gpioreg#ByName): board header `P1` position `23`
  - [`gpioreg.ByName("SPI0_CLK")`](https://periph.io/x/periph/conn/gpio/gpioreg#ByName): function clock on SPI bus 0
- Using global variables:
  - [`rpi.P1_23`](https://periph.io/x/periph/host/rpi#P1_33) to
    select the pin via its _position on the board_
  - [`bcm283x.GPIO11`](https://periph.io/x/periph/host/bcm283x#GPIO13) for the
    pin as defined by the CPU

This example uses basically no CPU: the
[Out()](https://godoc.org/periph.io/x/periph/conn/gpio#PinOut) call doesn't call
into the kernel, unlike other Go hardware libraries. Instead it *directly*
writes to the GPIO memory mapped register.


## Examples

See [device/](/device/) for more examples.

![boardimage](/img/lab-280.jpg)


# Code

Code is located at [github.com/google/periph](https://github.com/google/periph)

Supplemental projects are located at
[github.com/periph](https://github.com/periph). This includes:

- The [website](https://github.com/periph/website) itself, so you can easily
  submit a PR to improve the documentation.
- [gohci](https://github.com/periph/gohci) for hardware smoke testing.
- [periph-tester](https://github.com/periph/periph-tester) which is a board used
  to confirm the buses (I²C, SPI, 1-wire) are correctly working.
- [bootstrap](https://github.com/periph/bootstrap) tool to automate the
  deployment of Raspberry Pis.


# Contact

- [#periph](https://gophers.slack.com/messages/periph/) on gophers.slack.com.
  Request access at
  [invite.slack.golangbridge.org](https://invite.slack.golangbridge.org/)
- File issues at
  [github.com/google/periph/issues](https://github.com/google/periph/issues)
- [periph-users@googlegroups.com](https://groups.google.com/forum/#!forum/periph-users)
  for users of the library and general questions
- [periph-dev@googlegroups.com](https://groups.google.com/forum/#!forum/periph-dev)
  for driver developers


# Contributions

`periph` provides an extensible driver registry and common bus interfaces which
are explained in more details at [project/](/project/). `periph` is designed to
work well with drivers living in external repositories so you are not _required_
to fork the periph repository to load out-of-tree drivers for your platform.

**Every commit is [tested on real hardware](/project/contributing/#testing)
via [gohci](https://github.com/periph/gohci) workers.**

We gladly accept contributions for documentation improvements and from device
driver developers via GitHub pull requests, as long as the author has signed the
Google Contributor License. Please see
[project/contributing/](/project/contributing/) for more details.


# Philosophy

1. Optimize for simplicity, correctness and usability in that order.
   - e.g. everything, interfaces and structs, uses strict typing, there's no
     `interface{}` in sight.
2. OS agnostic. Clear separation of interfaces in
   [conn/](https://periph.io/x/periph/conn),
   enablers in [host/](https://periph.io/x/periph/host) and device
   drivers in [devices/](https://periph.io/x/periph/devices).
   - e.g. no devfs or sysfs path in sight.
   - e.g. conditional compilation enables only the relevant drivers to be loaded
     on each platform.
3. ... yet doesn't get in the way of platform specific code.
   - e.g. A user can use statically typed global variables
     [rpi.P1_3](https://periph.io/x/periph/host/rpi#P1_3),
     [bcm283x.GPIO2](https://periph.io/x/periph/host/bcm283x#GPIO2)
     to refer to the exact same pin on a Raspberry Pi.
3. The user can chose to optimize for performance instead of usability.
   - e.g.
     [apa102.Dev](https://periph.io/x/periph/devices/apa102#Dev)
     exposes both high level
     [draw.Image](https://golang.org/pkg/image/draw/#Image) to draw an image and
     low level [io.Writer](https://golang.org/pkg/io/#Writer) to write raw RGB
     24 bits pixels. The user chooses.
4. Use a divide and conquer approach. Each component has exactly one
   responsibility.
   - e.g. instead of having a driver per "platform", there's a driver per
     "component": one for the CPU, one for the board headers, one for each
     bus and sensor, etc.
5. Extensible via a [driver
   registry](https://periph.io/x/periph#Register).
   - e.g. a user can inject a custom driver to expose more pins, headers, etc.
     A USB device (like an FT232H) can expose headers _in addition_ to the
     headers found on the board.
6. The drivers must use the fastest possible implementation.
   - e.g. both [allwinner](https://periph.io/x/periph/host/allwinner) and
     [bcm283x](https://periph.io/x/periph/host/bcm283x) leverage [sysfs
     gpio](https://periph.io/x/periph/host/sysfs#Pin) to expose interrupt driven
     edge detection, yet use memory mapped GPIO registers to perform
     single-cycle reads and writes.


# Authors

`periph` was initiated with ❤️️ and passion by [Marc-Antoine
Ruel](https://github.com/maruel).  The full list of contributors is in
[AUTHORS](https://github.com/google/periph/blob/master/AUTHORS) and
[CONTRIBUTORS](https://github.com/google/periph/blob/master/CONTRIBUTORS).


# Disclaimer

This is not an official Google product (experimental or otherwise), it
is just code that happens to be owned by Google.

This project is not affiliated with the Go project.
